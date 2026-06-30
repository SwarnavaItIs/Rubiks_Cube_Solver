# Rubik's Cube Solver

A C++17 Rubik's Cube project that models a 3x3 cube, applies legal moves, creates solvable scrambles, and solves those scrambles with graph-search based algorithms. The project is built around one common cube interface and several storage models, so the same solver ideas can be tested against different internal representations.

The main workflow is simple: create a solved cube, scramble it by applying legal moves, treat every cube state as a graph node, treat every move as an edge, search until the solved state is found, and return the move sequence that solves the scramble.

## Cube Conventions

| Face | Color | Letter |
| --- | --- | --- |
| Up | White | `W` |
| Left | Green | `G` |
| Front | Red | `R` |
| Right | Blue | `B` |
| Back | Orange | `O` |
| Down | Yellow | `Y` |

Each face supports a clockwise quarter turn, a counterclockwise quarter turn, and a half turn. Across all six faces, this gives 18 legal moves: `L`, `L'`, `L2`, `R`, `R'`, `R2`, `U`, `U'`, `U2`, `D`, `D'`, `D2`, `F`, `F'`, `F2`, `B`, `B'`, and `B2`.

## Repository Structure

```text
Rubiks Cube Project/
|-- .gitignore
|-- CMakeLists.txt
|-- README.md
|-- Planning.md
|-- Korfs_algo.md
|-- main.cpp
|-- Databases/
|   `-- cornerDepth5V1.txt
|-- docs/
|   `-- AAAI97-109.pdf
|-- Models/
|   |-- RubiksCube.h
|   |-- RubiksCube.cpp
|   |-- RubiksCube3dArray.cpp
|   |-- RubiksCube1dArray.cpp
|   `-- RubiksCubeBitboard.cpp
|-- PatternDatabases/
|   |-- math.h
|   |-- math.cpp
|   |-- NibbleArray.h
|   |-- NibbleArray.cpp
|   |-- PermutationIndexer.h
|   |-- PatternDatabase.h
|   |-- PatternDatabase.cpp
|   |-- CornerPatternDatabase.h
|   |-- CornerPatternDatabase.cpp
|   |-- CornerDBMaker.h
|   `-- CornerDBMaker.cpp
`-- Solvers/
    |-- DFS_Solver.h
    |-- BFS_Solver.h
    |-- IDDFS_Solver.h
    `-- IDAstar_Solver.h
```

## Core Architecture

The project separates what a Rubik's Cube can do from how the cube is stored. `RubiksCube` is the abstract base class. It defines faces, colors, moves, move dispatching, inverse moves, printing, random shuffling, and corner helpers. The derived model classes decide how stickers are stored and how rotations change that storage.

This separation is important because solvers do not need to know whether the cube is stored in a 3D array, a 1D array, or a bitboard. A solver only needs a cube type that supports the common operations and, for hash-based search, an equality operator plus a hash function.

## Cube Models

### `RubiksCube`

Files: `Models/RubiksCube.h`, `Models/RubiksCube.cpp`

`RubiksCube` is the shared interface for all cube representations. It defines the `FACE`, `COLOR`, and `MOVE` enums, then declares the virtual methods that every model must implement: color lookup, solved-state checking, and all face rotations. The `move()` function maps a `MOVE` enum to the correct rotation, while `invert()` applies the opposite rotation. This lets search algorithms apply and undo moves without knowing the concrete representation.

The base class also contains representation-independent utilities. `print()` renders a cube net using `getColor()`, so every model can be printed the same way. `randomShuffleCube()` creates a solvable scramble by starting from a valid cube and applying random legal moves. The corner helper functions build compact corner identity and orientation values, which are later used by the pattern database.

### `RubiksCube3dArray`

File: `Models/RubiksCube3dArray.cpp`

`RubiksCube3dArray` stores the cube as `cube[6][3][3]`. This is the easiest representation to understand because it matches the physical cube layout: one face index, one row index, and one column index. The constructor fills each face with its solved color, `getColor()` converts stored letters back to the shared enum, and `isSolved()` checks that each sticker still matches the expected color of its face.

The move algorithm has two parts. First, `rotateFace()` rotates the selected 3x3 face clockwise by copying its old values into temporary storage and writing them back in rotated positions. Second, each face move cycles the three-sticker strips on the neighboring faces. Prime moves are implemented as three clockwise turns, and half turns are implemented as two clockwise turns. The class also defines equality and `Hash3d`, so complete cube states can be stored in `unordered_map` during BFS.

### `RubiksCube1dArray`

File: `Models/RubiksCube1dArray.cpp`

