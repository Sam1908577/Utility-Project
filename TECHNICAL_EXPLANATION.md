# GPU Sudoku Solver - Comprehensive Technical Explanation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture & Design](#architecture--design)
3. [Algorithms Implemented](#algorithms-implemented)
4. [GPU Parallel Computing Architecture](#gpu-parallel-computing-architecture)
5. [CUDA Implementation Details](#cuda-implementation-details)
6. [Memory Hierarchy & Optimization](#memory-hierarchy--optimization)
7. [Parallel Execution Model](#parallel-execution-model)
8. [Performance Analysis](#performance-analysis)

---

## Project Overview

This project implements a **GPU-accelerated Sudoku solver** using NVIDIA CUDA (Compute Unified Device Architecture) for parallel computation. The solver supports multiple Sudoku grid sizes (4×4, 9×9, and 16×16) and implements five distinct algorithmic approaches:

1. **Sequential Brute Force Backtracking** (CPU-based)
2. **Sequential Dancing Links X (DLX)** (CPU-based)
3. **Simple Parallel Brute Force Backtracking** (GPU-accelerated)
4. **Optimized Parallel Brute Force Backtracking** (GPU-accelerated with optimizations)
5. **Parallel Dancing Links X** (GPU-accelerated DLX)

The project demonstrates **heterogeneous computing** principles, leveraging both CPU and GPU resources through a **hybrid CPU-GPU execution model**.

---

## Architecture & Design

### System Architecture

The project follows a **two-phase hybrid architecture**:

1. **CPU Preprocessing Phase**: 
   - Initial search tree expansion using **breadth-first search (BFS)**
   - Generates multiple partial solutions (search nodes) for parallel processing
   - Prepares data structures for GPU transfer

2. **GPU Parallel Execution Phase**:
   - **Massively parallel** exploration of search space
   - Each GPU thread processes a distinct search node
   - Early termination when solution is found

### Code Organization

- **`main.c`**: Sequential CPU implementations (backtracking, DLX)
- **`methods.c`**: Core algorithm implementations (validation, board checking)
- **`utils.c`**: Utility functions (board validation, printing)
- **`sudoku_bt.cu`**: Simple parallel backtracking CUDA kernel
- **`sudoku_obt.cu`**: Optimized parallel backtracking with shared memory
- **`sudoku_dlx.cu`**: Parallel Dancing Links X implementation

---

## Algorithms Implemented

### 1. Brute Force Backtracking (Sequential)

**Algorithm**: Depth-First Search (DFS) with constraint checking

**Time Complexity**: O(b^d) where b = branching factor, d = depth

**Key Characteristics**:
- **Recursive backtracking** through search tree
- **Constraint propagation** at each node
- **Pruning** invalid branches early

**Implementation Details**:
```c
backtracking_dfs(board, n, position, fp):
    if position == N*N:
        return solution_found
    for each candidate value:
        if valid_move(board, position, value):
            board[position] = value
            if backtracking_dfs(board, n, position+1, fp):
                return solution_found
            board[position] = 0  // backtrack
    return no_solution
```

### 2. Dancing Links X (DLX) Algorithm

**Algorithm**: Exact Cover Problem solver using **doubly-linked circular lists**

**Key Data Structure**: 
- **Dancing Links**: Circular doubly-linked list structure
- **Column headers**: Represent constraints
- **Row nodes**: Represent candidate placements

**Advantages**:
- Efficient constraint satisfaction
- Fast column removal/restoration (O(1) operations)
- Reduces search space through constraint propagation

**Constraint Matrix Representation**:
- **4×N² constraints** per Sudoku:
  - N² cell constraints (each cell must have exactly one number)
  - N² row constraints (each row must contain each number once)
  - N² column constraints (each column must contain each number once)
  - N² box constraints (each box must contain each number once)

### 3. Parallel Brute Force Backtracking

**Parallelization Strategy**: **Tree-level parallelism**

- Each GPU thread explores a different branch of the search tree
- **Work distribution**: One search node per thread
- **Early termination**: Atomic operations signal solution discovery

### 4. Optimized Parallel Backtracking

**Optimizations**:
- **Shared memory utilization** for faster data access
- **Cooperative thread blocks** for constraint checking
- **Reduced global memory access** through data locality

### 5. Parallel Dancing Links X

**Parallelization**: Multiple DLX structures processed simultaneously
- Each thread maintains its own dancing links structure
- Parallel exact cover solving across search space

---

## GPU Parallel Computing Architecture

### CUDA Compute Architecture

**Compute Capability**: Requires NVIDIA GPU with **compute capability 3.5+** (Kepler architecture or newer)

**Key GPU Components**:

1. **Streaming Multiprocessors (SMs)**:
   - Each SM contains multiple **CUDA cores** (scalar processors)
   - Execute instructions in **SIMT** (Single Instruction, Multiple Thread) fashion
   - Support **warp-level parallelism** (32 threads execute together)

2. **Memory Hierarchy**:
   - **Global Memory**: Large, high-latency (hundreds of cycles)
   - **Shared Memory**: Fast on-chip memory (tens of cycles), shared within thread block
   - **Registers**: Fastest, per-thread storage
   - **Constant Memory**: Cached read-only memory
   - **Texture Memory**: Cached read-only memory with spatial locality

3. **Thread Hierarchy**:
   - **Grid**: Collection of thread blocks
   - **Block**: Collection of threads (up to 1024 threads)
   - **Warp**: 32 threads executing in lockstep
   - **Thread**: Individual execution unit

### CUDA Execution Model

**Kernel Launch Configuration**:
```cuda
kernel<<<grid_dim, block_dim, shared_mem_size>>>(parameters)
```

**Example from `sudoku_bt.cu`**:
```cuda
dim3 grid_dim(expand / tile_size);      // Number of thread blocks
dim3 block_dim(tile_size);               // Threads per block
backtrack_kernel<<<grid_dim, block_dim, shared_mem>>>(...)
```

**Execution Flow**:
1. **Host (CPU)** prepares data structures
2. **Host-to-Device transfer** via `cudaMemcpy`
3. **Kernel launch** spawns parallel threads
4. **Device execution** on GPU SMs
5. **Device-to-Host transfer** of results
6. **Host** processes final solution

---

## CUDA Implementation Details

### 1. Simple Parallel Backtracking (`sudoku_bt.cu`)

**Kernel Function**: `backtrack_kernel`

**Thread Mapping**:
- Each thread processes one search node (partial board state)
- `task_id = blockIdx.x * blockDim.x + threadIdx.x`

**Memory Usage**:
- **Shared Memory**: Stores board state and backtracking stack
  - `shared_board[threadIdx.x * N*N]`: Board state per thread
  - `shared_board[blockDim.x * N*N + threadIdx.x]`: Stack per thread
- **Global Memory**: 
  - `ans_all[task_id * N*N]`: All board states
  - `ans_found`: Atomic flag for early termination

**Key Features**:
- **Stack-based backtracking** (non-recursive)
- **Atomic Compare-And-Swap (CAS)** for solution detection:
  ```cuda
  atomicCAS(ans_found, -1, task_id)
  ```
- **Early termination**: All threads check `*ans_found` flag

**Device Function**: `check_partial_board_d`
- GPU version of constraint checking
- Validates row, column, and box constraints

### 2. Optimized Parallel Backtracking (`sudoku_obt.cu`)

**Optimizations**:

1. **2D Thread Block Layout**:
   ```cuda
   dim3 block_dim(N, N);  // N×N threads per block
   ```
   - Maps threads to board positions
   - Enables **cooperative constraint checking**

2. **Shared Memory Communication**:
   - Thread (0,0) coordinates backtracking decisions
   - Other threads participate in constraint validation
   - Uses `__syncthreads()` for synchronization

3. **Parallel Constraint Checking**:
   ```cuda
   if (ROW(nowp, N) == threadIdx.x || 
       COL(nowp, N) == threadIdx.y || 
       BOX(nowp, n) == box_now) {
       // Mark invalid numbers
   }
   ```
   - Each thread checks its assigned row/column/box
   - Reduces sequential constraint checking overhead

4. **Reduced Memory Footprint**:
   - Stores only necessary board state
   - Optimized shared memory allocation

### 3. Parallel Dancing Links X (`sudoku_dlx.cu`)

**Data Structure Transfer**:
- Dancing links structure copied to device memory
- Each thread maintains independent DLX state

**Kernel Function**: `exact_cover_kernel`

**Parallelization Strategy**:
- Multiple DLX search trees explored simultaneously
- Each thread executes independent exact cover algorithm
- Stack-based backtracking (non-recursive)

**Device Functions**:
- `remove_column_d`: Removes column from dancing links
- `restore_column_d`: Restores column (backtracking)

**Memory Management**:
- `dlxs_d`: Array of dancing links structures (one per thread)
- `dlx_props_d`: Column/row properties (shared across threads)
- `ans_d`: Answer arrays per thread

---

## Memory Hierarchy & Optimization

### Memory Access Patterns

1. **Coalesced Memory Access**:
   - Threads access consecutive memory locations
   - Enables **memory coalescing** (single transaction for multiple accesses)
   - Critical for performance

2. **Shared Memory Optimization**:
   - **Bank conflicts**: Avoided through careful indexing
   - **Synchronization**: `__syncthreads()` ensures data consistency
   - **Lifetime**: Shared memory persists for block lifetime

3. **Global Memory Minimization**:
   - Frequent access to global memory is expensive
   - Shared memory used as **scratchpad** for frequently accessed data
   - Data loaded once, reused multiple times

### Memory Transfer Optimization

**Host-Device Transfer**:
```cuda
cudaMemcpy(device_ptr, host_ptr, size, cudaMemcpyHostToDevice)
```
- **Pinned memory** (not explicitly used here) could improve transfer speed
- **Asynchronous transfers** could overlap computation and transfer

**Device Memory Allocation**:
```cuda
cudaMalloc((void**)&ptr, size)
cudaMemset(ptr, value, size)
```

### Cache Utilization

- **L1 Cache**: Per-SM cache for global memory access
- **L2 Cache**: Shared across all SMs
- **Texture Cache**: Optimized for 2D spatial locality
- **Constant Cache**: For read-only data

---

## Parallel Execution Model

### SIMT (Single Instruction, Multiple Thread)

**Warp Execution**:
- 32 threads execute same instruction simultaneously
- **Divergence**: If threads take different branches, execution serializes
- Minimizing divergence improves performance

**Example of Divergence**:
```cuda
if (threadIdx.x < 16) {
    // Path A
} else {
    // Path B
}
// Warp splits: 16 threads execute A, 16 execute B sequentially
```

### Thread Block Scheduling

**Block Assignment to SMs**:
- GPU scheduler assigns blocks to available SMs
- **Occupancy**: Ratio of active warps to maximum warps per SM
- Higher occupancy → better latency hiding

**Occupancy Factors**:
- **Register usage**: Limits number of concurrent blocks
- **Shared memory**: Limits block count per SM
- **Thread block size**: Affects warp count

### Synchronization Primitives

1. **`__syncthreads()`**: 
   - Synchronizes all threads within a block
   - Ensures memory consistency

2. **Atomic Operations**:
   - `atomicCAS`: Compare-and-swap (used for early termination)
   - Ensures thread-safe updates to shared variables

3. **Memory Fences**:
   - `__threadfence()`: Ensures memory writes visible to other threads
   - `__threadfence_block()`: Block-level memory fence

---

## Performance Analysis

### Experimental Results

Based on the provided benchmarks:

| Puzzle | BF (seq) | DLX (seq) | SPBF | OPBF | PDLX |
|--------|----------|-----------|------|------|------|
| 3      | 0.011s   | **0.010s**| 0.284s | 0.292s | 0.288s |
| 4      | 0.357s   | **0.057s**| 0.400s | 0.358s | 0.531s |
| 5      | **0.383s**| 0.493s   | 0.596s | 0.505s | 0.930s |
| 6      | 1.114s   | 0.815s   | 1.813s | **0.668s** | 1.532s |
| 7      | **0.160s**| 0.589s   | 3.295s | 1.084s | 1.685s |
| 8      | 0.495s   | 1.539s   | 1.049s | **0.422s** | 5.213s |
| 9      | ~300s    | ~75s     | 1.683s | **1.445s** | 1.892s |

### Performance Characteristics

**GPU Overhead**:
- **Small puzzles**: GPU overhead dominates (transfer + kernel launch)
- **Large puzzles**: Parallelism benefits overcome overhead

**Scalability**:
- Performance improves with problem size
- More search nodes → better GPU utilization

**Algorithm Efficiency**:
- **DLX**: Efficient for constraint-heavy puzzles
- **Backtracking**: Better for puzzles with fewer constraints
- **GPU acceleration**: Most beneficial for large search spaces

### Bottlenecks & Optimization Opportunities

1. **Memory Transfer Overhead**:
   - Host-Device transfers are expensive
   - Could use **unified memory** (CUDA 6.0+) or **pinned memory**

2. **Load Balancing**:
   - Some threads finish early (solution found)
   - **Dynamic parallelism** could redistribute work

3. **Occupancy**:
   - Limited by register/shared memory usage
   - Could optimize kernel to increase occupancy

4. **Divergence**:
   - Conditional branches cause warp divergence
   - Restructure algorithms to minimize divergence

---

## Technical Terminology Glossary

### GPU Architecture Terms

- **CUDA Core**: Scalar processor within an SM
- **Streaming Multiprocessor (SM)**: Processing unit containing multiple CUDA cores
- **Warp**: Group of 32 threads executing in lockstep
- **Thread Block**: Collection of threads that can cooperate via shared memory
- **Grid**: Collection of thread blocks executing the same kernel
- **SIMT**: Single Instruction, Multiple Thread execution model

### Memory Terms

- **Global Memory**: High-capacity, high-latency device memory
- **Shared Memory**: Fast on-chip memory shared within a thread block
- **Register**: Fastest memory, private to each thread
- **Coalescing**: Combining multiple memory accesses into fewer transactions
- **Bank Conflict**: Multiple threads accessing same shared memory bank simultaneously

### Parallel Computing Terms

- **Parallelism**: Simultaneous execution of multiple tasks
- **Concurrency**: Managing multiple tasks (may not execute simultaneously)
- **Occupancy**: Ratio of active warps to maximum warps per SM
- **Divergence**: Threads in a warp taking different execution paths
- **Latency Hiding**: Using parallelism to hide memory access latency

### CUDA-Specific Terms

- **Kernel**: Function executed on GPU
- **Host**: CPU and its memory
- **Device**: GPU and its memory
- **Device Function**: Function callable from device code
- **Host Function**: Function callable from host code
- **Atomic Operation**: Thread-safe operation on shared memory

### Algorithm Terms

- **Backtracking**: Systematic search with constraint checking and pruning
- **Constraint Propagation**: Eliminating invalid candidates based on constraints
- **Exact Cover Problem**: Finding subsets that cover all elements exactly once
- **Dancing Links**: Data structure for efficient exact cover solving
- **Search Tree**: Tree representation of solution space
- **Pruning**: Eliminating branches that cannot lead to solutions

---

## Compilation & Execution

### Build Requirements

- **CUDA Toolkit**: Version 9.1 or compatible
- **NVIDIA GPU**: Compute capability 3.5+ (Kepler, Maxwell, Pascal, Volta, Turing, Ampere, Ada, Hopper)
- **Compiler**: NVCC (NVIDIA CUDA Compiler)

### Compilation Process

```bash
sh compile.sh
```

**Generated Executables**:
1. `main`: Sequential CPU solvers
2. `sudoku_bt`: Simple parallel backtracking
3. `sudoku_obt`: Optimized parallel backtracking
4. `sudoku_dlx`: Parallel Dancing Links X

### Execution Examples

```bash
# Sequential backtracking
./main input.txt output.txt 0

# Sequential DLX
./main input.txt output.txt 1

# Simple parallel backtracking
./sudoku_bt input.txt output.txt <expand> <block_size>

# Optimized parallel backtracking
./sudoku_obt input.txt output.txt <expand>

# Parallel DLX
./sudoku_dlx input.txt output.txt <expand> <block_size>
```

**Parameters**:
- `expand`: Number of search nodes to generate (parallelism level)
- `block_size`: Threads per CUDA block (affects occupancy)

---

## Conclusion

This project demonstrates **heterogeneous parallel computing** principles through GPU-accelerated Sudoku solving. Key achievements:

1. **Multiple algorithmic approaches** with different parallelization strategies
2. **Efficient GPU memory utilization** through shared memory optimization
3. **Scalable parallel execution** across thousands of threads
4. **Hybrid CPU-GPU architecture** for optimal performance

The implementation showcases fundamental GPU programming concepts including **CUDA kernels**, **memory hierarchy management**, **thread synchronization**, and **parallel algorithm design**.
