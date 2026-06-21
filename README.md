# Rubik's Cube Project

This project models a 3x3 Rubik's Cube in C++ and treats cube solving as a graph-search problem. A cube configuration is a graph node, and each legal face rotation is an edge leading to another valid configuration. The project separates common cube behavior from storage, allowing the same solver design to operate on different representations.

## Cube Conventions

| Face | Color |
| --- | --- |
| Up | White |
| Left | Green |
| Front | Red |
| Right | Blue |
| Back | Orange |
| Down | Yellow |

Each face supports a clockwise quarter-turn, a counterclockwise quarter-turn marked with `'`, and a half-turn marked with `2`. Across all six faces, these form the 18 moves `L`, `L'`, `L2`, `R`, `R'`, `R2`, `U`, `U'`, `U2`, `D`, `D'`, `D2`, `F`, `F'`, `F2`, `B`, `B'`, and `B2`.

## Architecture and Workflow

`RubiksCube` defines the behavior shared by every representation. Concrete models decide how stickers are stored and rotated, while solvers work with a model type and, when required, its matching hash function. This keeps the search algorithms independent of whether the cube uses a multidimensional array, a flat array, or bit-packed storage.

1. Construct a solved concrete cube model.
2. Apply moves directly or call `randomShuffleCube()` to produce a reachable, solvable scramble.
3. Pass the scrambled model to a solver.
4. Let the solver explore states through the 18 legal moves.
5. Receive a `vector<RubiksCube::MOVE>` and convert its values to notation with `RubiksCube::getMove()`.

## Models

### `RubiksCube` Base Model

Files: `Models/RubiksCube.h` and `Models/RubiksCube.cpp`

`RubiksCube` is the abstract contract for every representation. It defines face, color, and move enumerations, then requires concrete models to implement color lookup, solved-state detection, and every face rotation. The shared `move()` dispatcher maps a `MOVE` value to the corresponding operation, while `invert()` applies its inverse. Because all representations expose this interface, the solvers can use one consistent set of 18 operations.

The base model also handles representation-independent workflows. `print()` obtains colors through the virtual `getColor()` method and renders a planar cube net. `randomShuffleCube()` applies randomly selected legal rotations to the current valid state, ensuring the scramble remains reachable. The corner helpers collect a corner's three colors, encode its identity as a compact index, and determine orientation from the location of its white or yellow sticker.

### `RubiksCube3dArray`

File: `Models/RubiksCube3dArray.cpp`

`RubiksCube3dArray` is made for a direct geometric representation. It stores stickers in `cube[6][3][3]`, so each sticker is addressed by face, row, and column as it appears in the physical layout. The constructor fills every face with its solved color, `getColor()` translates stored characters into the shared color enum, and `isSolved()` checks that all nine positions on each face match that face's expected color.

Each move has two stages. `rotateFace()` first copies a 3x3 face and writes its rows and columns back in clockwise order. The move method then cycles the three-sticker strips on the four adjacent faces, reversing strip direction where the geometry requires it. Counterclockwise moves reuse three clockwise turns, and half-turns reuse two. Equality compares all 54 positions, while `Hash3d` flattens those positions into a string and hashes it so complete cube states can be used as `unordered_map` keys.

### `RubiksCube1dArray`

File: `Models/RubiksCube1dArray.cpp`

`RubiksCube1dArray` is made for a contiguous representation of the same 54 stickers. It stores the cube in `cube[54]` and maps `(face, row, column)` to `face * 9 + row * 3 + column`. The model preserves the coordinate-based interface while keeping all stickers in one linear block, making state copying, comparison, and serialization straightforward.

Its rotation algorithm mirrors the 3D model but routes every access through `getIndex()`. A face rotation copies nine values into temporary storage and writes them back in rotated positions, after which the move cycles adjacent strips through flattened indices. Prime and half-turn moves reuse the clockwise operation. Solved-state detection checks each face segment against its fixed color, and `Hash1d` hashes the 54-character state string for visited-state containers.

### `RubiksCubeBitboard`

File: `Models/RubiksCubeBitboard.cpp`

`RubiksCubeBitboard` is made for a compact bit-packed representation. Its design assigns one 64-bit value to each face and represents the eight movable stickers around a fixed center with eight-bit color values. With ring positions stored clockwise, rotating the face value by 16 bits shifts every sticker by two ring positions, corresponding to a 90-degree face rotation. A complete move also transfers the relevant strips between adjacent face values.

