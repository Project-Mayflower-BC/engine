# Engine Architecture

**Status:** Phase 1 prototype · **Updated:** 2026-11-05 · **Repo:** `Project-Mayflower-BC/engine`

## 1. Purpose

The engine is the headless core that every other part of the project runs on: the bots, self-play and RL
training, opening review, move review, practice and the Arena. Phase 1 builds a harness first. The harness
defines how a game is played, checked, benchmarked and trained on. The Catan rules are then written to plug
into it.

Requirements covered: FR-04 (legal play), FR-05 (hidden-information views), FR-06 (bot interface and
baselines), NFR-04 (throughput measurement), and section 2.7 (seeded determinism, configurable settings,
a Python-first performance path).

## 2. Layers

```
 tools (generic)    bench · runner.play_game · testing.run_all · rl.SeatEnv
        │                         │
        ▼                         ▼
 Game interface  ◄────────►  Bot interface
 + GameSettings              choose_move(view, legal_moves, time_limit)
        │
        ▼
 rules modules      games/toy (harness test subject) · games/catan (team)
```

- **Rules modules** implement `Game`. They are the only code that knows the rules of a specific game.
- **Tools** use only the `Game` and `Bot` interfaces. They work unchanged for any rules module, and for a
  later Rust rewrite of slow code paths behind the same interface (I-6).
- **Bots** only ever receive a view and the legal moves. They never see the full state.

## 3. Design style

The interfaces are object-oriented, and everything that passes through them is plain data.

| Thing | Form | Why |
| --- | --- | --- |
| `Game`, `Bot`, `RLAdapter` | Abstract classes | Rules modules and bots can be swapped |
| State | Plain record of lists and ints, no methods | Fast `copy()` for search; easy to port to Rust |
| Move | Small immutable value (tuple or frozen dataclass) | Hashable, so it can be compared and counted |
| View | Frozen record built by copying | Can't leak hidden data (FR-05) |
| Settings | Frozen record that converts to and from JSON | Goes straight into job tickets |

Rules are pure functions: `apply(state, move) -> new_state`. The `Game` object itself stores nothing.
Rules are code, not config. Each rules module keeps its constants as data tables (board topology and
adjacency, costs, deck composition, ports, piece limits), so variants mostly mean different tables plus
settings.

## 4. Components

### 4.1 `GameSettings` (`settings.py`)

Every rule that can vary between games, so that **(settings, seed) fully defines a game**.

| Field | Default | Notes |
| --- | --- | --- |
| `num_players` | 4 | Project scope is 4-player; the engine doesn't hard-code it |
| `vp_to_win` | 10 | 1v1 is played to 15 |
| `discard_limit` | 7 | A 7 makes players holding more than this discard half |
| `friendly_robber`, `friendly_robber_vp` | False, 2 | |
| `dice_mode` | `"random"` | `"balanced"` is pending I-12 |
| `hide_bank_cards` | False | Colonist option; views must hide bank counts when set |
| `max_trade_offers_per_turn` | 3 | Keeps the move list, random play and search bounded |
| `max_turns` | None | Safety cap for training; a game that hits it ends with no winner |
| `expansions` | `{}` | Opt-in flags, e.g. `{"seafarers"}`, `{"1v1"}` |
| `options` | `{}` | Free-form, mode-specific values (must convert to JSON) |

Presets: `standard()`, `colonist_ranked()`, `one_v_one()`. A rules module calls
`settings.require_supported(...)` in `new_game`, so unsupported settings fail immediately rather than
partway through a game.

### 4.2 `Game` interface (`game.py`)

