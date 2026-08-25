Update the existing Google Analytics module in the website to add an **AEC SaaS Benchmark** comparison to the existing analytics metrics.

### Objective

For each supported Google Analytics metric, display:

* Current period value from GA4
* Previous period value
* WoW change
* AEC SaaS benchmark
* Performance status compared with the benchmark

Do NOT change or remove the existing Google Analytics functionality.

### Benchmark Configuration

Create a centralized benchmark configuration so benchmark values are NOT hardcoded inside UI components.

Use the following initial benchmark values:

```text
Avg Engagement Time:
  benchmark_min: 45 seconds
  benchmark_max: 90 seconds
  label: "45–90s (SaaS goal)"

Views/User:
  benchmark_min: 2  
  benchmark_max: 4
  label: "2–4 (SaaS typical)"

Demo Page Views:
  benchmark_type: "signal"
  label: "High intent signal"

Users:
  benchmark_type: "none"
  label: "—"
```

Keep this configuration easy to modify or extend later.

### UI

Update the existing analytics comparison table to include the column:

**AEC SaaS Benchmark**

Example:

| Metric              | Aug 17–23 | Aug 10–16 |    WoW Δ | AEC SaaS Benchmark |
| ------------------- | --------: | --------: | -------: | ------------------ |
| Users               |        69 |        41 | ▲ +68.3% | —                  |
| Avg Engagement Time |       26s |       21s | ▲ +23.8% | 45–90s (SaaS goal) |
| Views/User          |      1.65 |         — |        — | 2–4 (SaaS typical) |
| Demo Page Views     |         5 |         — |        — | High intent signal |

### Benchmark Comparison Logic

For numeric range benchmarks:

* If current value < benchmark_min:
  status = "Below benchmark"
* If current value >= benchmark_min AND current value <= benchmark_max:
  status = "Within benchmark"
* If current value > benchmark_max:
  status = "Above benchmark"

Example:

```text
Avg Engagement Time = 26s
Benchmark = 45–90s
Result = Below benchmark
```

```text
Views/User = 2.5
Benchmark = 2–4
Result = Within benchmark
```

Do not apply numeric comparison logic to metrics such as Demo Page Views where the benchmark is qualitative.

### Status Display

Add a small visual status indicator next to the benchmark or below the metric where appropriate:

* Below benchmark
* Within benchmark
* Above benchmark
* No benchmark

Use the existing design system and styling of the analytics module. Do not introduce a completely different visual style.

### Data Source

Continue using the existing Google Analytics 4 Data API integration for the actual metric values.

Do NOT attempt to fetch the custom AEC SaaS benchmark values from the Google Analytics API.

The architecture should be:

GA4 Data API
→ Actual analytics metrics
→ Benchmark configuration
→ Comparison logic
→ UI

### Date Range

The benchmark comparison must always use the **current period value** selected by the existing analytics date-range/filter system.

Do not compare the benchmark against the previous period value.

For example:

```text
Current period:
Aug 17–23
Avg Engagement Time = 26s

Previous period:
Aug 10–16
Avg Engagement Time = 21s

Benchmark:
45–90s

Comparison:
26s vs 45–90s
Status:
Below benchmark
```

### Important Requirements

1. Preserve the existing GA4 API integration.
2. Preserve the existing date-range functionality.
3. Preserve the existing WoW calculation.
4. Do not hardcode benchmark values directly inside React/UI components.
5. Put benchmark values in one centralized configuration/service.
6. Make the benchmark configuration easy to update later.
7. Handle missing/null GA4 values gracefully.
8. Handle metrics that do not have a benchmark.
9. Do not display misleading benchmark comparisons when the required data is unavailable.
10. Keep the implementation responsive and consistent with the existing UI.
11. Do not expose Google Analytics credentials, OAuth secrets, refresh tokens, or API credentials to the frontend.
12. Perform benchmark configuration and comparison logic in a reusable way so additional metrics and industries can be added later.

### Future Extensibility

Structure the benchmark configuration so that additional industries can be added later, for example:

```text
AEC SaaS
Construction SaaS
Architecture SaaS
Engineering SaaS
General SaaS
```

The current implementation should use:

```text
AEC SaaS
```

as the active benchmark profile.

Also make it possible to add additional benchmark metrics later without modifying the main analytics table component.

Finally, test the implementation with the existing GA4 data and verify that the displayed benchmark and status values are calculated correctly.