# Architecture

**Status:** intended design. API names below are contracts to implement and refine as their milestone begins; this document must be updated to reflect actual code. Standard chess only, C++20.

## Goals and dependencies

Build a correct, testable core that allows later search to call generation, make, and undo at high frequency without heap allocation or full-position copies. Optimize measured bottlenecks only after correctness gates pass.

```text
types ──> bitboard ──> attacks ──> position ──> movegen ──> perft
   └──────────────> move ────────────┘
              zobrist <── position (scratch hash reads Position)
```

The last relation needs a careful header boundary: `zobrist.h` declares key accessors and forward-declares `Position` for `compute(const Position&)`; `zobrist.cpp` includes `position.h`. `position.cpp` consumes the Zobrist accessors. No cyclic header includes. `attacks` never includes or depends on `Position`.

Create a reusable `chess_core` library, a thin `chess_engine` executable, and a `chess_tests` CTest executable. No speculative empty search, NNUE, UCI, or platform directories.

## Representation

Squares use little-endian rank-file (LERF): `A1=0`, `B1=1`, `H1=7`, `A2=8`, `H8=63`. Thus `fileOf(s)=s&7`, `rankOf(s)=s>>3`, north `+8`, south `-8`; directional shifts must mask file wraparound. `Bitboard` and `Key` are `std::uint64_t`; only valid squares `0..63` may be shifted into a bitboard. `SQ_NONE` is a sentinel, never a shift operand.

`Position` owns:

```cpp
Piece board[64];
Bitboard byType[7];    // index 0 = ALL; indices 1..6 = P,N,B,R,Q,K
Bitboard byColor[2];   // WHITE, BLACK
Color sideToMove;
int gamePly;            // derived from FEN fullmove and side; increments/decrements
StateInfo* state;
```

`pieces(color, type) = byColor[color] & byType[type]`; `occupied() = byType[ALL]`. Derive `kingSquare(color)` from the single set bit in `pieces(color, KING)`. `board[s]` answers piece-on-square queries. Piece placement/removal/movement helpers alone mutate all three board representations; test that they agree. Do not expose raw mutable arrays.

`gamePly = 2 * (fullmoveNumber - 1) + (sideToMove == BLACK)` for accepted FEN. `toFEN` recovers `fullmoveNumber = gamePly / 2 + 1`. Use a sufficiently wide signed integral type and reject FEN counter overflow; avoid narrowing large valid halfmove clocks. Halfmove clock belongs in `StateInfo`.

### State lifetime and make/unmake

The root `StateInfo` is owned by the caller that loads a Position; each `doMove` receives a distinct caller-owned child state that remains alive until undo. A `Position` is non-owning with respect to `StateInfo` and cannot outlive its active state chain. Prevent accidental Position copies or explicitly define safe copy semantics before allowing them. When loading FEN, initialize the root state and reset any old chain.

The initial `StateInfo` contains `Key key`, four-bit `CastlingRights`, `Square epSquare`, halfmove clock, `Piece capturedPiece`, and `StateInfo* previous`. Only the current state's fields are read for current rights and hash. The physical board, side, and game ply are mutated reversibly. `doMove` links and fills the child; `undoMove` restores board, side, ply, and previous state exactly. Later state fields for repetition, cached checks, and NNUE can be added when needed.

`doMove` takes a structurally valid pseudo-legal move for the current Position. It does **not** assert that the moving side's king remains safe. Legal generation calls it, checks the king, and undoes it. Never allow a pseudo move to capture the enemy king. Generate castling only after confirming correct king/rook, rights, empty path, and king safety on starting and transit squares; after do/check/undo also reject attacked destination. For en passant, remove the captured pawn from its actual square before testing king safety. King-capture legality must assess danger with post-move occupancy.

Castling-right masks update on king move, original rook move, or capture of an original rook on its home square. Rights never reappear because another rook occupies that square. Halfmove resets on pawn moves and captures. Clear the FEN en-passant square on every move except a double pawn push. Undo restores the exact prior rights, en-passant target, clock, and key via the state chain.

## Move and bitboard APIs

`Move` is a 16-bit value: bits `0..5` destination, `6..11` source, `12..13` promotion selector (N/B/R/Q), and `14..15` kind (normal/promotion/en-passant/castling). Normal moves have the lower metadata bits zero. A zero raw value can represent `MOVE_NONE` because a legal chess move never has identical source and destination; validate this before using that sentinel. Castling uses king-to-destination encoding (`e1g1`, `e1c1`, `e8g8`, `e8c8`). Captured piece is read from Position or StateInfo, with en passant special handling. Strings exist only at FEN/UCI or diagnostic boundaries.

`bitboard` exposes `squareBB`, file/rank masks, safe directional shifts, `popcount`, `lsb`, and `popLsb`; `lsb(0)` must have an explicit precondition or sentinel result, documented and tested. C++20 `<bit>` is preferred where portable and appropriate.

## Attack and Position query APIs

```cpp
namespace Attacks {
    Bitboard pawn(Color, Square);
    Bitboard knight(Square);
    Bitboard king(Square);
    Bitboard bishop(Square, Bitboard occupancy);
    Bitboard rook(Square, Bitboard occupancy);
    Bitboard queen(Square, Bitboard occupancy);
}

class Position {
    Piece pieceAt(Square) const;
    Bitboard occupied() const;
    Bitboard pieces(Color) const;
    Bitboard pieces(PieceType) const;
    Bitboard pieces(Color, PieceType) const;
    Square kingSquare(Color) const;
    Bitboard attackersTo(Square, Bitboard occupancy) const;
    bool isSquareAttacked(Square, Color) const;
    bool inCheck(Color) const;
    void doMove(Move, StateInfo&);
    void undoMove(Move);
};
```

