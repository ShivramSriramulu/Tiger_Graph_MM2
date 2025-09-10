# Kafka MirrorMaker 2 — Fault-Tolerant DR Demo

## Project Overview

This project focuses on enhancing Apache Kafka's MirrorMaker 2 for mission-critical data replication between primary cluster (PR) and disaster recovery (DR) cluster scenarios. The project includes:

1. **Synthetic data replication pipeline** - A complete producer → MM2 → DR setup
2. **Enhanced MirrorMaker 2** - Modified source code with fault-tolerance features

### Background

This project simulates a real-world data replication pipeline where Kafka topics serve as Write-Ahead Logs (WAL) containing ordered events that represent state changes in a distributed system.

- **Primary Cluster**: Events are written to `commit-log` and replayed by local services
- **Cross-Cluster Replication**: MirrorMaker 2 replicates events from `commit-log` to `primary.commit-log` in the DR cluster
- **DR Cluster**: Standby services consume from `primary.commit-log` to maintain a synchronized state

### Technical Challenge

Real-world Kafka deployments face two critical scenarios that can compromise data integrity:

1. **Silent Data Loss**: Kafka retention policies may purge data from source topics before replication completes, creating undetectable gaps in the replicated data stream.
2. **Service Disruption**: Planned maintenance operations involving topic reset (topic deletion and recreation) can cause the replication service to be unable to find the expected offset and stop replication.

### Project Objective

Enhance MirrorMaker 2 with intelligent fault detection and automatic recovery capabilities to handle these scenarios gracefully.

---

## Architecture

### Cluster Setup
- **Primary Cluster**: Single-node Kafka cluster hosting `commit-log` topic
- **Standby Cluster**: Single-node Kafka cluster hosting `primary.commit-log` topic
- **Topic Configuration**: Both topics with exactly 1 partition, 1 replica
- **Test Configuration**: Set `log.retention.ms=60000` (60 seconds) on `commit-log` for truncation testing

### Components

#### Commit Log Producer
CLI application generating JSON events to the primary cluster's `commit-log` topic.

**Requirements:**
- Accept `--count N` parameter to produce exactly N messages then exit
- Generate valid JSON with unique UUIDs and current timestamps

**Event Schema:**
```json
{
  "event_id": "a8a1c867-05c3-4d43-9884-f7b55f1f0a7c",
  "timestamp": 1724684407,
  "op_type": "UPDATE",
  "key": "doc:8f7b",
  "value": {
    "status": "archived"
  }
}
```

#### Enhanced MirrorMaker 2
Modified MirrorMaker 2 service with error-handling features.

**Fault-Tolerance Features:**
- **Fail-fast on truncation**: Detect offset gaps and throw exception immediately
- **Auto-recover on topic reset**: Detect topic deletion/recreation and resubscribe from beginning

---

## Core Tasks Implementation

### Task 1: Commit Log Producer 

**Objective**: CLI application generating JSON events to the primary cluster's commit-log topic.

**Implementation**: `producer/commitlog_producer.py`
- Accepts `--count N` parameter
- Generates exactly N messages with unique UUIDs and timestamps
- Uses proper JSON schema as specified
- Exits cleanly after producing all messages

### Task 2: Log Truncation Detection (Fail-Fast) 

**Problem**: Aggressive retention policies may purge messages before replication, causing silent data loss.

**Solution**: Enhanced MirrorMaker 2 to detect and respond to log truncation.

**Technical Implementation**:
- Detects `OffsetOutOfRangeException` during consumer polling
- Logs detailed error with earliest offsets and assignment information
- Throws `ConnectException` to fail-fast and alert operators immediately
- Configurable via `mirrorsource.fail.on.truncation=true`

### Task 3: Graceful Topic Reset Handling 

**Problem**: Topic deletion/recreation can cause MirrorMaker 2 failures or stalls.

**Solution**: Add automatic recovery capabilities to MirrorMaker 2.