`RubiksCube1dArray` stores all 54 stickers in one flat `char cube[54]` array. It uses the mapping `face * 9 + row * 3 + col` to convert a face-row-column position into a single index. This keeps the same logical cube coordinates as the 3D model while storing the state in one contiguous block of memory.

The rotation workflow is the same as the 3D model, but every sticker access goes through `getIndex()`. This makes the representation compact and straightforward to compare or hash. `operator==` checks all 54 stickers, and `Hash1d` serializes the 54-character state into a string before applying `std::hash<string>`. It is a useful middle ground between readability and compact state storage.

### `RubiksCubeBitboard`

File: `Models/RubiksCubeBitboard.cpp`

`RubiksCubeBitboard` is the compact representation. It stores each face in a `uint64_t`, using eight 8-bit slots for the eight movable stickers around the fixed center. The center is not stored because each face center defines that face's identity. Ring positions are ordered clockwise, so rotating a face can be done by bit-shifting the 64-bit face value.

After rotating the face itself, the move method transfers the affected edge strips between adjacent faces using bit masks. `one_8` extracts or writes one sticker slot, while `one_24` handles three adjacent sticker slots. The model also implements solved-state checking, equality, assignment, `HashBitboard`, and `getCorners()`. The corner packing is important for Korf-style solving because it turns the cube's corner state into a compact value that can be indexed by the pattern database.

## Solvers

### `DFSSolver`

File: `Solvers/DFS_Solver.h`

`DFSSolver<T, H>` performs recursive depth-limited depth-first search. It is made for simple bounded searches where memory usage should stay low. The solver tries moves in enum order, applies a move, records it in the current path, and recurses to the next depth. If a branch fails, it removes the move from the path and applies the inverse move to restore the parent state.

DFS stores only the active recursion path, so it uses much less memory than BFS. The tradeoff is that it does not maintain a visited set, so it can revisit the same state through different move sequences. It also returns the first solution it finds within the depth limit, not necessarily the shortest solution.

### `BFSSolver`

File: `Solvers/BFS_Solver.h`

`BFSSolver<T, H>` performs breadth-first search over the cube-state graph. It is made for finding the shortest solution in an unweighted move graph. The solver starts from the scrambled cube, pushes it into a queue, and explores all states at depth 1 before depth 2, all states at depth 2 before depth 3, and so on.

BFS uses an `unordered_map` for visited states and another map to remember which move reached each state. When it finds the solved cube, it walks backward from the solved state to the original scramble using `move_done`, inverts each stored move to reconstruct the predecessor, then reverses the collected path to produce the forward solution. BFS gives optimal depth results for small scrambles, but memory grows quickly because full frontiers and visited states are stored.

### `IDDFSSolver`

File: `Solvers/IDDFS_Solver.h`

`IDDFSSolver<T, H>` performs iterative deepening depth-first search. It repeatedly runs DFS with increasing depth limits: first depth 1, then depth 2, then depth 3, continuing until the configured maximum depth. This gives DFS-like memory usage while still searching shallow solutions before deeper ones.

The intended workflow is to create a fresh depth-limited DFS solver for each limit and stop when the cube is solved. In the current source, the header refers to `DFS_Solver<T, H>`, while the DFS class is named `DFSSolver<T, H>`. That naming mismatch is why a direct syntax check of `IDDFS_Solver.h` fails until the symbol names are aligned.

### `IDAstarSolver`

File: `Solvers/IDAstar_Solver.h`

`IDAstarSolver<T, H>` implements an IDA*-style search using a corner pattern database as the heuristic. It loads the database from disk, computes an estimated remaining distance with `cornerDB.getNumMoves(cube)`, and searches using the priority value `depth + estimate`. If a state exceeds the current bound, the solver records the smallest exceeded value as the next bound.

The workflow is: load the corner database, start from an initial bound, expand states whose `g(n) + h(n)` fits within that bound, and increase the bound when no solution is found. When the solved state is found, the solver reconstructs the move sequence with the same backward `move_done` approach used in BFS. The heuristic is admissible in idea because solving the whole cube cannot require fewer moves than solving just the corners.

## Pattern Database System

### `NibbleArray`

Files: `PatternDatabases/NibbleArray.h`, `PatternDatabases/NibbleArray.cpp`

`NibbleArray` stores two 4-bit values inside each byte. It is used because the corner pattern database only needs small move depths, not full integers. A value like `0xF` marks an unset entry, while values such as `0` to `11` can store known minimum move counts. This cuts database memory roughly in half compared with storing one byte per entry.

### `PatternDatabase`

Files: `PatternDatabases/PatternDatabase.h`, `PatternDatabases/PatternDatabase.cpp`

