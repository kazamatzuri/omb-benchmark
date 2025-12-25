# OMB Message Size Distribution Feature

## Implementation Plan

**Goal**: Add histogram-based message size distribution to OMB for realistic production load testing.

**Use Case**: Regular distributed load testing of Pulsar cluster at 2M+ msgs/s with production-realistic size profiles.

**Approach**: Fork https://github.com/openmessaging/benchmark, add single feature, maintain compatibility.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         WorkloadGenerator                                │
│  1. Parse messageSizeDistribution config                                │
│  2. Create ONE payload per bucket (8 buckets = 8 payloads, ~4MB total)  │
│  3. Extract weights array from distribution                              │
│  4. Send payloads + weights to workers via ProducerWorkAssignment       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           LocalWorker                                    │
│  1. Receive payloads (one per bucket) + weights array                   │
│  2. Build cumulative weight array for O(log n) selection                │
│  3. For each message: weighted random select → pick payload → send      │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key Design Decision**: Only ONE payload is created per size bucket. Workers perform weighted random selection at runtime to choose which payload to send. This keeps network transfer minimal (~4MB vs hundreds of MB).

---

## 1. Workload Configuration

### Target YAML Format

```yaml
name: pulsar-prod-load-test

topics: 10
partitionsPerTopic: 100

# NEW: Replaces messageSize with distribution
messageSizeDistribution:
  "0-256": 234
  "256-1024": 456
  "1024-4096": 678
  "4096-16384": 312
  "16384-65536": 98
  "65536-262144": 45
  "262144-1048576": 18
  "1048576-5242880": 6

useRandomizedPayloads: true
randomBytesRatio: 0.5

subscriptionsPerTopic: 1
consumerPerSubscription: 10
producersPerTopic: 20

producerRate: 2500000
consumerBacklogSizeGB: 0
testDurationMinutes: 60
```

### Behavior

- `messageSizeDistribution` and `messageSize` are mutually exclusive
- Weights are relative (your histogram counts work directly)
- One payload created per bucket, sized at midpoint of range
- Workers select payload based on distribution weights at runtime

---

## 2. Files to Change

```
benchmark-framework/src/main/java/io/openmessaging/benchmark/
├── Workload.java                              # [MODIFY] Add distribution field
├── WorkloadGenerator.java                     # [MODIFY] Generate one payload per bucket
├── worker/
│   ├── LocalWorker.java                       # [MODIFY] Weighted payload selection
│   └── commands/
│       └── ProducerWorkAssignment.java        # [MODIFY] Add payloadWeights field
└── utils/payload/
    └── MessageSizeDistribution.java           # [NEW] Distribution parsing + weights
```

**5 files total**: 1 new class, 4 modifications.

---

## 3. Implementation

### 3.1 MessageSizeDistribution.java (NEW)

