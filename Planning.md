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

## We are making a header file .h and not writing all the code in a .cpp file since when we need to import the .cpp file, it gets copied fully into the new file where we wanna import it. hence importing a header file is much easier and convinient.

## Cube Representations ----
1. 3D Cube - cube[face/6][row/3][col/3] - cube[i][r][c] - color of ith face, ith row, jth col
** centers of the cube are static hence fix them -
Up → 0 (White)
Left → 1 (Green)
Front → 2 (Red)
Right → 3 (Blue)
Back → 4 (Orange)
Down → 5 (Yellow)

2. 1D Cube Representation - 6 * 3 * 3 = 54 length 1D array
3. BitBoard Representation -
White 	00000001
Green 	00000010
Red 	00000100
Blue 	00001000
Orange 	00010000
Yellow 	00100000

Color	White	Green	Red	Blue	Orange	Yellow
Face	Up	Left	Front	Right	Back	Down

8 (stickers) * 8 (8-bit color) = 64 bits

----------------------------------
the order is this:- 
0    1    2

7    _    3

6    5    4
Keeping indices of the cube in a clockwise rotational fashion

This idea helps us greatly in rotating a particular face. While doing any operation this idea makes it very easy to do a face rotation. We just need to rotate the 64-bit integer by 16 bits and the face would be rotated. Next, we can adjust the adjacent sides of the face manually.
-----------------------------------

64 bit format looks something like this - 

Color	G	        W	        Y	        O	        B	        R	        G	        W
Bits	00000010 	00000001 	00100000 	00010000 	00001000 	00000100 	00000010	00000001 
Index	7	        6	        5	        4	        3	        2	        1	        0

bitboard[ i ] → the 64-bit integer representing the ith face