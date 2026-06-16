# Rubiks Cube Project

A C++ Rubik's Cube model project.

## Structure

```text
Rubiks Cube Project/
├── CMakeLists.txt
├── Models/
│   ├── RubiksCube.cpp
│   └── RubiksCube.h
└── README.md
```

## Build

This project currently builds the shared Rubik's Cube model as a static library.

```powershell
cmake -S . -B build
cmake --build build
```

For a quick compile check without CMake:

```powershell
g++ -std=c++17 -Wall -Wextra -pedantic -c Models\RubiksCube.cpp -o build\RubiksCube.o
```

## Next Steps

- Add concrete cube implementations that inherit from `RubiksCube`.
- Add solver modules once the cube representation is ready.
- Add a small `main.cpp` or test target to exercise moves and shuffles.
