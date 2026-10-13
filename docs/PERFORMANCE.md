# Timing test: sequential vs parallel

_Fill in during Phase 6. Real numbers only._

## Setup

| Item | Value |
|---|---|
| Machine | CPU model, number of cores, RAM |
| Java | 21 |
| Dataset | N trades, N portfolios, N instruments, N days of prices |
| Method | average of 5 runs, after 1 warm-up run |

## Results

| Runner | Thread pool size | Average EOD run time | Speed-up |
|---|---|---|---|
| Sequential | – | | 1.0× |
| Parallel | 1 | | |
| Parallel | 2 | | |
| Parallel | 4 | | |

## Observations

- Which jobs ran in parallel:
- Why the speed-up is not exactly ×N (hint: the longest chain of dependent jobs cannot be parallelised):
- What I would try next:
