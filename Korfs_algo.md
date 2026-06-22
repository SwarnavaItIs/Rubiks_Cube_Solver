# Korf's Algorithm Notes

## Reference Paper

[Open *Finding Optimal Solutions to Rubik's Cube Using Pattern Databases*](./docs/AAAI97-109.pdf)

<object data='./docs/AAAI97-109.pdf' type='application/pdf' width='100%' height='800'>
  <p>PDF preview unavailable. <a href='./docs/AAAI97-109.pdf'>Open the paper directly.</a></p>
</object>

## Graph Search as a Black Box

Every graph algorithm can be described as having a black box:

1. Put all neighbors of a particular root node into the black box.
2. Take one node out of the black box.
3. Put all neighbors of that node into the black box.
4. Continue until the goal is found or no states remain.

The data structure and priority rule used by the black box determine the search algorithm:

| Algorithm | Black box and priority |
| --- | --- |
| BFS | Queue |
| DFS | Stack |
| Dijkstra's algorithm | Priority queue ordered by distance traveled so far |
| A* (Korf's algorithm) | Priority queue ordered by distance traveled plus the heuristic |

## Heuristic

A heuristic is a guess of how far the goal node is from the current node. The guess must be equal to or less than the actual distance, making it an underestimate.

```text
f(n) = g(n) + h(n)
```

- `g(n)` is the distance traveled to reach node `n`.
- `h(n)` is the estimated distance from node `n` to the goal.
- `f(n)` is pushed into the priority queue.

## Corner Pattern Database

The corner pattern database stores a mapping:

```text
Rubik's Cube -> corner configuration -> moves required to solve the corners
```

It works as a useful guess because it is an underestimate. A Rubik's Cube cannot be solved in fewer moves than the number of moves required to solve its corners.

## Why Store Corner Configurations?

We store corner configurations instead of complete Rubik's Cube configurations because there are quintillions of full-cube configurations, while the number of corner configurations is manageable.

## Combining Heuristics

Suppose one database returns `x` and another returns `y` as heuristic values. It is better to use:

```text
max(x, y)
```

## IDA*

IDA* is A* combined with IDDFS.

## Corner Database Storage

The corner pattern database stores the minimum number of moves, `n`, required to solve the corners, where `n <= 11`.

- The database contains values from `0` to `11`.
- An 8-bit integer can store each value directly.
- A 4-bit integer does not exist as a standard C++ type, but it can be implemented with a nibble array.
- A nibble array uses half the memory of a byte array.

The number of corner states is:

```text
8! * 3^7 = 88,179,840 states
```

- At one byte per state, the database needs approximately 84 MiB.
- At four bits per state, the database needs approximately 42 MiB.

## Database File Workflow

1. Create a file containing all pattern-database values.
2. Load the complete file into memory when the solver starts.
3. Store the loaded values in an array.
4. Use the corner configuration's array index to retrieve its heuristic value.

## Building the Database File

0. Initialize an array.
1. Start BFS from the solved state as the root node.
2. Find the solved state's index and assign it depth `0`.
3. Explore every unexplored neighbor.
4. Assign each newly discovered state its BFS depth.
5. After BFS covers all 88,179,840 corner states, write the array to disk.
