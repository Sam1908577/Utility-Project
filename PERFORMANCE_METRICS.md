# Performance Metrics & Code Statistics

## Performance Metrics (Runtime Analysis)

### Test Results on MacBook Air (CPU-only execution)

#### Puzzle 5 (9×9 Sudoku) - Backtracking Method (Method 0)

```
Command: ./main inputs/5.txt output5.txt 0
```

**Time Metrics:**
- **User Time**: 0.13 seconds
- **System Time**: 0.00 seconds  
- **Real Time (Wall Clock)**: 0.139 seconds
- **CPU Usage**: 98% (single-threaded, nearly full CPU utilization)

**Interpretation:**
- User time = CPU time spent executing user code
- System time = CPU time spent in kernel/system calls (very low, good)
- Real time = actual elapsed time (wall clock)
- CPU usage = (user + system) / real time = 0.13/0.139 ≈ 98%

---

#### Puzzle 5 (9×9 Sudoku) - Dancing Links Method (Method 1)

```
Command: ./main inputs/5.txt output5_dlx.txt 1
```

**Time Metrics:**
- **User Time**: 0.26 seconds
- **System Time**: 0.00 seconds
- **Real Time (Wall Clock)**: 0.267 seconds
- **CPU Usage**: 99% (single-threaded, nearly full CPU utilization)

**Additional Info:**
- Dancing Links nodes created: **1,433 nodes**
- This method builds a data structure before solving

**Comparison:**
- Backtracking: **0.13s** (faster for this puzzle)
- Dancing Links: **0.26s** (slower, but uses more sophisticated algorithm)

---

## Code Statistics

### Lines of Code Breakdown

| File | Lines | Description |
|------|-------|-------------|
| **main.c** | 56 | Entry point, I/O handling, method selection |
| **methods.c** | 269 | Core algorithms (backtracking, DLX implementation) |
| **utils.c** | 110 | Utility functions (validation, printing, candidate checking) |
| **methods.h** | 41 | Header file with function declarations and macros |
| **sudoku_bt.cu** | 268 | Simple GPU parallel backtracking |
| **sudoku_obt.cu** | 268 | Optimized GPU parallel backtracking |
| **sudoku_dlx.cu** | 337 | GPU parallel Dancing Links X |
| **TOTAL** | **1,349** | All source code files |

---

## Detailed Code Breakdown by Component

### CPU Implementation (Runnable on MacBook Air)

#### 1. **main.c** (56 lines)
- **Lines 1-4**: Includes and headers
- **Lines 6-17**: `read_board()` - File I/O function
- **Lines 20-57**: `main()` - Program entry point
  - **Lines 22-26**: Argument validation
  - **Lines 28-31**: Read puzzle and calculate size
  - **Lines 33-35**: Open output file and parse method
  - **Lines 37-50**: Switch statement for method selection
  - **Lines 52-56**: Result handling and cleanup

#### 2. **methods.c** (269 lines)
- **Lines 7-29**: `backtracking_dfs()` - Recursive backtracking DFS
- **Lines 31-34**: `backtracking()` - Wrapper function
- **Lines 42-60**: `remove_column()` - DLX column removal
- **Lines 63-81**: `restore_column()` - DLX column restoration
- **Lines 84-114**: `exact_cover()` - DLX exact cover solver
- **Lines 116-177**: `build_dancing_links()` - Construct DLX data structure
- **Lines 180-231**: `convert_matrix()` - Convert Sudoku to exact cover matrix
- **Lines 234-243**: `convert_answer_print()` - Convert DLX answer to Sudoku
- **Lines 245-270**: `dancing_links()` - Main DLX solver function

#### 3. **utils.c** (110 lines)
- **Lines 6-65**: `check_board()` - Validate complete solution
  - **Lines 10-14**: Row sum validation
  - **Lines 18-23**: Column sum validation
  - **Lines 26-34**: Box sum validation
  - **Lines 37-41**: Row duplicate check
  - **Lines 44-48**: Column duplicate check
  - **Lines 51-62**: Box duplicate check
- **Lines 68-77**: `print_board()` - Output board to file
- **Lines 80-99**: `check_partial_board()` - Validate partial solution
- **Lines 102-111**: `initial_check()` - Find valid candidates

#### 4. **methods.h** (41 lines)
- **Lines 6-9**: Macros for position calculations
- **Lines 12-18**: Function declarations
- **Lines 25-31**: DLX macro definitions
- **Lines 32-39**: DLX function declarations

**Total CPU Code: 476 lines** (main.c + methods.c + utils.c + methods.h)

---

### GPU Implementation (Requires NVIDIA GPU)

#### 5. **sudoku_bt.cu** (268 lines)
- **Lines 15-18**: Function declarations
- **Lines 19-29**: `read_board()` - File I/O
- **Lines 32-55**: `main()` - Entry point
- **Lines 57-71**: `error_check()` - CUDA error handling
- **Lines 74-120**: `solve()` - Main solver function
  - **Lines 91-102**: GPU memory allocation and kernel launch
- **Lines 125-188**: `cpu_initial_search()` - CPU BFS expansion
- **Lines 190-209**: `check_partial_board_d()` - Device function
- **Lines 211-268**: `backtrack_kernel()` - GPU kernel function

