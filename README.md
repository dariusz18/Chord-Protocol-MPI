# Chord-Protocol-MPI

Implementation of the **CHORD** distributed hash table (DHT) protocol using **MPI** (Message Passing Interface) in C, developed as part of a Parallel and Distributed Algorithms course assignment.

## Overview

CHORD is a scalable peer-to-peer lookup protocol that uses consistent hashing to efficiently locate the node responsible for a given key. This implementation simulates a static CHORD ring across multiple MPI processes, where each process represents a node in the ring.

The identifier space has size `2^M = 16` (M = 4), and each node maintains a **finger table** that enables **O(log N)** key lookup routing.

## Features

- Static CHORD ring construction using sorted node IDs
- Finger table computation per node: `finger[i].start = (id + 2^i) mod 2^M`
- Logarithmic routing via `closest_preceding_finger()`
- Distributed lookup handling using `MPI_Send` / `MPI_Recv`
- Sequential lookup initiation to guarantee correct output ordering
- Deadlock-free service loop with `TAG_DONE` synchronization across all nodes

## Project Structure

```
.
├── tema2.c       # Main source file
├── Makefile      # Build configuration
├── in0.txt       # Input file for rank 0 (node ID + lookup keys)
├── in1.txt       # Input file for rank 1
└── ...           # One input file per MPI process
```

## Input Format

Each MPI process reads from its own input file `inX.txt` (where X is the MPI rank):

```
<chord_node_id>
<number_of_lookups>
<key_1>
<key_2>
...
```

## Building

```bash
make
```

Requires an MPI implementation (e.g., OpenMPI or MPICH) and a C compiler.

## Running

```bash
mpirun -np <num_processes> ./tema2
```

Make sure input files `in0.txt`, `in1.txt`, ..., `in(N-1).txt` exist before running.

## Output

For each lookup, the program prints the routing path taken through the CHORD ring:

```
Lookup <key>: <node1> -> <node2> -> ... -> <responsible_node>
```

## Implementation Details

### `build_finger_table()`
Computes the finger table for the current node. Each entry `i` has:
- `start = (self.id + 2^i) mod RING_SIZE`
- `node = find_successor_simple(start)` — searches only among existing nodes in `sorted_ids[]`

### `closest_preceding_finger(key)`
Iterates the finger table from `M-1` down to `0`, returning the largest finger node that falls strictly in the interval `(self.id, key)` on the circular ring. Returns `self.successor` as fallback. An explicit check `finger[i].node != key` prevents infinite routing loops.

### `handle_lookup_request()`
Processes an incoming lookup message:
1. Appends `self.id` to the routing path
2. If `key` falls in `(self.id, self.successor]`, appends the successor and sends `TAG_LOOKUP_REP` to the initiator
3. Otherwise, forwards `TAG_LOOKUP_REQ` to `closest_preceding_finger(key)`

### Service Loop
Each node runs a `while` loop receiving messages until it has collected `TAG_DONE` from all processes. Lookups are initiated sequentially (one at a time) to preserve output ordering without buffering results.

## Message Tags

| Tag | Value | Description |
|-----|-------|-------------|
| `TAG_LOOKUP_REQ` | 1 | Routing request forwarded through the ring |
| `TAG_LOOKUP_REP` | 2 | Lookup result sent back to the initiator |
| `TAG_DONE` | 3 | Signals that a node has finished all its lookups |

# Chord-Protocol-MPI
