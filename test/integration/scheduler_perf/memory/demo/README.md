# QHint inFlightPods Memory Leak Demo

## Overview

This demo demonstrates how the memory tracking feature in scheduler_perf can detect memory leaks in the kube-scheduler. We simulate a memory leak in the QHint's inFlightPods map by preventing proper cleanup of scheduled pods.

## Memory Leak Implementation

The memory leak is implemented in `pkg/scheduler/backend/queue/active_queue.go` simulating https://github.com/kubernetes/kubernetes/issues/120622:

```go
// Memory leak simulation for demo purposes
// When SCHEDULER_SIMULATE_INFLIGHT_MEMORY_LEAK=true is set,
// skip deleting pods from inFlightPods map, causing a memory leak.
if os.Getenv("SCHEDULER_SIMULATE_INFLIGHT_MEMORY_LEAK") == "true" {
    // Keep the pod in inFlightPods to simulate memory leak
    return
}
```

This simulates the QHint memory leak pattern from issue #120622 where pods are retained in memory indefinitely.

## How to Run the Demo

1. Run Baseline Test (No Memory Leak)

```bash
cd test/integration/scheduler_perf/memory
go test -run=^$ -bench=BenchmarkPerfScheduling/MemoryBaseline/15000Nodes_LongRunning \
  -benchtime=1ns -v -timeout=30m -data-items-dir=./demo/baseline \
  -perf-scheduling-label-filter=memory-demo
```

2. Run Memory Leak Test

```bash
export SCHEDULER_SIMULATE_INFLIGHT_MEMORY_LEAK=true
go test -run=^$ -bench=BenchmarkPerfScheduling/MemoryBaseline/15000Nodes_LongRunning \
  -benchtime=1ns -v -timeout=30m -data-items-dir=./demo/leak \
  -perf-scheduling-label-filter=memory-demo
```

3. Compare Results

Extract and compare memory metrics:

```bash
# Baseline memory usage
jq '.dataItems[] | select(.labels.Metric == "HeapMemoryInUse")' demo/baseline/*.json

# Memory leak test results
jq '.dataItems[] | select(.labels.Metric == "HeapMemoryInUse")' demo/leak/*.json
```

## Results

[results-summary.md](results-summary.md) provides a detailed comparison of memory metrics between the baseline and memory leak tests.

### Metrics

- **HeapMemoryInUse Max**: Shows peak memory usage
- **MemoryGrowthRate**: Rate of memory increase (MB/min)
- **HeapMemoryInUse Average**: Overall memory consumption pattern

## Performance Impact of Memory Tracking

We conducted extensive testing to ensure the memory tracking feature has minimal impact on scheduler performance:

### Test Configuration

- **Test Case**: BenchmarkPerfScheduling/SchedulingBasic/500Nodes (1500 pods)
- **Iterations**: 10 runs each with/without memory collector
- **Sampling Interval**: 500ms

### Results Summary

| Metric          | Without Memory Collector | With Memory Collector | Impact     |
| --------------- | ------------------------ | --------------------- | ---------- |
| Mean Throughput | 155.8 pods/sec           | 155.5 pods/sec        | **-0.17%** |
| Std Deviation   | 2.7 pods/sec             | 5.4 pods/sec          | -          |

**Verdict**: Performance impact is negligible and within acceptable range (±5%)