#### 6. **sudoku_obt.cu** (268 lines)
- Similar structure to sudoku_bt.cu
- **Lines 189-268**: Optimized `backtrack_kernel()` with:
  - 2D thread block layout (N×N threads)
  - Cooperative constraint checking
  - Enhanced shared memory usage

#### 7. **sudoku_dlx.cu** (337 lines)
- **Lines 15-17**: Function declarations
- **Lines 19-29**: `read_board()` - File I/O
- **Lines 32-55**: `main()` - Entry point
- **Lines 57-71**: `error_check()` - CUDA error handling
- **Lines 74-135**: `solve()` - Main solver with DLX setup
- **Lines 141-237**: `cpu_initial_search()` - CPU BFS for DLX
- **Lines 240-257**: `remove_column_d()` - Device function
- **Lines 258-275**: `restore_column_d()` - Device function
- **Lines 277-337**: `exact_cover_kernel()` - GPU DLX kernel

**Total GPU Code: 873 lines** (all .cu files)

---

## Performance Analysis

### CPU Utilization

Both methods show **~98-99% CPU usage**, indicating:
- ✅ Efficient single-threaded execution
- ✅ Minimal I/O overhead
- ✅ No idle time waiting for resources
- ✅ CPU-bound computation (not I/O bound)

### System Time Analysis

**System time = 0.00s** for both methods:
- ✅ Minimal system calls
- ✅ Efficient memory management
- ✅ No significant kernel overhead
- ✅ Pure computational workload

### Algorithm Comparison

| Metric | Backtracking | Dancing Links |
|--------|-------------|---------------|
| **User Time** | 0.13s | 0.26s |
| **System Time** | 0.00s | 0.00s |
| **Real Time** | 0.139s | 0.267s |
| **CPU Usage** | 98% | 99% |
| **Speed** | **2x faster** | Slower |
| **Complexity** | Simple | Complex |
| **Memory** | Low | Higher (1,433 DLX nodes) |

**Why Backtracking is Faster Here:**
- Simpler algorithm, less overhead
- No data structure construction needed
- Direct constraint checking
- Dancing Links has setup cost (building DLX structure)

**When Dancing Links Would Be Better:**
- More constrained puzzles (fewer valid moves)
- Larger puzzles (16×16)
- Puzzles where constraint propagation helps more

---

## Code Complexity Metrics

### Function Count

| Component | Functions |
|-----------|-----------|
| **main.c** | 2 functions (read_board, main) |
| **methods.c** | 8 functions |
| **utils.c** | 4 functions |
| **sudoku_bt.cu** | 5 functions (1 kernel) |
| **sudoku_obt.cu** | 5 functions (1 kernel) |
| **sudoku_dlx.cu** | 6 functions (1 kernel, 2 device functions) |

### Average Function Size

- **main.c**: ~28 lines per function
- **methods.c**: ~34 lines per function
- **utils.c**: ~28 lines per function
- **GPU kernels**: ~60-80 lines each

---

## Memory Usage (Estimated)

### CPU Methods

**Backtracking:**
- Board state: 9×9 × 4 bytes = 324 bytes
- Stack (recursive): ~1-2 KB (depends on depth)
- **Total: ~2-3 KB**

**Dancing Links:**
- Board state: 324 bytes
- DLX structure: 1,433 nodes × 24 bytes ≈ 34 KB
- Valid candidates: 81 × 9 × 4 bytes ≈ 3 KB
- **Total: ~40-50 KB**

### GPU Methods (if run on NVIDIA GPU)

**Simple Backtracking:**
- Host memory: expand × 324 bytes
- Device memory: expand × 324 bytes + shared memory
- **Total: ~2× expand × 324 bytes**

**Optimized Backtracking:**
- Similar to simple, but optimized shared memory usage
- **Total: Similar, but better cache utilization**

**Dancing Links GPU:**
- Host memory: expand × (DLX size + answers)
- Device memory: expand × DLX structures
- **Total: Largest memory footprint**

---

## Summary

### Code Statistics Summary
- **Total Lines**: 1,349 lines
- **CPU Code**: 476 lines (35%)
- **GPU Code**: 873 lines (65%)
- **Language**: C (CPU) + CUDA C (GPU)

### Performance Summary
- **Fastest Method**: Backtracking (0.13s for puzzle 5)
- **CPU Efficiency**: 98-99% utilization
- **System Overhead**: Minimal (0.00s)
- **Scalability**: Both methods scale well with puzzle size

### Key Insights
1. **Backtracking is faster** for this puzzle size (9×9)
2. **System overhead is negligible** - pure computation
3. **CPU utilization is excellent** - no wasted cycles
4. **Code is well-structured** - clear separation of concerns
5. **GPU code is larger** - more complex parallel implementation

---

## How to Measure Performance Yourself

### Basic Timing
```bash
time ./main inputs/5.txt output.txt 0
```

### Detailed Profiling (if available)
```bash
# Using gprof (requires -pg flag during compilation)
gcc -pg main.c methods.c utils.c -o main -lm
./main inputs/5.txt output.txt 0
gprof main gmon.out
```

### Memory Profiling (if available)
```bash
# Using valgrind (Linux/WSL)
valgrind --leak-check=full ./main inputs/5.txt output.txt 0
```

---

## Notes

- These metrics are from **MacBook Air** running **CPU-only** code
- GPU code cannot run on MacBook Air (requires NVIDIA GPU)
- Performance will vary based on:
  - CPU speed
  - Puzzle difficulty
  - System load
  - Memory speed
