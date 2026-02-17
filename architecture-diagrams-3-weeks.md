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

## 2. Panel Orchestration / Global Batching (Jan 30 - Feb 4)

Introduced global batching across panels - requests grouped by timeframe, not by panel.

### Before (Per-Panel Requests)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Dashboard                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐               │
│  │  Panel A    │   │  Panel B    │   │  Panel C    │               │
│  │ KPI-1,KPI-2 │   │ KPI-3,KPI-4 │   │ KPI-5,KPI-6 │               │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘               │
│         │                 │                 │                       │
│         ▼                 ▼                 ▼                       │
│    ┌─────────┐       ┌─────────┐       ┌─────────┐                 │
│    │ POST    │       │ POST    │       │ POST    │                 │
│    │ /job    │       │ /job    │       │ /job    │                 │
│    └────┬────┘       └────┬────┘       └────┬────┘                 │
│         │                 │                 │                       │
│         ▼                 ▼                 ▼                       │
│      Poll #1           Poll #2           Poll #3                    │
│         │                 │                 │                       │
└─────────┴─────────────────┴─────────────────┴───────────────────────┘

        Result: 3 panels = 3 separate API calls + 3 polling loops
```

### After (Global Batching)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Dashboard                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐               │
│  │  Panel A    │   │  Panel B    │   │  Panel C    │               │
│  │ KPI-1,KPI-2 │   │ KPI-3,KPI-4 │   │ KPI-5,KPI-6 │               │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘               │
│         │                 │                 │                       │
│         └────────────┬────┴────┬────────────┘                       │
│                      ▼         │                                    │
│              ┌───────────────┐ │                                    │
│              │  Batch Queue  │◀┘                                    │
│              │ (by timeframe)│                                      │
│              └───────┬───────┘                                      │
│                      │ 500ms debounce                               │
│                      ▼                                              │
│              ┌───────────────┐                                      │
│              │  Single POST  │                                      │
│              │  /bulk/job    │                                      │
│              │ [all 6 KPIs]  │                                      │
│              └───────┬───────┘                                      │
│                      │                                              │
│                      ▼                                              │
│              ┌───────────────┐                                      │
│              │  Single Poll  │                                      │
│              └───────┬───────┘                                      │
│                      │                                              │
│              ┌───────▼───────┐                                      │
│              │  Distribute   │                                      │
│              │   Results     │                                      │
│              └───────┬───────┘                                      │
│         ┌────────────┼────────────┐                                 │
│         ▼            ▼            ▼                                 │
│     Panel A      Panel B      Panel C                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

        Result: 3 panels = 1 API call + 1 polling loop
```

### Key Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                      System Components                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  KpiValueService (orchestrator)                                     │
│  ├── isLegacyKpi === true  → LegacyKpiLoaderService (direct GET)   │
│  └── isLegacyKpi === false → CustomKpiBatchLoaderService (batched) │
│                                                                     │
│  CustomKpiBatchLoaderService                                        │
│  ├── BatchKeyManager      (creates keys from timestamps)            │
│  ├── RequestVersionTracker (conflict resolution on timeframe change)│
│  ├── PanelReferenceCounter (tracks panel interest for cancellation) │
│  └── BatchMapHelper       (distributes results to panels)           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. KpiDataService Decomposition (Feb 11-12)

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

## 4. Cache-First Optimization (Feb 12-16)

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

## 5. Frontend Integration (Feb 4-11)

RxJS operator pattern for handling the new cache-first backend.

### Before (Job-First Polling)

```
┌─────────────────────────────────────────────────────────────┐
│                    Angular Service                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   loadKpiValues(): Observable<KpiValue[]> {                 │
│     return this.http.post('/api/kpi/values/job')  // Create │
│       .pipe(                                                │
│         map(response => response.jobId),                    │
│         switchMap(jobId => this.pollJobStatus(jobId))       │
│       );                                                    │
│   }                                                         │
│                                                             │
│   pollJobStatus(jobId): Observable<Result> {                │
│     return interval(1000).pipe(                             │
│       switchMap(() => this.http.get(`/api/job/${jobId}`)),  │
│       takeWhile(res => res.status !== 'COMPLETED', true),   │
│       filter(res => res.status === 'COMPLETED'),            │
│       map(res => res.result)                                │
│     );                                                      │
│   }                                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘

                    Request Flow:
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │   POST /job ──▶ {jobId} ──▶ Poll GET /job/{id}      │
    │                                    │                 │
    │                                    ▼                 │
    │                             [PENDING...]             │
    │                                    │                 │
    │                                    ▼                 │
    │                             [COMPLETED]              │
    │                                    │                 │
    │                                    ▼                 │
    │                              Use Result              │
    │                                                      │
    │         (Always creates job, always polls)           │
    └──────────────────────────────────────────────────────┘
```

### After (Cache-First)

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