`PatternDatabase` is the base class for table-based heuristics. It owns a `NibbleArray`, tracks the database size and number of filled entries, and provides methods to set a move count, retrieve a move count, write the database to a binary file, read it back from disk, inflate it into a byte vector, and reset it. Derived databases only need to define how a cube state maps to an integer database index.

### `CornerPatternDatabase`

Files: `PatternDatabases/CornerPatternDatabase.h`, `PatternDatabases/CornerPatternDatabase.cpp`

`CornerPatternDatabase` maps the cube's corner permutation and corner orientation into one integer index. It ranks the eight-corner permutation using `PermutationIndexer<8>`, converts seven corner orientations into a base-3 number, and combines them with:

```cpp
return (rank * 2187) + orientationNum;
```

The `2187` value is `3^7`, because the orientation of the eighth corner is determined by the first seven. The theoretical corner state count is `8! * 3^7 = 88,179,840`; the current source allocates `100,179,840` entries, and the generated database file size matches that current allocation.

### `PermutationIndexer`

File: `PatternDatabases/PermutationIndexer.h`

`PermutationIndexer<N, K>` computes a lexicographic rank for a permutation using a Lehmer-code style method. It precomputes factorial or pick values and a lookup table for the number of set bits. This lets the corner permutation become a compact integer rank, which is required before the corner pattern database can address an array entry.

### `math`

Files: `PatternDatabases/math.h`, `PatternDatabases/math.cpp`

These files provide small combinatorics helpers: `factorial`, `pick`, and `choose`. The permutation indexer uses `pick()` while building its ranking table.

### `CornerDBMaker`

Files: `PatternDatabases/CornerDBMaker.h`, `PatternDatabases/CornerDBMaker.cpp`

`CornerDBMaker` creates a corner pattern database by running BFS from the solved cube. Each visited cube contributes its corner database index and the depth at which it was first reached. Since BFS visits states in increasing depth order, the first depth stored for a corner state is the minimum depth discovered by that database-generation run. The current implementation stops when `curr_depth == 9`, so it creates a depth-limited corner database rather than a fully exhaustive corner database.

### `cornerDepth5V1.txt`

File: `Databases/cornerDepth5V1.txt`

This is the generated binary database file used by `IDAstarSolver`. It is not meant to be read as text. The file is loaded into `CornerPatternDatabase` and then queried during IDA* search to estimate how many moves remain for the cube's corner state.

## Documentation Files

`Planning.md` contains the early design notes: cube representation choices, why an abstract class is useful, how scrambling should work, and how the cube becomes a graph-search problem.

`Korfs_algo.md` explains the Korf/IDA* idea at a high level, including graph-search black boxes, heuristics, pattern databases, nibble storage, and the database-building workflow.

`docs/AAAI97-109.pdf` is the reference paper, *Finding Optimal Solutions to Rubik's Cube Using Pattern Databases*. GitHub Markdown does not reliably render embedded PDF viewers, so `Korfs_algo.md` links to the raw PDF directly.

## Demo Driver

File: `main.cpp`

`main.cpp` is a testing and experimentation driver. Most sections are commented blocks that were used to test individual parts of the project: model printing, move correctness, equality, hashing, DFS, BFS, IDDFS, IDA*, and corner database behavior. The active section creates a bitboard cube, shuffles it, loads the corner database file, runs `IDAstarSolver`, prints the solved cube, and prints the solution moves.

When compiling the active demo, two current source details matter:

- `IDDFS_Solver.h` uses `DFS_Solver<T, H>`, but the DFS class is named `DFSSolver<T, H>`.
- The database path in `main.cpp` is an absolute Windows path with backslashes, so it should be treated carefully when compiling or running on another machine.

## Build Notes

The repository includes a CMake file for the core library:

```powershell
cmake -S . -B build
cmake --build build
```

The current `CMakeLists.txt` builds `Models/RubiksCube.cpp` into the static library target `rubiks_cube_core` and exposes `Models/` as an include directory. The concrete model `.cpp` files, solver headers, pattern database files, and `main.cpp` are documented in this README because they are part of the project source tree, but they are not all wired into the current CMake target.

Useful syntax-check commands for individual source files are:

```powershell
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only Models\RubiksCube.cpp
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only Models\RubiksCube3dArray.cpp
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only Models\RubiksCube1dArray.cpp
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only Models\RubiksCubeBitboard.cpp
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only PatternDatabases\PatternDatabase.cpp
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only PatternDatabases\CornerPatternDatabase.cpp
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only PatternDatabases\CornerDBMaker.cpp
```

The repository's `.gitignore` excludes build output and local editor artifacts such as `build/`, executables, object files, and `.vscode/`.