**Technical Implementation**:
- Uses `AdminClient` to track topic IDs and detect delete/recreate events
- Logs reset events with timestamp and topic details
- Automatically seeks to beginning offset for reset topics
- Handles `UnknownTopicOrPartitionException` with retry logic
- Configurable via `mirrorsource.auto.recover.on.reset=true`

---

## Deliverables

### 1. Source Code

#### Repository Links
- **GitHub Repository**: [https://github.com/ShivramSriramulu/Tiger_Graph_MM2](https://github.com/ShivramSriramulu/Tiger_Graph_MM2)
- **Kafka Fork**: [https://github.com/ShivramSriramulu/kafka](https://github.com/ShivramSriramulu/kafka) (Fork of [Apache Kafka](https://github.com/apache/kafka))
- **Pull Request**: `MM2: fail-fast on truncation + auto-recover on topic reset (MirrorSourceTask)`
- **Patch Location**: `mm2/patches/mm2-fault-tolerance.patch` (≤500 LOC)

#### Docker Hub Images
- **Enhanced MM2**: [shivramsriramulu/enhanced-mm2:latest](https://hub.docker.com/r/shivramsriramulu/enhanced-mm2)
- **Commit Log Producer**: [shivramsriramulu/commitlog-producer:latest](https://hub.docker.com/r/shivramsriramulu/commitlog-producer)

### 2. Docker Compose Setup

Complete environment setup including:
- Primary Kafka cluster (single-node KRaft)
- Standby Kafka cluster (single-node KRaft)
- Enhanced MirrorMaker 2 (using custom image)
- Commit Log Producer
- Bounded verifier for testing

### 3. Automation Scripts

**`run_challenge.sh`**: Orchestrates test scenarios using docker-compose:

1. **Normal replication flow**: Producer generates 1000 messages, verifies replication
2. **Log truncation simulation**: Trigger truncation, verify MirrorMaker 2 detects and fails
3. **Topic reset simulation**: Delete/recreate topic, verify MirrorMaker 2 recovers automatically

### 4. Documentation

This README includes all required sections:
- Repository links and pull request information
- Docker Hub images with tags
- Setup instructions and initial configuration
- Test execution guide and result interpretation
- Log analysis guidance
- Design rationale and integration approach
- AI usage documentation

---

## Setup Instructions

### Prerequisites
- Docker Desktop
- Git
- Bash

### Quick Start

```bash
# Clone repository
git clone https://github.com/ShivramSriramulu/Tiger_Graph_MM2.git
cd Tiger_Graph_MM2

# Start environment
docker compose up -d

# Run complete demo
./run_challenge.sh
```

### Manual Setup
###— Docker Images & Containerization
**2.1 Built Custom Images**
- Enhanced MM2 image (built from Kafka 4.0.0 + patch)
- Producer image (Python + kafka-python)

**Pushed to Docker Hub**
- Enhanced MM2: **https://hub.docker.com/r/shivramsriramulu/enhanced-mm2** (tag: `latest`)
- Producer: **https://hub.docker.com/r/shivramsriramulu/commitlog-producer** (tag: `latest`)

---

### GitHub Repository Setup (Demo)
- **Main Project Repository:** **https://github.com/ShivramSriramulu/Tiger_Graph_MM2**
- Includes: `docker-compose.yml`, `run_challenge.sh`, producer, verifier, MM2 enhanced Dockerfile, patch, README

---

### Kafka Fork & Pull Request
- **Forked Kafka (4.0.0):** **https://github.com/ShivramSriramulu/kafka**
- **Branch:** **mm2-fault-tolerance-enhancement**  
  https://github.com/ShivramSriramulu/kafka/tree/mm2-fault-tolerance-enhancement
- **PR link (create/new):**  
  https://github.com/ShivramSriramulu/kafka/pull/new/mm2-fault-tolerance-enhancement 
```bash
# 1. Start clusters
docker compose up -d kafka-primary kafka-dr

# 2. Create topic with 60s retention
docker exec kafka-primary /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server kafka-primary:9092 \
  --create --topic commit-log --partitions 1 --replication-factor 1

docker exec kafka-primary /opt/kafka/bin/kafka-configs.sh \
  --bootstrap-server kafka-primary:9092 \
  --alter --topic commit-log --add-config retention.ms=60000

# 3. Start MM2
docker compose up -d mm2-vanilla

# 4. Produce messages
docker compose run --rm producer --count 1000

# 5. Verify replication
docker compose run --rm -e EXPECT=1000 -e TIMEOUT=20 verifier
```

---

## Test Execution

### Running the Demo

```bash
# Complete end-to-end demo (takes ~3-4 minutes)
./run_challenge.sh
```

### Expected Results

1. **Baseline Replication**:
   ```
   produced 1000
   verifier → 1000/1000
   ```

2. **Truncation Test**:
   ```
   TRUNCATION_DETECTED: Source offsets are out of range (likely retention purge)
   Enhanced MM2 exits with ConnectException (by design)
   ```

3. **Topic Reset Test**:
   ```
   RESET_DETECTED: Topic commit-log recreated (oldId=... newId=...)
   Seeking to beginning
   Mirroring continues with new events
   ```

---

## Log Analysis

### Key Log Messages to Monitor

#### Truncation Detection
```bash
docker logs mm2-enhanced | grep "TRUNCATION_DETECTED"
```
**Expected**: `ERROR: TRUNCATION_DETECTED: Source offsets are out of range (likely retention purge). Earliest per partition={...}, assignment={...}`

#### Topic Reset Detection
```bash
docker logs mm2-enhanced | grep "RESET_DETECTED"
```
**Expected**: `WARN: RESET_DETECTED: Topic commit-log recreated (oldId=... newId=...). Seeking to beginning.`

#### Topic Reset Suspected
```bash
docker logs mm2-enhanced | grep "TOPIC_RESET_SUSPECTED"
```
**Expected**: `WARN: TOPIC_RESET_SUSPECTED: {reason}. Will retry metadata and resubscribe in 5000 ms.`

### Container Logs to Monitor

```bash
# Enhanced MM2 logs
docker logs mm2-enhanced --tail=50

# Producer logs
docker logs commitlog-producer

# Verifier output
docker logs verifier
```

---
## Replication Models: Sync vs Async

| **Dimension**          | **Synchronous Replication** | **Asynchronous Replication (Used in this Project)** |
|-------------------------|-----------------------------|-----------------------------------------------------|
| **Write Acknowledgement** | Waits until data is copied to all replicas before confirming success | Confirms write on primary immediately; replication happens later |
| **Data Consistency**   | Strong consistency, no data loss between replicas | Eventual consistency, small risk of lag or loss if primary fails |
| **Latency**            | Higher (slower writes)      | Lower (faster writes, but replicas may lag) |
| **Fault Tolerance**    | Safer, no acknowledged writes are lost | May lose the last few writes if failure occurs before replication |
| **Best For**           | Critical financial or healthcare systems | High-volume event streaming, analytics, DR pipelines |

👉 **This project uses Asynchronous Replication** with MirrorMaker 2, since it’s the standard model for cross-cluster disaster recovery.

---

## TigerGraph Disaster Recovery Features

TigerGraph’s approach to **high availability (HA)** and **disaster recovery (DR)** aligns closely with the asynchronous replication model used here:

- **Cluster-Level HA**: Data is sharded and replicated within the cluster to survive node failures.  
- **Cross-Cluster DR**: Kafka/Data Source connectors replicate updates to a remote standby cluster.  
- **Operational Controls**: Simple commands (`RUN/SHOW/ABORT/RESUME`) for loading jobs, and `CLEAR GRAPH STORE`/`DROP ALL` for controlled resets.  
- **Monitoring & Logs**: Detailed logs and error sampling (since v4.1) provide visibility into ingestion health.  
- **Backup & Restore**: Full and incremental backups ensure quick recovery from corruption or loss.  
- **Concurrency & Scale**: Distributed loaders allow fast catch-up if the DR cluster falls behind.

**Summary:**  
- Inside a TigerGraph cluster → **Sync replication** ensures strict consistency.  
- Across regions for DR → **Async replication** (like this project) balances performance with resilience, delivering low RPO and manageable RTO.  

---

## DR Architecture Diagram

```mermaid
flowchart LR
    A[Primary Cluster] -->|Async Replication| B[DR Cluster]
    A --> C[Local Services consume commit-log]
    B --> D[Standby Services consume primary.commit-log]

----
## Design Rationale

### MirrorMaker 2 Modifications

The enhancements are implemented as a focused patch to `MirrorSourceTask.java`:

1. **Fail-Fast Truncation Detection**:
   - Catches `OffsetOutOfRangeException` during consumer polling
   - Provides detailed diagnostics with partition assignments and earliest offsets
   - Throws `ConnectException` to immediately surface data loss

2. **Topic Reset Auto-Recovery**:
   - Uses `AdminClient` to track topic IDs and detect changes
   - Automatically seeks to beginning offset for reset topics
   - Handles temporary topic unavailability with retry logic

3. **Configuration Toggles**:
   - `mirrorsource.fail.on.truncation=true` (default)
   - `mirrorsource.auto.recover.on.reset=true` (default)
   - `mirrorsource.topic.reset.retry.ms=5000` (default)

### Integration Approach

- **Minimal Disruption**: All changes are additive and maintain backward compatibility
- **Configurable Behavior**: Fault-tolerance features can be toggled via properties
- **Comprehensive Logging**: Uses dedicated logger `mm2.fault.tolerance` for easy filtering
- **Production Ready**: <500 LOC patch with proper error handling

---

## AI Usage Documentation

### Methodology

AI tools (Cursor, Claude) were used to accelerate development while maintaining full understanding and control over the final implementation.

### Specific Contributions

1. **Project Scaffolding**: AI helped generate initial Docker Compose configuration and project structure
2. **Code Generation**: AI assisted with producer/verifier scripts and MM2 patch outline
3. **Documentation**: AI helped draft comprehensive README and architecture documentation
4. **Troubleshooting**: AI provided guidance on Kafka internals and error handling patterns

### Human Validation

- **Code Review**: Every line of code was reviewed and validated
- **Design Decisions**: All architectural choices were made with full understanding
- **Testing**: Complete end-to-end testing was performed manually
- **Documentation**: All documentation was reviewed and refined for accuracy

### AI Prompts Used

- "Generate a Docker Compose setup for Kafka 4.0 KRaft clusters with MM2"
- "Create a Python producer script for Kafka with JSON event generation"
- "Design a fault-tolerance patch for MirrorSourceTask with fail-fast and auto-recovery"

### Value Delivered

- **Faster Iteration**: Reduced development time from weeks to days
- **Best Practices**: AI helped identify and implement Kafka best practices
- **Comprehensive Coverage**: Ensured all assessment requirements were met
- **Professional Quality**: Delivered production-ready code and documentation

---

## Code Quality Standards

### Production-Ready Code
- Clean, efficient, well-structured implementation
- Proper error handling and logging mechanisms
- Comprehensive input validation and edge case handling

### Modular Design
- Minimal disruption to existing Kafka codebase
- Additive changes that maintain backward compatibility
- Clear separation of concerns with focused responsibilities

### Comprehensive Logging
- Clear error messages using SLF4J
- Structured logging with appropriate log levels
- Easy-to-parse log messages for operational monitoring

### Pull Request Quality
- Clear commit messages with descriptive changes
- Comprehensive PR description explaining modifications
- Proper code formatting and documentation



## Conclusion

This project successfully demonstrates sophisticated fault-tolerant replication capabilities for Apache Kafka MirrorMaker 2, addressing critical data loss scenarios while maintaining production-ready code quality and comprehensive documentation. The solution provides immediate value for mission-critical cross-cluster replication scenarios and serves as a reference implementation for Kafka DR best practices.