| Method | Contract |
| --- | --- |
| `new_game(settings, seed)` | Starting state; rejects settings the rules don't support |
| `current_player(state)` | Seat to decide now (may not be the turn player: discards, trade responses); None when over |
| `legal_moves(state)` | Every legal move for that seat, in a fixed order; non-empty until the game ends |
| `apply(state, move)` | Returns a new state and never changes its input; raises `IllegalMove(move, reason)` |
| `is_terminal`, `winner`, `returns` | Whether the game is over, the winner, and the payoff per seat (1 for the winner, 0 otherwise) |
| `view_for(state, seat)` | What the seat may know, built by copying |
| `copy(state)` | Fast hand-written copy (the default deep copy is slow) |
| `determinize(view, rng)` | Samples a full state consistent with the view, for search bots |

Randomness (dice, card draws, steals) comes from an RNG stored in the state, so the same seed and moves
always give the same game. Copying a state copies its RNG too.

### 4.3 Bots (`bots/`)

- `Bot.reset(seat, settings, seed)` is called before each game. `choose_move(view, legal_moves, time_limit)`
  returns a move, and `win_probability(view)` returns an estimate (FR-06).
- `RandomBot` is the baseline. `FlatMonteCarloBot` is the benchmark: for each move it samples hidden
  information, plays the game out randomly, and picks the best average result.

### 4.4 Runner and benchmark (`runner.py`, `bench.py`)

- `play_game(game, bots, settings, seed)` rejects any bot move that isn't in the legal list (`BotError`).
  It counts time-limit overruns per seat and returns a `GameResult` with the winner, returns and move list.
- `uv run python -m mayflower_engine.bench --game toy --games 1000` measures games per second for NFR-04.

### 4.5 Conformance suite (`testing/`)

`run_all(game, settings, bad_moves)` checks that:
1. Random games finish, every listed move applies, and the move list is never empty and always in the same order.
2. `apply` never changes its input.
3. The same seed gives the same game.
4. `copy` is independent of the original, including its RNG.
5. Views share no objects with the state, and an old view doesn't change later (FR-05).
6. Each seat's event log only grows and never rewrites past events.
7. A state built by `determinize` gives the same view and the same legal moves.
8. Moves that aren't legal raise `IllegalMove`.

Every rules module must pass `run_all`. Rule-specific behavior, such as a city requiring your own
settlement, is tested separately against specific positions.

### 4.6 Event log (`events.py`)

Every view carries `view.events`, the ordered list of everything that seat has seen. Bots use it to count
cards and track hands.
- **Event format:** each event is an immutable tuple such as `("roll", seat, value)` or
  `("trade", a, b, gave, got)`. `log.append(event)` records a public event, and
  `log.append(event, {seat, ...})` records a private one (a stolen card is visible to the thief and the
  victim only).
- **Streams:** each seat has its own stream. One extra stream holds every event, for replay and review
  (`log.all()`), and is never given to bots.
- **Sharing instead of copying:** copying or filtering the whole log on every move made long games
  quadratic. So `EventLog.copy()` shares streams and only copies a stream when two copies diverge.
  `view_for` hands out an `EventView`, a fixed-length read-only window. Both are O(1) per move.
- **Bots:** use `events.since(n)` to process only new events.
- **Determinize:** `EventLog.from_view(view.events, seat, n)` rebuilds a log for search, since other
  seats' private events are unknown.

### 4.7 Player trading

Trading is an ordinary part of the move list, so any bot can offer, accept, reject and confirm trades.
1. The turn player makes an offer, which is a move.
2. `current_player` then moves to each other player in seat order, and each one accepts or rejects. Only
   a player who can pay may accept.
3. The offerer confirms with one of the players who accepted, or cancels.

`settings.max_trade_offers_per_turn` caps the offers per turn, so random play and search always finish.
The toy game implements this whole flow, which shows that the runner, conformance tests and RL env all
handle decisions made out of turn.

### 4.8 RL (`rl/`, optional `rl` extra)

- `SeatEnv` is a Gymnasium env: one learning seat plays against fixed bots, the learner's seat can be random
  each episode, and the reward is the learner's return at the end of the game.
