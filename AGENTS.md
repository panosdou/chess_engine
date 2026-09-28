# Chess Engine — instructions for Codex

## Mission and scope

Build a strong, maintainable standard-chess engine in C++20, incrementally. Correctness of the board, move generation, and make/unmake comes before search strength and optimization. This file governs work in this repository. Read `docs/ARCHITECTURE.md`, `docs/DECISIONS.md`, and `docs/ROADMAP.md` when they exist; keep them aligned with the implementation.

Do not implement the whole engine in one pass. Work on the smallest coherent milestone requested by the user. If no milestone is specified in a fresh repository, begin with CMake/CTest scaffolding, then primitive types and square mapping. Do not advance into attacks, moves, or Position in that first pass. Explain proposed architectural changes before adopting them, and record accepted fundamental changes in `docs/DECISIONS.md`.

## Sources and engineering method

- Consult the relevant [Chess Programming Wiki](https://www.chessprogramming.org/) pages before designing a chess-specific subsystem. Check primary implementation sources such as Stockfish when useful, but do not copy complexity or code without a concrete reason. Link sources in the change summary or decision record.
- For each subsystem, explain its purpose, chosen design, meaningful alternatives, dependencies, tests, and how to run them. Implement and verify the current layer before moving to the next.
- Prefer a simple correct backend behind a stable API. Benchmark and profile before adding a faster backend; rerun correctness tests afterward.
- Never silently change a representation, move encoding, state ownership rule, FEN policy, or public API. Update architecture and ADRs with the reason and consequences.
- Keep runtime hot paths allocation free. Avoid copying a full `Position` at every search node. Use fixed caller-owned move buffers and per-ply state storage.

## Baseline design

- **Mapping:** little-endian rank-file mapping: `A1 = 0`, `H1 = 7`, `A2 = 8`, `H8 = 63`; north is `+8`. Document and test file/rank conversion and pawn directions.
- **Types:** `Bitboard` and `Key` are `uint64_t`; define compact `Color`, `PieceType`, `Piece`, `Square`, `File`, and `Rank` types and `constexpr` conversion helpers. Prefer clear low-level code to elaborate wrappers.
- **Board:** a `Piece board[64]` mailbox plus piece-type bitboards (including an `ALL` occupancy entry) and two color bitboards. Obtain a color/type set by intersection. Initially derive each king square from its king bitboard. Keep redundant representations exactly synchronized.
- **Move:** a 16-bit value with 6-bit source, 6-bit destination, and 4 bits for normal/promotion/en-passant/castling metadata. Encode all four promotion choices. Standard castling uses king destination (`e1g1`, etc.). Do not use move strings inside the core or encode the captured piece without a demonstrated need.
- **State:** mutate one `Position` with `doMove(Move, StateInfo&)` and `undoMove(Move)`. `StateInfo` holds the prior link and state needed for exact reversal, including captured piece, castling rights, en-passant state, halfmove clock, and hash. Its lifetime must outlast all descendant moves that refer to it. Never keep pointers into a temporary or reallocated state container.
- **Attacks:** expose pawn, knight, king, bishop, rook, and queen attack functions. Precompute non-sliding geometry; begin sliding attacks with tested ray scans behind `bishopAttacks(square, occupancy)` and `rookAttacks(square, occupancy)`. Keep the API independent of the future magic/PEXT choice.
- **Attack queries:** `Position` supplies `attackersTo` and `isSquareAttacked` using the shared attack primitives. Use this one system for checks, king safety, castling, and legal-move filtering.
- **Move generation:** initially generate pseudo-legal moves into a fixed caller-provided buffer, then filter with do/check/undo. Castling must check the starting, transit, and destination squares; en passant must account for removal of both pawns and discovered attacks. A later direct-legal generator or staged move picker may replace the implementation without changing callers.
- **Hashing:** deterministic 64-bit Zobrist keys from a fixed, specified PRNG seed. Cover piece-square, side, castling rights, and relevant en-passant state. Provide a slow from-scratch computation and compare it with incremental updates in Debug tests.
- **FEN:** parse and serialize standard FEN with explicit validation. Define and test the exact en-passant normalization policy before implementing it; distinguish a FEN target square from a legal capture and ensure hashing/repetition semantics agree. Do not promise a byte-identical FEN round trip if serialization canonicalizes fields.
- **Scope:** standard chess first. Do not add Chess960, NNUE, SMP, tablebases, or speculative directories until a milestone requires them.

## Module boundaries

`types` defines primitives; `bitboard` defines pure bit operations and geometry; `attacks` defines attack geometry and sliding backends; `move` defines compact encoding; `position` owns board/state, FEN, attack queries, and make/unmake; `zobrist` provides keys and scratch recomputation; `movegen` writes moves into caller-owned storage; `perft` checks move generation and state transitions. Keep `attacks` independent of `Position`. Build a reusable `chess_core` target, a thin `chess_engine` executable, and a `chess_tests` target using CMake and CTest.

## Position invariants

In Debug builds, validate the following after parsing and representative make/undo operations:

1. Every mailbox piece appears in exactly its color and type bitboards; empty squares appear in none.
2. Color sets are disjoint; piece-type sets are mutually disjoint; their union equals `ALL` and the union of color sets.
3. Each color has exactly one king in a valid playable position; `sideToMove` and castling bits are valid.
4. An en-passant target, when present, is empty, has the rank appropriate to the side to move, and satisfies the documented normalization policy.
5. The incremental Zobrist key equals a recomputation from the current state.
6. `doMove` followed by `undoMove` restores every observable field and representation exactly, including FEN-relevant counters, the hash, and the state-chain link.

Distinguish a valid playable position from an intermediate or rejected FEN parse; never run king-dependent queries on an invalid position.

## Validation gates

Add focused tests for square mapping, bit operations, attack tables and sliding rays, FEN, special moves, castling rights, en passant, promotions, state reversal, and hash consistency as the respective layer is built. Use Debug assertions and run CTest; use sanitizers where configured. Once legal move generation exists, implement `perft` and `divide` without caching or shortcuts. Check established Chess Programming Wiki positions, including castling, pins, checks, promotions, and en passant. Starting-position counts at depths 1–6 are `20`, `400`, `8902`, `197281`, `4865609`, and `119060324`; use practical depths during routine tests and run deeper checks as an explicit validation milestone. Do not begin serious search with unexplained Perft failures.

## Incremental order

1. CMake, C++20, warnings, CTest, optional sanitizers, and minimal documentation.
2. Primitive types, A1=0 mapping, and tests.
3. Bitboard operations, masks, and tests.
4. Non-sliding attacks, then reference sliding attacks and tests.
5. Board representation, consistency checks, and FEN.
6. Move encoding, Zobrist scratch hash, StateInfo, and make/unmake (normal, captures, then special moves); incremental hash and restoration tests.
7. Pseudo-legal generation, castling, legality filtering, and targeted tests.
8. Perft/divide, published regression positions, and a measured performance baseline.
9. Minimal UCI shell, then baseline search and measured enhancements.

Prefer reviewable commits at these boundaries when working in a Git repository. Do not commit, push, or publish unless the user requests it or repository instructions authorize it. At the end of each task, report what changed, the commands and results used to verify it, remaining limitations, and the next logical milestone.
