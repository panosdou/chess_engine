# Architectural decisions

These are design decisions for the project, not evidence that code already implements them. Each ADR is **Accepted for the initial implementation**. When an ADR changes, amend its status, reason, consequences, and date; update `ARCHITECTURE.md` and tests together. Chess Programming Wiki references are starting points, not a substitute for verifying behavior against tests.

## ADR-001 — Mailbox and intersected bitboards

- **Context:** Occupancy and set operations favor bitboards; answering which piece occupies a square favors a mailbox.
- **Decision:** Store `board[64]`, `byType[ALL,P,N,B,R,Q,K]`, and `byColor[WHITE,BLACK]`. Intersect sets for one color and type. Mutate representations through shared piece helpers.
- **Alternatives:** Twelve color/type boards plus three occupancy boards; mailbox only; bitboards only.
- **Why:** Few mutable boards, constant-time piece lookup and efficient attacks.
- **Consequences:** Redundant data requires Debug consistency assertions; type/color intersection is an extra bitwise operation.
- **Revisit:** Benchmark a different layout only after profiling.
- **Reference:** [Bitboards](https://www.chessprogramming.org/Bitboards).

## ADR-002 — A1 is bit zero

- **Context:** Every rank/file calculation and pawn shift assumes a square layout.
- **Decision:** Little-endian rank-file mapping, `A1=0`, `H8=63`, rank increments by eight.
- **Alternatives:** A8=0, file-major, rotated mappings.
- **Why:** Simple file/rank arithmetic and conventional bitboard examples.
- **Consequences:** Edge-file masking is mandatory for diagonal shifts; parsing FEN maps ranks 8 through 1 into this layout.
- **Revisit:** Only with a compelling reason before significant implementation.
- **Reference:** [Square Mapping Considerations](https://www.chessprogramming.org/Square_Mapping_Considerations).

## ADR-003 — Make/unmake and caller-owned state chain

- **Context:** Search and Perft enter and leave large numbers of nodes.
- **Decision:** Mutate one Position with `doMove(move, childState)`/`undoMove(move)`; link stable caller-owned `StateInfo` records. Root state must remain alive while Position uses it.
- **Alternatives:** Copy Position per node; reconstruct irreversible state by replaying history; heap allocation per node.
- **Why:** Low overhead and exact saved castling, en-passant, clock, captured-piece, and key state.
- **Consequences:** Lifetime contract and exact restoration are critical. Debug snapshot and randomized reversible-sequence tests are required. Avoid implicit Position copies with borrowed state pointers.
- **Revisit:** Layout and contents may change, not the correctness contract.
- **Reference:** [Make Move](https://www.chessprogramming.org/Make_Move).

## ADR-004 — Compact 16-bit Move

- **Context:** Moves are stored and ordered at every search ply.
- **Decision:** Bits `0..5` destination; `6..11` source; `12..13` promotion selector; `14..15` move kind. Castling records the king's destination. Promotion selector maps N, B, R, Q explicitly. Captured piece lives in Position/StateInfo.
- **Alternatives:** String moves, large records including captured piece, Chess960 king-to-rook castling encoding.
- **Why:** 16 bits covers every standard-chess move with no node-level allocation.
- **Consequences:** Validate special-move tags and never treat a string as an internal move. Chess960 will need an explicit design review.
- **Revisit:** If a supported variant or benchmark demonstrates a different encoding is necessary.
- **Reference:** [Encoding Moves](https://www.chessprogramming.org/Encoding_Moves).

## ADR-005 — Pseudo-legal generation with legality filtering first

- **Context:** Pins, king moves, checks, castling, and en passant complicate direct legal generation.
- **Decision:** Generate pseudo-legal moves into caller-owned storage, then do/check/undo; check castling's starting and transit squares separately before making it.
- **Alternatives:** Direct legal generation with pin and checker masks immediately; legal generation by copying Position.
- **Why:** Establish a small correctness baseline and test it with Perft.
- **Consequences:** Extra make/undo work per candidate. Do not generate captures of an opposing king. `generateLegal` needs separate candidate/output storage or another safe compaction method.
- **Revisit:** Replace hot paths only after profiling and rerunning the same regressions.
- **Reference:** [Move Generation](https://www.chessprogramming.org/Move_Generation).

## ADR-006 — Stable sliding-attack interface

- **Context:** Slider attacks need current occupancy; faster lookup techniques have build, memory, and platform tradeoffs.
- **Decision:** Start with ray scanning behind `Attacks::bishop/rook/queen(square, occupancy)`.
- **Alternatives:** Magic bitboards, PEXT, hyperbola quintessence at the outset.
- **Why:** Easy to check against reference rays; future backend swaps do not affect Position or move generation.
- **Consequences:** Early Perft speed is not representative of final engine performance.
- **Revisit:** After correctness and benchmarks compare alternatives on target hardware.
- **Reference:** [Magic Bitboards](https://www.chessprogramming.org/Magic_Bitboards).

## ADR-007 — Reproducible incremental Zobrist hashing

- **Context:** Repetition and future transposition tables need cheap position identities.
- **Decision:** Generate fixed 64-bit keys with specified SplitMix64 seed and unsigned arithmetic; cover piece-square, side, all sixteen castling states, and hashable EP file. Update incrementally; compute independently from scratch for tests.
- **Alternatives:** Library RNG with unspecified cross-platform sequence; full rehash after every move; omit castling/EP state.
- **Why:** Deterministic debugging and fast move updates without losing rule-relevant state.
- **Consequences:** Key tables and seed become part of reproducibility. Hash collisions remain theoretically possible. EP determination must not recursively depend on hash computation.
- **Revisit:** Hash width and extra keys if a later subsystem demands them.
- **Reference:** [Zobrist Hashing](https://www.chessprogramming.org/Zobrist_Hashing).

## ADR-008 — Caller-owned fixed move storage

- **Context:** Perft and future search generate moves at each ply.
- **Decision:** Caller provides room for `MAX_MOVES=256`; generator returns one-past-last pointer. Document and assert capacity during development.
- **Alternatives:** `std::vector` per node, heap-linked moves, a custom allocator immediately.
- **Why:** Simple traversal with no allocations in hot paths.
- **Consequences:** Caller honors capacity, and legal filtering must not overwrite unread candidates. Search will allocate buffers by ply, not return pointers to temporary storage.
- **Revisit:** If other variants or measured cache behavior require a different container.
- **Reference:** [Move Generation](https://www.chessprogramming.org/Move_Generation).

## ADR-009 — Preserve FEN EP target, hash only effective legal EP

- **Context:** Standard FEN records a target after any double pawn push, even when no capture is possible. For repetition, a redundant target must not distinguish positions with identical legal moves; a pseudo-capturing pawn may also be pinned.
- **Decision:** Retain a geometrically coherent target in `StateInfo` and serialize it in FEN. Include its file in the Zobrist key only if at least one legal EP capture exists. Evaluate candidate EP captures on the correctly adjusted board without a dependency cycle through the current key.
- **Alternatives:** Canonicalize the state itself to `SQ_NONE`; hash any raw EP field; hash only when an adjacent pawn exists.
- **Why:** Standard FEN round trips coexist with correct effective-position identity.
- **Consequences:** EP hash calculation costs a small legality test on eligible positions. Incremental hash and scratch hash must share the exact same EP rule. Test pinned EP pawns and discovered horizontal attacks.
- **Revisit:** Only with a documented repetition and FEN compatibility analysis.
- **References:** [FEN](https://www.chessprogramming.org/Forsyth-Edwards_Notation), [En passant](https://www.chessprogramming.org/En_passant).

## ADR-010 — Standard chess and staged feature growth

- **Context:** Chess960, NNUE, threading, and tablebases each add contracts beyond a trustworthy first core.
- **Decision:** Implement standard chess through published Perft validation before UCI/search, then add stronger features in measured stages. Keep the StateInfo extension point for future cached checks and NNUE accumulators.
- **Alternatives:** Implement every future abstraction now; clone a mature engine architecture wholesale.
- **Why:** Smaller layers permit meaningful review, debugging, and profiling.
- **Consequences:** Early engine will not be strong or feature complete; those capabilities follow validation milestones.
- **Revisit:** When user goals change, with explicit scope and acceptance criteria.
- **Reference:** [Perft](https://www.chessprogramming.org/Perft).

## ADR-011 — Separate game state from search history

- **Context:** Fullmove/halfmove counters, repetition, and future search plies have different roles.
- **Decision:** Track fullmove through `gamePly` and halfmove in `StateInfo`; later repetition logic consumes a stable sequence of position keys and rights-effective state. No implicit global game record is needed for the initial Perft core.
- **Alternatives:** Derive fullmove only from side; put all history inside Position; conflate ply counter with halfmove rule clock.
- **Why:** Correct FEN and undo with a bounded current-position interface.
- **Consequences:** FEN initialization must derive `gamePly` without overflow; future UCI `position ... moves` needs to retain the entire chain of states for as long as Position points into it.
- **Revisit:** When search or protocol ownership is implemented.