Non-sliding attack tables can be `constexpr`. Start sliders with ray walking; retain the occupancy argument so magic or PEXT can replace the backend. `attackersTo(square, occupancy)` uses the piece sets of this Position with supplied occupancy for slider ray obstruction. **It does not pretend the piece sets were also updated.** For hypothetical moves that remove, move, or add attackers, use a helper with explicit adjusted sets or make/unmake before querying. This matters for king moves and en passant.

## FEN and en passant

`fromFEN` accepts six standard fields, validates syntax, numeric ranges, one king per side, pawns off first/eighth ranks, castling-right consistency with home king/rook, and coherent en-passant geometry (correct rank, empty target, correct pawn behind it, empty origin). It must reject impossible check-state combinations that would break move generation, but need not prove a full legal game history. Specify clear errors, leave no partially initialized Position on failure, and test rejected inputs. `toFEN` emits canonical piece placement, `KQkq` rights order, preserved en-passant target, halfmove, and fullmove. Exact whitespace or castling-letter ordering in input need not be retained.

**Chosen policy:** retain `epSquare` after every double pawn push and from coherent FEN even if there is no potential capturer. This matches standard FEN output. En-passant move generation uses that field, and the legal-move filter rejects pinned/discovered-check cases. For Zobrist, include the EP-file key only if the side to move has at least one **legal** en-passant capture; if none, hash as if EP were absent. An internal helper `hashableEpFile(const Position&) -> optional<File>` determines this by testing each candidate with correctly updated occupancy and piece sets; avoid recursive dependence on `doMove`/`key()` inside hash computation. Test FEN round trips such as the position after `e2e4`, positions with adjacent but pinned EP pawns, and the double-pawn-removal discovered-check case. The position key excludes move counters and raw redundant EP targets.

## Zobrist

Use fixed-seed SplitMix64 (specified unsigned-64 arithmetic) to generate 12×64 piece-square keys, one side key, 16 castling-state keys, and eight EP-file keys. `Zobrist::compute(Position const&)` recomputes from the mailbox and state. Piece changes update the key by XOR; remove old and add new castling, side, and hashable-EP components at the right points. `StateInfo::key` is the key of its associated position. Compare incremental and scratch keys in Debug tests, including on both sides of undo. This key supports repetition later; its zero-collision guarantee is probabilistic, as with ordinary 64-bit Zobrist hashing.

## Move generation and Perft

```cpp
namespace MoveGen {
    Move* generatePseudoLegal(const Position&, Move* first);
    Move* generateLegal(Position&, Move* first);
}
using NodeCount = std::uint64_t;
NodeCount perft(Position&, int depth);  // depth 0 => 1
void divide(Position&, int depth, std::ostream&);
```

The caller supplies capacity for at least `MAX_MOVES = 256` entries. Never write beyond it; assertions or bounded wrappers can help in Debug. Initial legal generation may use a separate local 256-entry pseudo buffer, then write accepted moves into the caller's buffer; do not overwrite unread candidate moves during filtering. For each candidate: create child `StateInfo`, do move, test the mover's king, undo, append if legal. Castling's starting/transit-square tests occur before do move. All four promotion choices, double pawn pushes, captures, en passant, and both castlings are required. Later specialized evasions and staged generation may replace the implementation while preserving the API.

Perft counts leaf nodes, not interior nodes, with `perft(depth=0)=1`; divide lists legal root moves and their subtree counts. No TT, bulk count, or parallelism for the first correctness baseline. Use [CPW published results](https://www.chessprogramming.org/Perft_Results), especially positions 2–6 and starting depths 1–6. Targeted single-position tests remain necessary because start-position Perft does not exercise all rules at shallow depth.

## Debug and future work

`assertValid` checks mailbox/bitboard agreement, disjoint occupancy, exactly one king per side for playable positions, state pointer, side, castling rights and geometry, and incremental versus scratch hash when hashing exists. Check undo by snapshotting Position's observable state and comparing after every tested do/undo sequence, not by blindly comparing a pointer-bearing object with `memcmp`.

After Perft passes: add UCI shell; then negamax/alpha-beta, quiescence, iterative deepening, transposition table, and move ordering. Profile before direct legal generation, magic/PEXT sliders, cache layout changes, NNUE accumulators, or threading. Update this document when reality differs.

## Sources

- [Chess Programming Wiki: Square Mapping Considerations](https://www.chessprogramming.org/Square_Mapping_Considerations)
- [Chess Programming Wiki: Bitboards](https://www.chessprogramming.org/Bitboards)
- [Chess Programming Wiki: Make Move](https://www.chessprogramming.org/Make_Move)
- [Chess Programming Wiki: En passant](https://www.chessprogramming.org/En_passant)
- [Chess Programming Wiki: FEN](https://www.chessprogramming.org/Forsyth-Edwards_Notation)
- [Chess Programming Wiki: Zobrist Hashing](https://www.chessprogramming.org/Zobrist_Hashing)
- [Chess Programming Wiki: Perft Results](https://www.chessprogramming.org/Perft_Results)