```java
package io.openmessaging.benchmark.utils.payload;

import java.util.*;

/**
 * Parses and represents a message size distribution from workload config.
 * Creates one payload size per bucket and provides weights for runtime selection.
 */
public class MessageSizeDistribution {
    
    private final List<Bucket> buckets;
    private final int totalWeight;
    
    public static class Bucket {
        public final int minSize;
        public final int maxSize;
        public final int weight;
        
        Bucket(int minSize, int maxSize, int weight) {
            this.minSize = minSize;
            this.maxSize = maxSize;
            this.weight = weight;
        }
        
        /** Returns midpoint of range as the representative size for this bucket */
        public int getRepresentativeSize() {
            return (minSize + maxSize) / 2;
        }
    }
    
    public MessageSizeDistribution(Map<String, Integer> config) {
        List<Bucket> parsed = new ArrayList<>();
        int total = 0;
        
        for (Map.Entry<String, Integer> e : config.entrySet()) {
            int[] range = parseRange(e.getKey());
            int weight = e.getValue();
            parsed.add(new Bucket(range[0], range[1], weight));
            total += weight;
        }
        
        this.buckets = parsed;
        this.totalWeight = total;
        
        if (totalWeight <= 0) {
            throw new IllegalArgumentException("Distribution weights must be positive");
        }
    }
    
    private int[] parseRange(String range) {
        String[] parts = range.split("-");
        if (parts.length != 2) {
            throw new IllegalArgumentException("Invalid range format: " + range);
        }
        return new int[]{parseSize(parts[0]), parseSize(parts[1])};
    }
    
    private int parseSize(String s) {
        s = s.trim().toUpperCase();
        int mult = 1;
        if (s.endsWith("KB")) { mult = 1024; s = s.substring(0, s.length()-2); }
        else if (s.endsWith("MB")) { mult = 1024*1024; s = s.substring(0, s.length()-2); }
        else if (s.endsWith("B")) { s = s.substring(0, s.length()-1); }
        return Integer.parseInt(s.trim()) * mult;
    }
    
    /** Returns list of representative sizes, one per bucket (for payload generation) */
    public List<Integer> getBucketSizes() {
        List<Integer> sizes = new ArrayList<>();
        for (Bucket b : buckets) {
            sizes.add(b.getRepresentativeSize());
        }
        return sizes;
    }
    
    /** Returns weights array matching bucket order (for runtime selection) */
    public int[] getWeights() {
        int[] weights = new int[buckets.size()];
        for (int i = 0; i < buckets.size(); i++) {
            weights[i] = buckets.get(i).weight;
        }
        return weights;
    }
    
    /** Returns cumulative weights array for O(log n) binary search selection */
    public int[] getCumulativeWeights() {
        int[] cumulative = new int[buckets.size()];
        int sum = 0;
        for (int i = 0; i < buckets.size(); i++) {
            sum += buckets.get(i).weight;
            cumulative[i] = sum;
        }
        return cumulative;
    }
    
    public int getTotalWeight() {
        return totalWeight;
    }
    
    public int getMaxSize() {
        return buckets.stream().mapToInt(b -> b.maxSize).max().orElse(0);
    }
    
    public int getAvgSize() {
        long sum = 0;
        for (Bucket b : buckets) {
            sum += (long) b.getRepresentativeSize() * b.weight;
        }
        return (int)(sum / totalWeight);
    }
    
    public int getBucketCount() {
        return buckets.size();
    }
}
```

### 3.2 Workload.java (MODIFY)

Add field to existing class:

```java
// Add to existing fields (around line 30):
public Map<String, Integer> messageSizeDistribution;

// Add helper method:
public boolean usesDistribution() {
    return messageSizeDistribution != null && !messageSizeDistribution.isEmpty();
}
```

### 3.3 ProducerWorkAssignment.java (MODIFY)

Add weights field for runtime selection:

```java
package io.openmessaging.benchmark.worker.commands;

import io.openmessaging.benchmark.utils.distributor.KeyDistributorType;
import java.util.List;

public class ProducerWorkAssignment {

    public List<byte[]> payloadData;
    
    // NEW: Weights for weighted payload selection (null = uniform selection)
    public int[] payloadWeights;

    public double publishRate;

    public KeyDistributorType keyDistributorType;

    public ProducerWorkAssignment withPublishRate(double publishRate) {
        ProducerWorkAssignment copy = new ProducerWorkAssignment();
        copy.keyDistributorType = this.keyDistributorType;
        copy.payloadData = this.payloadData;
        copy.payloadWeights = this.payloadWeights;  // NEW
        copy.publishRate = publishRate;
        return copy;
    }
}
```

### 3.4 WorkloadGenerator.java (MODIFY)

Replace payload generation logic (around lines 98-120):

```java
// Replace existing payload generation with:

if (workload.usesDistribution()) {
    // Distribution mode: one payload per bucket
    MessageSizeDistribution dist = new MessageSizeDistribution(workload.messageSizeDistribution);
    List<Integer> sizes = dist.getBucketSizes();
    Random r = new Random();
    
    log.info("Creating {} payloads for size distribution (sizes: {})", sizes.size(), sizes);
    
    for (int size : sizes) {
        byte[] payload = new byte[size];
        if (workload.useRandomizedPayloads) {
            int randomBytes = (int)(size * workload.randomBytesRatio);
            r.nextBytes(payload);
            // Zero out non-random portion for compressibility testing
            for (int j = randomBytes; j < size; j++) {
                payload[j] = 0;
            }
        }
        producerWorkAssignment.payloadData.add(payload);
    }
    producerWorkAssignment.payloadWeights = dist.getWeights();
    
} else if (workload.useRandomizedPayloads) {
    // Existing fixed-size randomized payload logic
    Random r = new Random();
    int randomBytes = (int) (workload.messageSize * workload.randomBytesRatio);
    int zerodBytes = workload.messageSize - randomBytes;
    for (int i = 0; i < workload.randomizedPayloadPoolSize; i++) {
        byte[] randArray = new byte[randomBytes];
        r.nextBytes(randArray);
        byte[] zerodArray = new byte[zerodBytes];
        byte[] combined = ArrayUtils.addAll(randArray, zerodArray);
        producerWorkAssignment.payloadData.add(combined);
    }
} else {
    // Existing file-based payload logic
    producerWorkAssignment.payloadData.add(payloadReader.load(workload.payloadFile));
}
```

