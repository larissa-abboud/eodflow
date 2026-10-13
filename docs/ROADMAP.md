# EODFlow – Step-by-step plan (focused version)

About **10 days** at 2–4 hours a day. Each phase ends with something that **works**, is **tested** and is **pushed**.

**Golden rules**
1. **Write the code yourself.** In the interview you'll be asked "why did you do it this way?" Use tutorials and AI to *understand*, not to paste.
2. **Commit small and often** with clear messages: `feat: detect cycles in job graph`, `test: P&L for a short position`.
3. **Read [FINANCE.md](FINANCE.md) first.** You must be able to explain position, mark-to-market and P&L in plain words.
4. **Finished beats ambitious.** Only start the optional extras once phases 0–6 are done.

---

## Phase 0 – Set up (today, ~1–2 h)

1. Go to **https://start.spring.io**:
   - Project **Maven** · Language **Java** · Spring Boot: latest stable **4.x** (not a SNAPSHOT or milestone version)
   - Group `com.larissaabboud` · Artifact `eodflow` · Java **21**
   - Dependencies: **Spring Web**, **Validation** (only these two; no database in this version)
2. Unzip, open in IntelliJ, run `./mvnw verify`, and check that it passes (green).
3. Create the **public** GitHub repo named exactly **`eodflow`**.
4. Copy this starter pack into the project folder: `README.md`, `docs/`, `.github/`, `.gitignore` (merge with the generated one).
5. `git init`, commit, push. On GitHub, open the **Actions** tab and check that the CI run is green. The badge in the README turns green too.
6. Create the packages (empty for now):
   ```
   com.larissaabboud.eodflow
   ├── domain     ← finance (Phase 1)
   ├── workflow   ← engine (Phases 2–3)
   ├── eod        ← the EOD jobs (Phase 4)
   └── api        ← REST (Phase 5)
   ```

✅ **Done when:** repo public, CI green, README shows the diagrams.

---

## Phase 1 – Finance domain in plain Java (days 2–3)

Package `domain`. No Spring annotations here.

1. `Side` enum: `BUY, SELL`.
2. `Trade` record: `id, portfolio, instrument, side, quantity (long), price (BigDecimal), tradeDate (LocalDate)`.
3. `MarketPrice` record: `instrument, date, closePrice (BigDecimal)`.
4. `PositionCalculator`: takes `List<Trade>` and returns the net quantity per instrument, as `Map<String, Long>`. BUY adds, SELL subtracts.
   - Tip: `trades.stream().collect(Collectors.groupingBy(Trade::instrument, Collectors.summingLong(...)))`
5. `ValuationCalculator`: market value = quantity × close price.
6. `PnlCalculator`: daily P&L per instrument = `position × (today's close − yesterday's close)`. Total = sum.
7. Always round the same way: `setScale(2, RoundingMode.HALF_EVEN)`.
8. **Unit tests** (JUnit 5). Write the expected value worked out by hand in a comment:
   - buy 100, buy 50, sell 30 → position 120
   - long position, price goes up → positive P&L
   - short position (−50), price goes up → negative P&L
   - no trades → P&L is 0
   - 2 instruments → total P&L = sum of both

✅ **Done when:** about 10 green tests, pushed, CI green.

---

## Phase 2 – Workflow engine: the job graph (days 4–5)

Package `workflow`. Generic: no finance in here.

1. The job contract:
   ```java
   public interface Job {
       String name();
       void execute(JobContext context) throws Exception;
   }
   ```
2. `JobContext`: wraps a `ConcurrentHashMap<String, Object>` so jobs can pass results to each other (`put("positions", ...)`, `get("positions")`).
3. `JobGraph`:
   - `addJob(Job job, String... dependsOn)`
   - `validate()`: throws if a dependency name doesn't exist, or if there is a **cycle**
   - `topologicalOrder()`: returns a valid order using **Kahn's algorithm** (count incoming edges, repeatedly take jobs with 0 remaining). If not every job comes out, there's a cycle.
4. `SequentialRunner`: runs jobs in topological order and records, for each job, its `status` and `durationMs` in a `JobResult` record.
5. `JobStatus` enum: `PENDING, RUNNING, SUCCEEDED, FAILED, SKIPPED`.
6. **Tests** (use small fake jobs that just record their name in a list):
   - chain A → B → C runs in that order
   - "diamond": A → B, A → C, B → D, C → D. D always runs last
   - cycle A → B → A is detected
   - a dependency on a job that doesn't exist is rejected

✅ **Done when:** engine runs fake jobs in the right order, tests green.
💬 **Be ready to explain:** what a DAG is, why cycles are a problem, how Kahn's algorithm works.

---

## Phase 3 – Run independent jobs in parallel (days 6–7)

This is the most important phase for Murex, because it shows real Java concurrency.

