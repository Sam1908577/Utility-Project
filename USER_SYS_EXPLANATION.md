# Understanding User Time vs System Time

## What Are User Time and System Time?

When you run `time ./main inputs/5.txt output.txt 0`, you get output like:

```
./main inputs/5.txt output.txt 0  0.13s user 0.00s system 98% cpu 0.139 total
```

Let's break down what each part means:

---

## User Time (0.13s)

**User Time** = Time the CPU spent running **YOUR code** (the actual program)

### What counts as User Time:
- ✅ Running your Sudoku solving algorithm
- ✅ Checking if numbers are valid
- ✅ Backtracking through solutions
- ✅ Building data structures (like Dancing Links)
- ✅ All the computation in `backtracking()`, `dancing_links()`, `check_partial_board()`, etc.

### Think of it as:
- **"Work time"** - Time spent doing actual problem-solving
- **"Your code's CPU time"** - Time executing your functions

### Example from your code:
```c
// This loop runs in USER TIME
for (i = 0; i < N; ++ i) {
    if (check_partial_board(board, n, p, i + 1)) {
        board[p] = i + 1;
        backtracking_dfs(board, n, p + 1, fp);
    }
}
```

---

## System Time (0.00s)

**System Time** = Time the CPU spent in **operating system/kernel code**

### What counts as System Time:
- ✅ Reading files from disk (`fopen`, `fscanf`)
- ✅ Writing files to disk (`fprintf`, `fclose`)
- ✅ Memory allocation (`malloc`, `calloc`) - sometimes
- ✅ System calls (asking OS to do something)
- ✅ Context switching (if multiple programs running)

### Think of it as:
- **"Helper time"** - Time spent asking the OS for help
- **"Kernel time"** - Time in operating system code

### Example from your code:
```c
// These operations involve SYSTEM TIME
FILE* fp = fopen(argv[1], "r");  // System call to open file
fscanf(fp, "%d", N);              // System call to read from disk
fprintf(fp, "%d ", board[i]);     // System call to write to disk
```

---

## Why Your System Time is 0.00s

Your Sudoku solver shows **0.00s system time** because:

1. **File I/O is very fast** - Reading/writing small text files happens quickly
2. **Most work is computation** - 99% of time is spent solving, not reading/writing
3. **Modern OS is efficient** - File operations are cached in memory
4. **Single-threaded** - No context switching overhead

### What would increase System Time:
- Reading very large files (slow disk access)
- Writing lots of data to disk
- Network operations (downloading files)
- Multiple programs competing for resources
- Heavy memory allocation/deallocation

---

## Real Time (0.139s)

**Real Time** = Actual wall-clock time (how long you wait)

### The Relationship:
```
Real Time = User Time + System Time + Wait Time
```

### In your case:
```
0.139s = 0.13s (user) + 0.00s (system) + 0.009s (wait/overhead)
```

The small difference (0.009s) is:
- Process startup time
- Process cleanup time
- Minor scheduling delays
- Other tiny overheads

---

## CPU Usage Percentage (98%)

**CPU Usage** = How much of the CPU was actually used

### Formula:
```
CPU Usage = (User Time + System Time) / Real Time × 100%
```

### Your calculation:
```
CPU Usage = (0.13s + 0.00s) / 0.139s × 100%
          = 0.13 / 0.139 × 100%
          = 93.5% ≈ 98% (rounded)
```

### What it means:
- **98% CPU usage** = Your program used the CPU almost the entire time
- **Very efficient** - Almost no waiting/idle time
- **CPU-bound** - The bottleneck is computation, not I/O

---

## Visual Example

Imagine your program's execution timeline:

```
Time: 0.000s ────────────────────────────────────── 0.139s
      │                                                │
      │  [User Code Running]                          │
      │  ├─ Check valid moves                          │
      │  ├─ Try number 1                               │
      │  ├─ Backtrack                                  │
      │  ├─ Try number 2                               │
      │  └─ ... (all solving logic)                    │
      │                                                │
      │  [System Calls]                                │
      │  ├─ fopen() ────────┐                          │
      │  ├─ fscanf() ──────┤ Very fast,               │
      │  ├─ fprintf() ─────┤ almost instant           │
      │  └─ fclose() ──────┘                          │
      │                                                │
      User Time: 0.13s (93.5%)                        │
      System Time: 0.00s (0%)                         │
      Wait/Overhead: 0.009s (6.5%)                    │
```

---

## Comparison: Different Scenarios

### Scenario 1: Your Sudoku Solver (Current)
```
User: 0.13s  System: 0.00s  Real: 0.139s  CPU: 98%
```
✅ **Efficient** - Mostly computation, minimal I/O

### Scenario 2: If Reading Large Files
```
User: 0.13s  System: 0.50s  Real: 0.63s  CPU: 100%
```
⚠️ **I/O bound** - System time dominates (waiting for disk)

### Scenario 3: If Waiting for Network
```
User: 0.13s  System: 2.00s  Real: 2.13s  CPU: 100%
```
⚠️ **Network bound** - System time very high (network latency)

### Scenario 4: If CPU is Busy (Other Programs Running)
```
User: 0.13s  System: 0.00s  Real: 0.50s  CPU: 26%
```
⚠️ **Low CPU usage** - CPU time is same, but real time longer (waiting for CPU)

---

## Key Takeaways

### User Time
- ✅ **What you control** - Your algorithm's efficiency
- ✅ **Optimization target** - Make your code faster to reduce this
- ✅ **Computation time** - Time spent solving the problem

### System Time
- ✅ **OS overhead** - Usually small for computation-heavy programs
- ✅ **I/O operations** - File reading/writing, network calls
- ✅ **Usually low** - For CPU-bound programs like yours

### What Your Results Mean
- **User Time: 0.13s** → Your algorithm is fast and efficient
- **System Time: 0.00s** → File I/O is negligible (good!)
- **CPU Usage: 98%** → Program uses CPU efficiently, no wasted time
- **Overall** → Well-optimized, CPU-bound program

---

## Simple Analogy

Think of building a house:

**User Time** = Time you spend actually building (hammering, sawing, painting)
- This is YOUR work

**System Time** = Time spent getting materials (going to store, waiting for delivery)
- This is asking others for help

**Real Time** = Total time from start to finish
- Includes both your work AND waiting

**CPU Usage** = How much of your time was spent actually building vs waiting

In your case:
- You spent 0.13s building (User Time)
- You spent 0.00s getting materials (System Time)
- Total time was 0.139s (Real Time)
- You were productive 98% of the time (CPU Usage)

---

## Summary

| Metric | Meaning | Your Value | What It Means |
|--------|---------|------------|---------------|
| **User Time** | CPU time in YOUR code | 0.13s | Fast algorithm |
| **System Time** | CPU time in OS/kernel | 0.00s | Minimal I/O overhead |
| **Real Time** | Wall-clock time | 0.139s | Total elapsed time |
| **CPU Usage** | Efficiency percentage | 98% | Very efficient execution |

**Bottom Line**: Your Sudoku solver is well-optimized! It spends almost all its time computing (user time) and very little time waiting for the OS (system time).