Also update backlog calculation in `buildAndDrainBacklog()` method:

```java
// Replace fixed messageSize with effective size:
int effectiveMessageSize = workload.usesDistribution() 
    ? new MessageSizeDistribution(workload.messageSizeDistribution).getAvgSize()
    : workload.messageSize;

// Use effectiveMessageSize instead of workload.messageSize in calculations
```

### 3.5 LocalWorker.java (MODIFY)

Update `submitProducersToExecutor` method for weighted selection:

```java
private void submitProducersToExecutor(
        List<BenchmarkProducer> producers, 
        KeyDistributor keyDistributor, 
        List<byte[]> payloads,
        int[] weights) {  // NEW parameter
    
    // Build cumulative weights for O(log n) weighted selection
    final int[] cumulative;
    final int totalWeight;
    if (weights != null && weights.length > 0) {
        cumulative = new int[weights.length];
        int sum = 0;
        for (int i = 0; i < weights.length; i++) {
            sum += weights[i];
            cumulative[i] = sum;
        }
        totalWeight = sum;
    } else {
        cumulative = null;
        totalWeight = payloads.size();
    }
    
    executor.submit(
            () -> {
                try {
                    ThreadLocalRandom r = ThreadLocalRandom.current();
                    while (!testCompleted) {
                        producers.forEach(
                                p -> {
                                    int idx;
                                    if (cumulative != null) {
                                        // Weighted selection via binary search
                                        int target = r.nextInt(totalWeight);
                                        idx = Arrays.binarySearch(cumulative, target + 1);
                                        if (idx < 0) idx = -idx - 1;
                                        idx = Math.min(idx, payloads.size() - 1);
                                    } else {
                                        // Uniform selection (backward compatible)
                                        idx = r.nextInt(payloads.size());
                                    }
                                    messageProducer.sendMessage(
                                            p,
                                            Optional.ofNullable(keyDistributor.next()),
                                            payloads.get(idx));
                                });
                    }
                } catch (Throwable t) {
                    log.error("Got error", t);
                }
            });
}
```

Update `startLoad` method to pass weights:

```java
@Override
public void startLoad(ProducerWorkAssignment producerWorkAssignment) {
    // ... existing code ...
    
    processorAssignment
            .values()
            .forEach(
                    producers ->
                            submitProducersToExecutor(
                                    producers,
                                    KeyDistributor.build(producerWorkAssignment.keyDistributorType),
                                    producerWorkAssignment.payloadData,
                                    producerWorkAssignment.payloadWeights));  // NEW
}
```

---

## 4. Build & Deploy

### 4.1 Build

```bash
cd benchmark
git checkout -b feature/size-distribution

# Make changes per above

mvn clean package -DskipTests
```

### 4.2 Deploy Workers (for 2M+ msg/s)

For 2M msg/s with your size profile (avg ~2KB based on your histogram), you'll need roughly:

- **Throughput**: ~4 GB/s aggregate
- **Workers**: 8-16 worker nodes (depending on instance size)
- **Worker sizing**: c5.4xlarge or similar (16 vCPU, 32GB RAM)

```bash
# On each worker node:
bin/benchmark-worker --port 8080

# On driver node:
bin/benchmark \
  --drivers driver-pulsar/pulsar.yaml \
  --workers http://worker1:8080,http://worker2:8080,... \
  workloads/prod-load-test.yaml
```

### 4.3 Docker Image (for K8s deployment)

```dockerfile
FROM openmessaging/openmessaging-benchmark:latest

# Copy modified JARs
COPY benchmark-framework/target/*.jar /opt/benchmark/lib/
```

---

## 5. Test Workloads

### 5.1 Your Production Profile

