# Chess Engine

A standard-chess engine in C++20, developed one verified layer at a time. The initial goal is a trustworthy position and legal-move core; UCI and search follow Perft validation.

## Start here

1. Put `AGENTS.md` and this file at the repository root, and put the other supplied files in `docs/`.
2. Ask Codex: **“Read AGENTS.md and docs/*.md. Implement only roadmap milestone 1: CMake, CTest, compiler warnings, and a minimal smoke test. Run the configured checks and report results. Do not start primitive types yet.”**
3. Review the result, then request milestone 2. Continue in order; each milestone has an exit check.

The documents describe intended APIs before their implementation. They are specifications, not claims that source files or build commands already exist.

## Project documents

| File | Purpose |
| --- | --- |
| `AGENTS.md` | Instructions Codex reads when working in this repository. |
| `docs/ARCHITECTURE.md` | Current technical design and module contracts. |
| `docs/DECISIONS.md` | Recorded choices, alternatives, and consequences. |
| `docs/ROADMAP.md` | Ordered implementation steps and acceptance criteria. |

## Planned layout

```text
.
├── AGENTS.md
├── README.md
├── CMakeLists.txt                 # created in milestone 1
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DECISIONS.md
│   └── ROADMAP.md
├── src/
│   ├── main.cpp
│   └── chess/
│       ├── types.h
│       ├── bitboard.h
│       ├── attacks.h
│       ├── move.h
│       ├── zobrist.h
│       ├── position.h
│       ├── movegen.h
│       └── perft.h
└── tests/
```

Add source files when their milestone begins. The planned CMake targets are `chess_core`, `chess_engine`, and `chess_tests`; CTest invokes the latter. The normal development loop will be configure, build, then `ctest --test-dir build --output-on-failure`, once milestone 1 creates the build.

## Reference material

- [Chess Programming Wiki: Board Representation](https://www.chessprogramming.org/Board_Representation)
- [Chess Programming Wiki: Move Generation](https://www.chessprogramming.org/Move_Generation)
- [Chess Programming Wiki: Perft Results](https://www.chessprogramming.org/Perft_Results)

Consult the relevant primary source for each subsystem as it is implemented. Record measurements before replacing a simple, validated algorithm with a faster one.
