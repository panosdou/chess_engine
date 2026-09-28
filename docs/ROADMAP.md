# Roadmap

The steps are implementation milestones, not preexisting files. Codex should work on the requested step only and finish its exit check before moving on. A new repo begins at step 1. Use small, reviewable commits if Git is configured and committing is authorized; the suggested labels below are commit boundaries, not an instruction to commit without authorization.

| Step | Suggested commit label | Deliverable and exit check |
| --- | --- | --- |
| 1 | `build: establish CMake and CTest` | C++20, `chess_core`, `chess_engine`, `chess_tests`, compiler warnings, optional ASan/UBSan, minimal smoke test. Configure/build/CTest work on available compiler. No chess logic. |
| 2 | `core: add primitive types and LERF mapping` | Color, PieceType, Piece, Square, File, Rank, counters, and `constexpr` helpers. Exhaustive 64-square mappings, piece helpers, and sentinel tests pass. |
| 3 | `core: add bitboard primitives` | Masks, safe shifts, bit scans, popcount, pop-lsb. Boundary, zero-input contract, and exhaustive square tests pass. |
| 4 | `core: add non-sliding attacks` | Pawn/knight/king tables for every square and color. Compare against independent reference enumeration. |
| 5 | `core: add reference sliding attacks` | Bishop/rook/queen ray walkers behind occupancy-taking API; test empty, single-blocker, edge, and random occupancies against a separate reference. |
| 6 | `position: add board representation` | Mailbox, type/color boards, piece helpers, root StateInfo ownership contract, Debug invariants. Test place/remove consistency and both kings. |
| 7 | `position: add FEN` | Parse/serialize six fields; reject malformed and incoherent states without changing an existing Position; preserve standard redundant EP target. Round-trip FEN tests pass. |
| 8 | `core: add 16-bit Move` | Constructors, accessors, four promotions, castling/EP tags, sentinel, tests. No UCI parser yet. |
| 9 | `position: add scratch Zobrist` | Fixed SplitMix64 seed and deterministic keys; independent from-scratch key with tests for pieces, side, rights, effective EP. |
| 10 | `position: add StateInfo and quiet do/undo` | Quiet moves, state link, side, counters and rights; snapshot restoration tests. |
| 11 | `position: add captures and pawns` | Captures, single/double pushes, capture resets, EP target, rook-capture rights; restoration tests. |
| 12 | `position: add all special move transitions` | Promotions (including captures), en passant, both castles; restoration tests including discovered EP attack geometry. |
| 13 | `position: maintain Zobrist incrementally` | Every do/undo matches scratch hash; tests include EP hash eligibility changes, castling rights, special moves, long sequences. |
| 14 | `movegen: add pseudo-legal pawns` | Single/double push, captures, four promotions, capture promotions, EP. Tests from FEN. |
| 15 | `movegen: add pieces and king` | Knight, bishop, rook, queen, king pseudo moves; no king capture; targeted tests. |
| 16 | `movegen: add castling` | Rights, pieces, empty path, starting/transit safety; final safety via legal filter. Tests both sides and attack configurations. |
| 17 | `movegen: add legal filtering` | Make/check/undo; in-check, pins, double checks, king moves, EP discovered checks; Position unchanged after generation. |
| 18 | `test: add Perft and divide` | Depth 0 semantics, legal-root divide, known start-position depths 1–4 (routine), 5–6 (extended). |
| 19 | `test: add published Perft regressions` | CPW positions 2–6 to practical depths, targeted pathological EP and castling cases; isolate mismatch with divide; no unexplained failures. |
| 20 | `perf: record baseline` | Reproducible Release build, compiler/CPU/config noted, Perft timings and nodes per second recorded after correctness gates. |
| 21 | `uci: add minimal engine shell` | `uci`, `isready`, `ucinewgame`, `position startpos|fen ... moves ...`, `go perft`, `quit`, with protocol tests. Persist root and move StateInfo lifetime correctly. |
| 22 | `search: add baseline` | Negamax/alpha-beta, quiescence, iterative deepening, legal mate/stalemate scoring, basic clock/stop behavior, then TT and ordering in separate measured changes. |

## Validation suite for step 19

Use the [Chess Programming Wiki Perft Results](https://www.chessprogramming.org/Perft_Results) as the authoritative source for exact FENs and expected counts. Record them as test data with source attribution. Suggested small but discriminating cases:

| Position | Depth | Expected leaf nodes |
| --- | ---: | ---: |
| Initial | 1, 2, 3, 4, 5, 6 | 20; 400; 8,902; 197,281; 4,865,609; 119,060,324 |
| CPW position 2 (Kiwipete) | 1, 2, 3, 4 | 48; 2,039; 97,862; 4,085,603 |
| CPW position 3 | 1, 2, 3, 4 | 14; 191; 2,812; 43,238 |
| CPW position 4 | 1, 2, 3, 4 | 6; 264; 9,467; 422,333 |
| CPW position 5 | 1, 2, 3, 4 | 44; 1,486; 62,379; 2,103,487 |
| CPW position 6 | 1, 2, 3, 4 | 46; 2,079; 89,890; 3,894,594 |

Routine CI should use modest depths; run the longer checks as a separately documented extended command. Perft success alone does not prove hashing, FEN clocks, or undo restoration; keep those tests independent.

## Definition of done for each step

1. Source and relevant docs agree; the milestone introduces no empty future abstractions.
2. Purpose and design choices are explained, with Chess Programming Wiki links where chess rules or algorithms are involved.
3. Focused tests pass through CTest and applicable Debug assertions; report exact commands and results.
4. Any remaining limitation is explicit, and the next step starts only after the current one passes.