- Legal actions are in `info["action_mask"]`, and in `action_masks()` for maskable PPO.
- When the learner has only one legal move (such as the roll), the env plays it automatically
  (`auto_play_forced`), so the agent only sees real decisions.
- Illegal actions raise an error by default. An optional penalty mode is available for learners that can't
  use masks.
- Each game provides an `RLAdapter` that turns moves into fixed action numbers and views into an
  observation vector. For Catan, this encoding is a design task in its own right.

## 5. Planned additions

### 5.1 Belief tracking (Phase 1, FR-03 / FR-05)

The event log (4.6) is built. Next is a generic belief-tracker interface: `update(event)`,
`possible_hands(seat)` and `sample(rng)`. It keeps, for each opponent, the hands they could be holding,
narrowing them with public events such as gains, spends and trades, and branching on hidden steals.
`determinize` samples from it instead of uniformly. Test: in self-play, the true hand must always be among
the possible hands.

### 5.2 Robber deals: promises (later phase; the option is built in now)

Robber deals are offers on a 7 or a knight, before the robber moves. Either side can start one:
- **No-block:** "give me X and I won't put the robber on your hexes".
- **No-steal:** "give me X and I won't steal from you".
- **Asking the table:** "will anyone give me a no-block deal for wheat?"

The engine doesn't force anyone to keep a deal, but every promise is public, so it works like a promise in
practice. Modelling:
- **A deal is a player trade with a promise attached.** It goes through the same offer → responses →
  confirm flow as 4.7, but in a robber-deal window between rolling the 7 (or playing the knight) and
  moving the robber. One side gives resources, and the other side gives a promise such as
  `("no_block", by, to)` or `("no_steal", by, to)`.
- **Promise ledger:** accepted promises go into the state's public promise ledger and the event log.
  Moving the robber against a promise is legal, but the engine logs a public `("promise_broken", ...)`
  event. Bots read the ledger and that history to learn trust and reputation (this feeds FR-18,
  Grudge-holder).
- **Settings:**
  - `options["robber_deals"]` turns the deal window on (off until that phase).
  - `options["enforce_promises"]` makes the engine remove promised hexes and victims from the legal robber
    moves, a simpler variant for training.
- **Replay:** the Collector already records Colonist chat. A replayed deal is the chat message with the
  promise, linked to the trade that follows it. The replay layer turns that pair into one deal move and
  can flag when a promise was broken.

The pieces built now that make deals a later addition rather than a redesign are: the event log, trades
made out of turn, and the `options` field.

### 5.3 Lessons from the Catanatron community

These come from the Catanatron Discord `#general` and `#machine-learning-talk` exports (Oct 2025 to
Jun 2026). Use them as ideas only. Catanatron is GPL-3.0, and the other projects mentioned have their own
licenses, so don't copy their code.

**Most of that discussion is about 1v1** (15 VP, no player trades, exact card counting). 4-player games
are longer, branch more because of trades and trade responses, and hide more information, so the speed
and strength numbers below aren't directly comparable to ours.

**Rules module design (for the team writing Catan)**
- **Discard one card at a time.** Catanatron discarded randomly because choosing a whole discard set
  explodes the action space. The working fix is a discard phase where the player picks one resource per
  move until they are at the limit. Do the same for Year of Plenty (two single picks).
  - The discard is still planned as a whole. The view shows how many cards are left to discard, so a
    bot sees its earlier picks when it makes the next one. Search plans the whole sequence, and a
    learning bot learns it.
  - Discarding is strategic. Players keep cards that work toward their next build, and they sometimes
    throw away the cards the robber is likely to steal from them, so the player who rolled the 7 can't
    get them. Since discards happen before the steal, that is a legal and real tactic. The engine
    doesn't need special support for it. It needs the event log, the possible-hands tracker (so a bot can
    tell what the roller needs) and turn order. Bots value it through evaluation or learn it.
  - To avoid duplicate orderings, discards can be required in resource order (e.g. no brick after ore),
    so "wood then ore" and "ore then wood" are the same single sequence.
- **Keep the move list in a fixed, sensible order.** Search can reuse work and look far deeper when moves
  are always generated in the same order. Our "fixed order" rule (4.2) is the start of this.
- **Copying the game state is the main cost in search.** A search bot asks "what if I do X, then they do
  Y?". To try that without changing the real game, it copies the game state (board, hands, deck, robber,
  whose turn it is) and plays the moves forward on the copy. It does this thousands of times per
  decision. A deep copy duplicates every nested piece of the state, which is slow. In Catanatron about
  30% of search time is copying, followed by road building and longest-road work. So keep the state small
  and flat with a hand-written `copy()`, and keep longest road incremental. The other option is to make
  a move on a single state and undo it afterwards. That's only worth building if copying dominates.
- **Leaked hidden information is a common bug** in other Catan environments, and it makes bots hard to
  compare fairly. Our view isolation and conformance checks (FR-05) are a real advantage. Keep them strict.
- **Hidden information in practice:** in 1v1 a careful player can count cards exactly. In 4-player,
  steals between other players hide cards, so you track the set of possible hands, and with
  `hide_bank_cards` the bank stack sizes are hidden too. That's the belief tracker in 5.1. Dev cards can be
  modelled with a hypergeometric distribution from the known deck.
- **Settings to carry from Colonist replays:** discard limit, friendly robber, dice setting, VP target,
  hide bank cards, max players. These are all in `GameSettings`.

**Speed reference points (NFR-04)**
- Catanatron in Python: about 1,000 random-vs-random games in 11 s, about 90 games/s, at 10 VP, 1v1.
- A Rust AlphaZero framework: 100,000 random games in 12 s. Its strong bot ran about 100k position
  evaluations/s using batched neural-network inference on GPUs.
- Expect a Python Catan module in the tens to low hundreds of random games per second. A Rust port of
  the hot paths (I-6) is the long-term route to fast self-play.
- A GPU (CUDA) speeds up neural-network evaluation, not the game rules. Rules code is branchy and runs
  one step at a time, so it runs on the CPU, and games per second comes from the CPU and the language.
- **If there's a rewrite, use Rust.**
  - Go would probably be about 20–50× faster than Python and somewhat slower than Rust, because of its
    garbage collector and fewer low-level controls.
  - The deciding factor is Python interop. Rust compiles into a normal Python module (PyO3 / maturin),
    so a Rust engine can sit behind the same `Game` interface and work with numpy, gymnasium and torch.
    Go has no comparable route.
  - Port only the profiled hot paths, after the Python version is correct.

**Bots and training**
- Catanatron's strongest bot is alpha-beta at depth 2 with a hand-tuned value function. Depth 3 was
  slower but not stronger. Most PPO attempts won about 10% or less against it.
- What has worked:
  - **Imitation, then PPO:** first learn to copy the value-function bot, then improve with PPO against
    it. Reported about a 70% win rate against alpha-beta depth 2.
  - **AlphaZero + information-set determinization:** gumbel exploration, a 4-layer graph network of about
    3M parameters, and around 1,300–1,400 input features. Around 1900 Colonist Elo after about 48 h on two
    high-end GPUs.
- Practical lessons:
  - Rich, redundant features (production per number, distance to the next settlement spot, road network
    features) speed up learning more than a bigger model does.
  - Batch the leaf positions in MCTS before running the neural network on them.
  - Track average game length as a training metric: it drops as the bot gets stronger.
- **Player trading is unsolved.** No public bot trades with players. Search bots can loop forever on
  trades, which is why we cap offers per turn. RL bots learn to spam bad offers unless rejected offers
  cost something, so the reward needs a small penalty per rejected offer (with total shaping kept below
  the win reward). Trading is where a 4-player project can stand out.
- **The dice model matters.** Strong players use Colonist's balanced dice when deciding how many cards to
  hold. Bots should get the dice model's current probabilities as features once `dice_mode="balanced"`
  exists (I-12).

