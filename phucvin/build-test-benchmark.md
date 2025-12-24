# Build, Test and Benchmark Report

This document outlines the steps taken to build, test, and benchmark the Wren programming language, along with the results of the benchmarks.

## Steps

### 1. Build

The project was built using `make` with the `release_64bit` configuration.

```bash
make -C projects/make config=release_64bit
```

This generated the `wren`, `wren_shared`, and `wren_test` binaries in `bin/`.

### 2. Test

The tests were run using the provided python script `util/test.py`.

```bash
python3 util/test.py
```

All 866 tests passed (3861 expectations).

### 3. Benchmark

The benchmarks were run using the `util/benchmark.py` script.

```bash
python3 util/benchmark.py
```

## Benchmark Results

The following table shows the benchmark results. The time is in seconds (lower is better).

| Benchmark | Wren | Python | Comparison (Wren vs Python) |
| :--- | :--- | :--- | :--- |
| api_call | 0.08s | N/A | N/A |
| api_foreign_method | 0.40s | N/A | N/A |
| binary_trees | 0.32s | 0.42s | 133.56% (faster) |
| binary_trees_gc | 1.34s | N/A | N/A |
| delta_blue | 0.19s | 0.24s | 127.72% (faster) |
| fib | 0.31s | 0.38s | 123.68% (faster) |
| fibers | 0.07s | N/A | N/A |
| for | 0.11s | 0.27s | 240.41% (faster) |
| method_call | 0.15s | 0.29s | 193.74% (faster) |
| map_numeric | 1.72s | 1.35s | 78.90% (slower) |
| map_string | 0.17s | 0.17s | 102.84% (faster) |
| string_equals | 0.27s | 0.51s | 185.00% (faster) |

**Note:**
*   "N/A" indicates that the benchmark was not implemented or the interpreter was not found for that language.
*   The comparison percentage indicates how much faster (or slower) Wren is compared to Python. Values > 100% mean Wren is faster.