1. `ParallelRunner` using `Executors.newFixedThreadPool(n)`:
   - Keep, for each job, how many dependencies are still unfinished: `Map<String, AtomicInteger> remaining`.
   - Submit every job with 0 remaining.
   - When a job **succeeds**, decrement its children; submit each child whose counter reaches 0.
   - Wait until all jobs are finished (e.g. a `CountDownLatch` set to the number of jobs), then shut down the pool.
2. **Failure handling:** if a job throws, mark it `FAILED` and mark **all jobs that depend on it (directly or indirectly) `SKIPPED`**. Jobs in other branches keep running.
3. Return a `RunResult`: overall status + list of `JobResult` (name, status, durationMs, error message).
4. **Tests:**
   - two independent jobs that each `Thread.sleep(500)` finish in **well under 1000 ms** in total, which proves parallelism
   - diamond graph: D still runs after both B and C
   - B throws → D is SKIPPED, C still SUCCEEDED
   - run the diamond test **100 times in a loop** to catch race conditions

✅ **Done when:** all tests green, many times in a row.
💬 **Be ready to explain:** why `AtomicInteger` and `ConcurrentHashMap` (thread safety), what a thread pool is, why you shut it down.

---

## Phase 4 – The real EOD jobs (day 8)

Package `eod`. Connect the finance code (Phase 1) to the engine (Phases 2–3).

1. `SampleDataGenerator`: creates fake data in memory:
   - N trades across 3 portfolios and 10 instruments
   - 30 days of closing prices per instrument using a random walk: `nextPrice = price × (1 + random between −2% and +2%)`
   - Use a fixed `Random(42)` seed so results are reproducible
2. Five jobs, each a small class implementing `Job`:

   | Job | Depends on | Does |
   |---|---|---|
   | `load-trades` | – | puts the trades in the context |
   | `load-market-data` | – | puts prices in the context |
   | `compute-positions` | load-trades | positions per portfolio |
   | `compute-pnl` | compute-positions, load-market-data | P&L per portfolio |
   | `generate-report` | compute-pnl | builds a summary (portfolio → total P&L) |

3. `EodService.run(businessDate)`: builds the graph, runs it with `ParallelRunner`, and keeps the `RunResult` and the report in an **in-memory store** (`ConcurrentHashMap<Long, RunResult>` with an `AtomicLong` id).
4. One test that runs the full EOD on a small dataset and checks every job SUCCEEDED and the P&L is correct.

✅ **Done when:** one method call runs the full EOD and produces a report.

---

## Phase 5 – REST API (day 9)

Package `api`. Now Spring comes in.

1. `EodService` becomes a `@Service`; the generator and runner become beans.
2. `RunController`:
   - `POST /api/runs` with `{"businessDate": "2026-10-13"}` runs the EOD and returns `201` with the run id and status
   - `GET /api/runs/{id}` returns the run: overall status + each job's status and duration
   - `GET /api/runs/{id}/report` returns P&L per portfolio
3. Request body as a Java `record` with `@NotNull` on the date.
4. `@RestControllerAdvice` returning `404` for an unknown run id and `400` for a bad request.
5. Tests with `@WebMvcTest` + MockMvc: POST returns 201, GET of an unknown id returns 404.
   - Spring Boot 4 tip: test support is split into smaller modules. If `@WebMvcTest` isn't found, check that the web test starter generated by start.spring.io is in `pom.xml`, and let IntelliJ fix the import. Most Spring Boot 3 tutorials still work, but some package names changed.
6. Try it with curl or Postman, and paste a real response into `docs/API.md`.

✅ **Done when:** you can start a run and read its result over HTTP.

---

## Phase 6 – Timing test + polish (day 10)

This gives you a real number for your CV.

1. Generate a big dataset, e.g. **500,000 trades**.
2. To make the parallel effect visible, also add a few extra independent jobs (e.g. P&L per portfolio as separate jobs, or a `compute-exposure` job next to `compute-pnl`).
3. Time the full run with `SequentialRunner` vs `ParallelRunner` (pool sizes 1, 2, 4). Run each 5 times and take the average.
4. Fill in `docs/PERFORMANCE.md` with the table and your machine (CPU cores, RAM).
5. Polish:
   - README: tick the roadmap, add the "Results" numbers, add an example curl output
   - Check that every public class has a clear name and short Javadoc where useful
   - Delete unused code; CI green

🎯 **Then update the CV** with your real number, e.g. *"Parallel execution reduced the EOD run time from X s to Y s on 500k trades."*

---

## Optional extras (only after Phase 6)

Pick at most one or two, in this order of value for Murex:

1. **Retries:** a job can be retried up to N times before FAILED.
2. **Docker:** a `Dockerfile` so the app runs with `docker run`.
3. **Actuator:** `/actuator/health` and a timer per job.
4. **Historical VaR:** see FINANCE.md.
5. **PostgreSQL:** store runs instead of keeping them in memory.

---

## After each phase

- [ ] Tests green locally and in CI
- [ ] Pushed with clear commit messages
- [ ] README roadmap ticked
- [ ] Two lines of notes: **what I built** and **one problem I hit and how I solved it**. These become your interview answers.
