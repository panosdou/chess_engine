# Chess Engine

A standard-chess engine in C++20, developed one verified layer at a time. The initial goal is a trustworthy position and legal-move core; UCI and search follow Perft validation.

## Start here

1. Configure: `cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug`
2. Build: `cmake --build build`
3. Test: `ctest --test-dir build --output-on-failure`

Use `-DCHESS_ENABLE_SANITIZERS=ON` with GCC or Clang to enable AddressSanitizer and UndefinedBehaviorSanitizer. Milestone 1 provides only the build scaffold and smoke test; chess primitives are deliberately not present yet.

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
├── CMakeLists.txt
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DECISIONS.md
│   └── ROADMAP.md
├── src/
│   └── main.cpp
└── tests/
    └── smoke_test.cpp
```

`chess_core` is an interface target until milestone 2 introduces its first source-level API. `chess_engine` is a no-op executable, and CTest invokes the `chess_tests` smoke-test executable. Add source files only when their milestone begins.

## Reference material

- [Chess Programming Wiki: Board Representation](https://www.chessprogramming.org/Board_Representation)
- [Chess Programming Wiki: Move Generation](https://www.chessprogramming.org/Move_Generation)
- [Chess Programming Wiki: Perft Results](https://www.chessprogramming.org/Perft_Results)

Consult the relevant primary source for each subsystem as it is implemented. Record measurements before replacing a simple, validated algorithm with a faster one.