```yaml
# workloads/pulsar-prod-2m.yaml
name: Production load test - 2M msg/s

topics: 10
partitionsPerTopic: 100

messageSizeDistribution:
  "0-256": 234
  "256-1024": 456
  "1024-4096": 678
  "4096-16384": 312
  "16384-65536": 98
  "65536-262144": 45
  "262144-1048576": 18
  "1048576-5242880": 6

useRandomizedPayloads: true
randomBytesRatio: 0.5

subscriptionsPerTopic: 1
consumerPerSubscription: 10
producersPerTopic: 20

producerRate: 2500000
consumerBacklogSizeGB: 0
testDurationMinutes: 60

warmupDurationMinutes: 5
```

### 5.2 Soak Test (longer duration)

```yaml
# workloads/pulsar-soak-24h.yaml
name: 24-hour soak test

topics: 10
partitionsPerTopic: 100

messageSizeDistribution:
  "0-256": 234
  "256-1024": 456
  "1024-4096": 678
  "4096-16384": 312
  "16384-65536": 98
  "65536-262144": 45
  "262144-1048576": 18
  "1048576-5242880": 6

useRandomizedPayloads: true
randomBytesRatio: 0.5

subscriptionsPerTopic: 1
consumerPerSubscription: 10
producersPerTopic: 20

producerRate: 2000000  # Slightly under peak for stability
consumerBacklogSizeGB: 0
testDurationMinutes: 1440  # 24 hours

warmupDurationMinutes: 10
```

---

## 6. Implementation Tasks

| # | Task | Est | Notes |
|---|------|-----|-------|
| 1 | Create `MessageSizeDistribution.java` | 45m | Parsing + weights |
| 2 | Modify `Workload.java` | 15m | Add field + helper |
| 3 | Modify `ProducerWorkAssignment.java` | 15m | Add weights field |
| 4 | Modify `WorkloadGenerator.java` | 45m | Bucket payload generation |
| 5 | Modify `LocalWorker.java` | 45m | Weighted selection logic |
| 6 | Unit tests for `MessageSizeDistribution` | 30m | Parsing + sampling |
| 7 | Integration test (local) | 30m | Single worker validation |
| 8 | Distributed test | 1h | Multi-worker at scale |
| **Total** | | **~5h** | |

---

## 7. Validation Checklist

- [ ] Distribution parsing handles all bucket formats (bytes, KB, MB)
- [ ] Weighted selection produces correct proportions (test with 100K samples)
- [ ] Backward compatible: existing `messageSize` workloads still work
- [ ] Backward compatible: `payloadWeights = null` falls back to uniform selection
- [ ] Backlog calculations use average size when distribution is used
- [ ] Distributed mode: workers receive payloads + weights correctly
- [ ] Network transfer is minimal (~4MB for 8 buckets, not 100s of MB)

---

## 8. Data Transfer Calculation

With your example distribution (8 buckets, max 5MB):

| Bucket | Range | Representative Size |
|--------|-------|---------------------|
| 1 | 0-256 | 128 B |
| 2 | 256-1024 | 640 B |
| 3 | 1024-4096 | 2.5 KB |
| 4 | 4096-16384 | 10 KB |
| 5 | 16384-65536 | 40 KB |
| 6 | 65536-262144 | 160 KB |
| 7 | 262144-1048576 | 640 KB |
| 8 | 1048576-5242880 | 3 MB |

**Total per worker: ~4 MB** (vs. potentially 500MB+ with a large pre-generated pool)

---

## 9. Pulsar Driver Config

For 2M+ msg/s, tune the Pulsar driver (`driver-pulsar/pulsar.yaml`):

```yaml
name: pulsar-high-throughput
driverClass: io.openmessaging.benchmark.driver.pulsar.PulsarBenchmarkDriver

client:
  serviceUrl: pulsar://pulsar-broker:6650
  ioThreads: 8
  connectionsPerBroker: 4

producer:
  batchingEnabled: true
  batchingMaxMessages: 1000
  batchingMaxPublishDelayMs: 1
  blockIfQueueFull: true
  maxPendingMessages: 10000
  maxPendingMessagesAcrossPartitions: 100000

consumer:
  receiverQueueSize: 10000
```

---

## 10. Out of Scope (Future)

- Per-bucket latency breakdown in results
- Rate limiting by MB/s instead of msg/s  
- Loading distribution from external file/API
- Real-time distribution adjustment during test
- Multiple payloads per bucket for more variety
