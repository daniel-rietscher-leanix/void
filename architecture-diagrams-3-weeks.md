# Architecture Diagrams: 3-Week Work Summary

This document captures the architectural changes made over a 3-week period through before/after diagrams for each significant feature.

---

## 1. KPI Time Series Feature (Jan 28-30)

Shows the new API endpoint and data flow introduced.

### Before

```
┌─────────────┐     ┌─────────────────┐     ┌─────────┐
│   Client    │────▶│  KpiDataService │────▶│  Cache  │
└─────────────┘     │  (values only)  │     └─────────┘
                    └─────────────────┘
```

### After

```
┌─────────────┐     ┌─────────────────┐     ┌─────────┐
│   Client    │────▶│  KpiDataService │────▶│  Cache  │
└─────────────┘     ├─────────────────┤     └─────────┘
                    │ getKPIValues()  │           │
      GET           │ getKPITimeseries│◀──────────┘
   /timeseries ────▶│  (NEW)         │
                    └─────────────────┘
                            │
                    ┌───────▼───────┐
                    │ Date Sampling │
                    │  Utilities    │
                    └───────────────┘
```

---

## 2. KpiDataService Decomposition (Feb 11-12)

The most significant architectural change - monolith to focused services.

### Before

```
┌────────────────────────────────────────────────────┐
│              KpiDataService (Monolith)             │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │ • getValues()                                │  │
│  │ • getTimeseries()                            │  │
│  │ • getPreview()                               │  │
│  │ • getDrilldown()                             │  │
│  │ • getDetails()                               │  │
│  │ • cache logic                                │  │
│  │ • error handling                             │  │
│  │ • 500+ lines of mixed concerns               │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

### After

```
┌────────────────────────────────────────────────────────────────┐
│                    KpiDataService (Facade)                      │
│              Delegates to specialized services                  │
└───────┬──────────┬──────────┬──────────┬──────────┬───────────┘
        │          │          │          │          │
        ▼          ▼          ▼          ▼          ▼
┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐
│  Values   │ │Timeseries │ │  Preview  │ │ Drilldown │ │  Detail   │
│  Service  │ │  Service  │ │  Service  │ │  Service  │ │  Service  │
├───────────┤ ├───────────┤ ├───────────┤ ├───────────┤ ├───────────┤
│getValues()│ │getTimes.. │ │getPreview │ │getDrill.. │ │getDetails │
│bulk fetch │ │date sample│ │computation│ │data slice │ │KPI detail │
└───────────┘ └───────────┘ └───────────┘ └───────────┘ └───────────┘
```

---

## 3. Cache-First Optimization (Feb 12-16)

Request flow pattern change across all 4 KPI endpoints.

### Before (Job-First)

```
┌────────┐    Request     ┌─────────────┐
│ Client │───────────────▶│   Backend   │
└────────┘                └──────┬──────┘
    ▲                            │
    │                    ┌───────▼───────┐
    │                    │ Always Create │
    │                    │     Job       │
    │                    └───────┬───────┘
    │                            │
    │                    ┌───────▼───────┐
    │    {status:        │  Return       │
    │◀────PENDING,       │  {PENDING}    │
    │     location}      │  + Location   │
    │                    └───────┬───────┘
    │                            │
    │                            ▼
    │                    ┌───────────────┐
    │                    │    Worker     │
    │                    │   Process     │
    │                    └───────┬───────┘
    │                            │
    │              ┌─────────────┴─────────────┐
    │              │                           │
    │        [HIT] ▼                     [MISS]▼
    │    ┌─────────────┐              ┌─────────────┐
    │    │ Read from   │              │  Compute    │
    │    │   Cache     │              │  (DuckDB)   │
    │    └──────┬──────┘              └──────┬──────┘
    │           │                            │
    │           └────────────┬───────────────┘
    │                        ▼
    │         Poll     ┌───────────┐
    └──────────────────│  Return   │
       Location        │  Result   │
                       └───────────┘

        (Job always created, cache checked by worker)
```

### After (Cache-First)

```
┌────────┐    Request     ┌─────────────┐
│ Client │───────────────▶│   Backend   │
└────────┘                └──────┬──────┘
    ▲                            │
    │                    ┌───────▼───────┐
    │                    │  Check Cache  │
    │                    └───────┬───────┘
    │                            │
    │              ┌─────────────┴─────────────┐
    │              │                           │
    │        [HIT] ▼                     [MISS]▼
    │    ┌─────────────┐              ┌─────────────┐
    │    │   Return    │              │ Queue Job   │
    │    │ {status:    │              │ Return      │
    │◀───│ COMPLETED,  │              │ {status:    │◀─┐
    │    │  result}    │              │  PENDING}   │  │
    │    └─────────────┘              │ + Location  │  │
    │                                 └──────┬──────┘  │
    │                                        │         │
    │                                        ▼         │
    │                                 ┌───────────┐    │
    │         Poll Location           │  Worker   │    │
    └─────────────────────────────────│  Process  │────┘
                                      │  (async)  │
                                      └─────┬─────┘
                                            │
                                            ▼
                                      ┌───────────┐
                                      │  DuckDB   │
                                      │ (compute) │
                                      └───────────┘
```

---

## 4. Frontend Integration (Feb 4-11)

RxJS operator pattern for handling the new cache-first backend.

### Before

```
┌─────────────────────────────────────────────────────────────┐
│                    Angular Service                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   loadKpiValues(): Observable<KpiValue[]> {                 │
│     return this.http.get('/api/kpi/values')                 │
│       .pipe(                                                │
│         map(response => response.data)                      │
│       );                                                    │
│   }                                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
              Simple synchronous response
```

### After

```
┌─────────────────────────────────────────────────────────────┐
│                    Angular Service                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   loadKpiValues(): Observable<KpiValue[]> {                 │
│     return this.http.get('/api/kpi/values')                 │
│       .pipe(                                                │
│         getJobResult(),      // Parse response type         │
│         handleJobResult({    // Route by status             │
│           onImmediate: ...,  // COMPLETED → use directly    │
│           onPending: ...     // PENDING → poll location     │
│         })                                                  │
│       );                                                    │
│   }                                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  New RxJS Utilities                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  getJobResult()                                             │
│  ├── Parses HTTP response                                   │
│  └── Returns: ImmediateResult | PendingResult               │
│                                                             │
│  handleJobResult(config)                                    │
│  ├── ImmediateResult → config.onImmediate()                 │
│  └── PendingResult   → config.onPending() → poll → result  │
│                                                             │
└─────────────────────────────────────────────────────────────┘

                    Request Flow:
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │   HTTP Call ──▶ getJobResult() ──▶ handleJobResult() │
    │                                          │           │
    │                         ┌────────────────┴────────┐  │
    │                         ▼                         ▼  │
    │                   [COMPLETED]               [PENDING]│
    │                        │                         │   │
    │                        ▼                         ▼   │
    │                   Use Result              Poll Loop  │
    │                   Immediately              Until     │
    │                                           Complete   │
    │                                              │       │
    │                                              ▼       │
    │                                         Use Result   │
    └──────────────────────────────────────────────────────┘
```

---

## Notes

- **KPI Panel Loading Stability** work was skipped as it involved primarily bug fixes and stabilization without significant architectural changes
- All diagrams use ASCII format for universal rendering compatibility
- Each section shows a clear before/after state to document the architectural evolution