The current source is a scaffold containing `solved_side_config[6]`. It does not define the abstract base operations, equality, hashing, or completed move logic, and it currently fails a standalone syntax check because the class definition has no terminating semicolon. This description reflects the implementation currently present rather than presenting the bitboard model as complete.

## Solvers

### `BFSSolver`

File: `Solvers/BFS_Solver.h`

`BFSSolver<T, H>` performs breadth-first search over the cube-state graph and is made for finding a minimum-move solution in an unweighted search space. `T` is the concrete cube model and `H` is its hash function. The solver queues the scrambled cube, marks it visited, then repeatedly removes the oldest state, checks whether it is solved, and generates up to 18 neighbors by applying every legal move.

During expansion, one move is applied to a local node. An unseen result is marked, associated with the move that reached it, and added to the queue; the inverse move then restores the parent before the next branch is generated. Once a solved state is found, `move_done` is followed backward: each stored move is collected and inverted to recover the predecessor until the original scramble is reached. Reversing that list produces the forward solution. BFS guarantees the shallowest solution, while its queue and visited maps require substantial memory.

### `DFSSolver`

File: `Solvers/DFS_Solver.h`

`DFSSolver<T, H>` performs recursive depth-limited depth-first search and is made for searches with a fixed maximum depth and lower auxiliary memory usage. It checks whether the current cube is solved, stops a branch after `max_search_depth`, and otherwise tries each of the 18 moves. Every chosen move is applied to the current cube and appended to the active solution path before recursion continues.

When a branch fails, the solver removes its last move and applies the inverse operation, restoring the exact parent state before exploring another branch. A successful branch returns without backtracking, preserving both the solution path and solved cube. The algorithm stores only its recursion path, but it has no visited-state set, can revisit configurations, and returns the first solution encountered within the limit rather than guaranteeing the shortest one. Its default maximum depth is 8.

### `IDDFSSolver`

File: `Solvers/IDDFS_Solver.h`

`IDDFSSolver<T, H>` performs iterative deepening depth-first search and is made to combine DFS's bounded memory use with a preference for shallow solutions. It launches a fresh depth-limited DFS with limits `1, 2, 3, ...` through `max_search_depth`, always beginning from the same scrambled cube. When one run finishes solved, IDDFS stores that solver's cube and returns its moves.

Since every smaller limit is exhausted before the next limit begins, the first successful iteration finds a solution at the shallowest reachable depth under the move model. The repeated searches use more computation in exchange for avoiding BFS's full frontier. The current header instantiates `DFS_Solver<T, H>`, while the included class is named `DFSSolver<T, H>`; this naming mismatch causes the IDDFS header's current syntax check to fail.

## State Identity and Hashing

BFS requires stable state identity. The 3D and 1D models therefore provide `operator==` and matching hash functors. Equality checks every sticker, while each hash functor serializes stickers in a fixed order and applies `std::hash<string>`. This pair lets an `unordered_map` use an entire cube as a key without exposing representation details to the solver.

DFS does not use a visited-state container, even though its template accepts a hash type for a consistent solver signature. It relies on correct inverse operations to restore state during backtracking.

## Project Structure

```text
Rubiks Cube Project/
|-- CMakeLists.txt
|-- Planning.md
|-- README.md
|-- Models/
|   |-- RubiksCube.h
|   |-- RubiksCube.cpp
|   |-- RubiksCube3dArray.cpp
|   |-- RubiksCube1dArray.cpp
|   `-- RubiksCubeBitboard.cpp
`-- Solvers/
    |-- BFS_Solver.h
    |-- DFS_Solver.h
    `-- IDDFS_Solver.h
```

## Build

The current CMake configuration builds `Models/RubiksCube.cpp` as the static library target `rubiks_cube_core` and exposes `Models` as a public include path:

```powershell
cmake -S . -B build
cmake --build build
```

The core, completed concrete models, and standalone solver headers can be syntax-checked with `g++`:

```powershell
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only Models\RubiksCube.cpp
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only Models\RubiksCube3dArray.cpp
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only Models\RubiksCube1dArray.cpp
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only -x c++ Solvers\BFS_Solver.h
g++ -std=c++17 -Wall -Wextra -pedantic -fsyntax-only -x c++ Solvers\DFS_Solver.h
```

The CMake target currently compiles only the shared base implementation; the concrete models and solver headers are documented above according to their present source implementations.
