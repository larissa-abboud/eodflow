# API reference

Base URL: `http://localhost:8080/api`

_Replace the examples with real responses once each endpoint works._

| Method | Path | Description | Success |
|---|---|---|---|
| POST | `/runs` | Run the EOD for a business date | 201 |
| GET | `/runs/{id}` | Run status with each job's status and duration | 200 |
| GET | `/runs/{id}/report` | P&L per portfolio | 200 |

## Start a run

`POST /api/runs`

```json
{ "businessDate": "2026-10-13" }
```

`201 Created`

```json
{ "runId": 1, "status": "SUCCEEDED" }
```

## Get a run

`GET /api/runs/1`

```json
{
  "runId": 1,
  "businessDate": "2026-10-13",
  "status": "SUCCEEDED",
  "durationMs": 840,
  "jobs": [
    { "name": "load-trades",       "status": "SUCCEEDED", "durationMs": 210 },
    { "name": "load-market-data",  "status": "SUCCEEDED", "durationMs": 190 },
    { "name": "compute-positions", "status": "SUCCEEDED", "durationMs": 260 },
    { "name": "compute-pnl",       "status": "SUCCEEDED", "durationMs": 300 },
    { "name": "generate-report",   "status": "SUCCEEDED", "durationMs": 40 }
  ]
}
```

## Get the report

`GET /api/runs/1/report`

```json
{
  "businessDate": "2026-10-13",
  "portfolios": [
    { "portfolio": "EQUITY-EU", "pnl": 1520.40 },
    { "portfolio": "EQUITY-US", "pnl": -310.75 }
  ],
  "totalPnl": 1209.65
}
```

## Errors (ProblemDetail, RFC 9457)

| Status | When |
|---|---|
| 400 | Missing or invalid `businessDate` |
| 404 | Unknown run id |