### 5.4 Later phases

- Learning bots (PPO, AlphaZero-style) and PyTorch, once I-5 is decided. The RTX 5070 needs CUDA wheels that support it.
- Colonist replay (FR-03 loader), balanced dice (I-12), the Arena with Elo (FR-16), and hard time limits
  enforced by running bots in worker processes (NFR-01).

## 6. Writing the Catan rules module

1. Create `src/mayflower_engine/games/catan/` and implement `Game`. Call `require_supported` in `new_game`.
2. Store rule constants as data tables, keep the state as lists and ints, and write `copy()` by hand.
3. Only generate a bounded set of player-trade offers (`max_trade_offers_per_turn`), and make discards and
   Year of Plenty one resource per move (5.3).
4. Store an `EventLog` in the state, log every action (private where a card is hidden), and expose
   `view.events = state.events.view(seat)`.
5. Register the module in `games/__init__.py`.
6. Add `tests/test_catan.py` with `run_all(CatanGame(), GameSettings.standard())` plus position-specific
   rule tests.
7. Run `bench` to get the real NFR-04 number, then write an `RLAdapter`.

## 7. Environment and supply chain

- uv-managed. Python is pinned to 3.12 (`.python-version`) for the widest PyTorch and RL library support.
- `exclude-newer = "7 days"`: uv never installs a release published in the last week.
- The core has no runtime dependencies. The `rl` extra adds numpy and gymnasium; the dev group adds pytest.
- Run `uv audit` after any dependency change, and look up recent incidents before adding a new package.
  Avoid `lightning` 2.6.2 and 2.6.3, which were compromised in April 2026.

## 8. Open decisions

| # | Decision | Owner |
| --- | --- | --- |
| 1 | Robber deals: a later phase, with the promise and option design in 5.2 (decided 2026-11-05); FUNC_SPECS needs to say which phase | Team / FUNC_SPECS |
| 2 | Which learning bots are due in Q1 (I-5), which decides when PyTorch is added | Team |
| 3 | CI on GitHub Actions (uses third-party actions) | Caleb |
| 4 | Rule source of truth: rulebook or Colonist settings (I-12) | Team |

## 9. File map

