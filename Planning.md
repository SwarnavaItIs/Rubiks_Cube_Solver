## 1. Representation of the RubiksCube in code

- a. How to take an input
- b. 18 Basic Operations
- c. Print

## 1.B. is the cude solvable?

## 2. Solving it :

- a.outputting the (min?) steps required to solve it.

## 3. Storing a Cube:

- a. 3D matrix - 3 * 3 * 6 - storing 6 face colors
- b. 1D array - just linearize the 3D array

## 4. Why do we need an Abstract Class????

- a. we write a solver for a generic class and it can solve for all the children classes.
- b. To maintain consistency across samples

```text
Solver 1 - bfs solver, Solver 2, Solver 3
   |----------------------|---------|
              |
        Abstract Class
              |
   -----------------------------------------
   |                   |                   |
Model 1 - 3D array, Model 2 - 1D array, Model 3 - bit-board
```

## 5. How do we get a shuffled Rubiks-Cube?

- a. Take 3*3*6 colors and place randomly - doesnot work since the resulting cube might not be solvable
- b. Start from a solved cube, and do some random operations - its always solvable.

## 6. Do we need to implement the random shuffle method in the abstract class, or derived class?

- a. If abstract class, then how?
  We can just use the already declared 18 rotation (ops).

## 7. Solvers------------------------------

How Graph???? 1. State is a Node, 2. Move is an edge

Note -----> A rubiks cube can be solved in atmost 20 steps (Always, irrespective of the orientation)

BFS takes more memory since it maintains a queue of the current level nodes.

### A. DFS Solver -

---How can i even maintain a visited map of an object belonging to a class?

To maintain an unordered_map, we need to maintain the visited:

1. we give a key value pair
2. it hashes the key
3. stores the value on the hashed key pair

To maintain an unordered_map, the things we need are -

1. Hash Function - that takes in the object and maps it to the value
2. operator== : we need to overload this operator