```
engine/
├── pyproject.toml                uv project; rl extra (numpy, gymnasium); dev group (pytest); 7-day cooldown
├── uv.lock                       pinned dependency versions
├── .python-version               3.12
├── .gitignore
├── README.md                     setup and layout
├── .github/workflows/ci.yml      pytest + bench smoke test
├── docs/
│   └── ARCHITECTURE.md           this document
├── src/mayflower_engine/
│   ├── __init__.py               public API re-exports
│   ├── settings.py               GameSettings, presets, require_supported, JSON round-trip
│   ├── game.py                   Game interface (the contract), IllegalMove
│   ├── events.py                 EventLog / EventView: shared append-only log, per-seat visibility
│   ├── belief.py                 BeliefTracker interface: update(event), possible_hands(seat), sample(rng)
│   ├── runner.py                 play_game, GameResult, BotError, MoveLimitExceeded
│   ├── bench.py                  CLI: games/sec, moves/sec, wins by seat (NFR-04)
│   ├── games/
│   │   ├── __init__.py           registry: GAMES, get_game(name)
│   │   ├── toy.py                small original game; harness test fixture
│   │   └── catan/
│   │       ├── __init__.py       CatanGame(Game)
│   │       ├── constants.py      data tables: resources, costs, dev deck, piece limits, bank size
│   │       ├── topology.py       19 hexes / 54 corners / 72 edges, adjacency tables, precomputed masks
│   │       ├── board.py          layouts (standard, seeded random), numbers, ports, robber start
│   │       ├── state.py          CatanState (flat lists/ints), hand-written copy()
│   │       ├── moves.py          move constructors; fixed move ordering
│   │       ├── rules.py          legal_moves / apply: setup, roll, discard (1 card/move), robber,
│   │       │                     build, dev cards, maritime + player trades, win check
│   │       ├── scoring.py        VP, longest road (incremental), largest army
│   │       ├── views.py          CatanView + view_for + determinize
│   │       ├── belief.py         Catan hand tracker (implements belief.BeliefTracker)
│   │       ├── dice.py           random 2d6; balanced dice (I-12)
│   │       └── deals.py          robber deals: deal window, promise ledger, enforce option
│   ├── bots/
│   │   ├── __init__.py
│   │   ├── base.py               Bot interface: reset, choose_move, win_probability
│   │   ├── random_bot.py         baseline
│   │   ├── flat_mc.py            benchmark bot (determinize + random rollouts)
│   │   ├── value_bot.py          hand-built value function, 1-ply
│   │   ├── alphabeta.py          expectimax / alpha-beta (I-5)
│   │   ├── ismcts.py             MCTS over determinized states
│   │   ├── ppo/                  policy, training script (needs torch)
│   │   └── alphazero/            network, batched search, self-play loop (needs torch)
│   ├── rl/
│   │   ├── __init__.py
│   │   ├── env.py                SeatEnv (Gymnasium), RLAdapter interface, action masks
│   │   ├── toy_adapter.py        toy move ↔ action id, view → observation
│   │   └── catan_adapter.py      Catan action space + feature encoding
│   ├── testing/
│   │   ├── __init__.py
│   │   └── conformance.py        run_all + individual checks every rules module must pass
│   ├── arena/
│   │   ├── tournament.py         seat rotation, many seeds, worker processes (hard time limits)
│   │   └── elo.py                ratings, leaderboard
│   └── replay/
│       ├── colonist_loader.py    recording → board + GameSettings
│       ├── replayer.py           recorded actions + recorded dice → engine states
│       └── chat_deals.py         link recorded chat promises to the trades that follow
└── tests/
    ├── test_settings.py
    ├── test_toy_game.py          conformance on toy + trading, events, flat MC vs random
    ├── test_rl_env.py            Gymnasium checker, masks, forced moves, penalty mode
    ├── test_events.py            EventLog: fork-on-write, visibility, pickling
    ├── test_belief.py            true hand always in the possible set (self-play)
    ├── test_catan_conformance.py run_all(CatanGame(), ...) for 4p and 1v1/15 VP settings
    └── test_catan_rules.py       position tests: distance rule, discards, robber, longest road, ...
```

### 9.1 Dependencies between modules

Arrows mean "imports / calls". Tools sit above the `Game` interface and rules modules plug in below it.
Bots never import a rules module's state.

```
  bench.py ──────┐
  arena/ ────────┼──► runner.py ──► bots/base.py ◄── random_bot, flat_mc, alphabeta, ismcts, ppo, alphazero
  replay/ ───────┤                                    │
  rl/env.py ─────┤                                    │ search bots: determinize, apply, copy
  testing/ ──────┤                                    │
                 ▼                                    ▼
          ┌─────────────────────── game.py (Game interface) ───────────────────────┐
          │  uses: settings.py, events.py, belief.py                               │
          └────────────────────────────────────────────────────────────────────────┘
                 ▲ implemented by
          games/toy.py, games/catan/      (registered in games/__init__.py → get_game("catan"))
```

### 9.2 What happens during one move

```
runner.play_game / rl.SeatEnv
  1. seat  = game.current_player(state)        rules
  2. legal = game.legal_moves(state)           rules (fixed order)
  3. view  = game.view_for(state, seat)        rules → copies public data + own hand + EventView
  4. move  = bot.choose_move(view, legal, t)   bot (may call game.determinize(view) and simulate copies)
  5. check move ∈ legal                        runner (BotError otherwise)
  6. state = game.apply(state, move)           rules → new state, RNG inside, events appended
  7. repeat until game.is_terminal(state); payoff = game.returns(state)
```
