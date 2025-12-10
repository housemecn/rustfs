# Copilot Instructions: EC Quorum and Disk Auto-Elimination Fixes

## Overview
This document provides comprehensive guidance for GitHub Copilot when working on EC (Erasure Coding) quorum management and disk auto-elimination features in the rustfs project.

## Context

### EC Quorum System
The EC quorum system manages redundancy and fault tolerance in distributed storage using erasure coding. Key responsibilities include:
- Tracking erasure coding parameters (data chunks, parity chunks)
- Managing quorum states and transitions
- Handling node/disk failures gracefully
- Ensuring data durability and availability

### Disk Auto-Elimination
Disk auto-elimination is an automated mechanism that removes underperforming or failing disks from the system:
- Monitors disk health metrics (I/O latency, error rates, throughput)
- Triggers removal when disks exceed failure thresholds
- Updates cluster topology and rebalances data
- Provides notifications and logging for operational visibility

## Code Structure Guidelines

### File Organization
```
rustfs/
├── src/
│   ├── ec/
│   │   ├── quorum.rs           # Core quorum logic
│   │   ├── manager.rs          # EC management
│   │   └── mod.rs              # EC module exports
│   ├── disk/
│   │   ├── auto_elimination.rs # Auto-elimination logic
│   │   ├── health.rs           # Disk health tracking
│   │   └── mod.rs              # Disk module exports
│   └── lib.rs                  # Main library entry
└── tests/
    ├── ec_quorum_tests.rs
    └── disk_elimination_tests.rs
```

## Key Implementation Patterns

### 1. Quorum Management

**Data Structures:**
- Use `QuorumState` enum for state management (Active, Degraded, ReadOnly, Critical)
- Maintain `ErasureCodeConfig` with data/parity chunk counts
- Track `NodeHealth` for each participant

**Error Handling:**
- Use `Result<T, QuorumError>` for fallible operations
- Define specific error variants: InsufficientQuorum, InvalidTransition, DataLoss
- Propagate errors with context using `map_err()`

**Thread Safety:**
- Use `RwLock<T>` for quorum state (read-heavy workloads)
- Use `Mutex<T>` for state transitions (write-heavy operations)
- Avoid deadlocks by acquiring locks in consistent order

### 2. Disk Health Monitoring

**Metrics Collection:**
- Track `ReadLatencyMs`, `WriteLatencyMs`, `ErrorCount`, `SuccessRate`
- Use sliding window (e.g., 5-minute windows) for trend analysis
- Implement exponential moving average for smoothing

**Thresholds:**
```rust
pub struct DiskHealthThresholds {
    max_latency_ms: u64,           // e.g., 500ms
    max_error_rate: f64,           // e.g., 0.05 (5%)
    min_success_rate: f64,         // e.g., 0.95 (95%)
    consecutive_failures: usize,   // e.g., 3 consecutive timeouts
}
```

**Auto-Elimination Flow:**
1. Collect health metrics
2. Evaluate against thresholds
3. Trigger elimination if thresholds exceeded
4. Update quorum state
5. Initiate data rebalancing
6. Emit events/logs

### 3. State Transitions

**Quorum State Machine:**
```
Active → Degraded → ReadOnly
  ↓        ↓          ↓
  └────────┴──────────┘
         ↓
      Critical (data loss risk)
```

**Disk Health State Machine:**
```
Healthy → Warning → Eliminated
   ↓         ↓           ↓
   └─────────┴───────────┘
```

## Implementation Best Practices

### Code Quality
- **Logging**: Use `debug!`, `info!`, `warn!`, `error!` macros appropriately
  - `debug!`: Low-level implementation details
  - `info!`: State transitions, major operations
  - `warn!`: Threshold violations, approaching limits
  - `error!`: Critical failures, data loss risks

- **Testing**: Write tests for:
  - State transitions (valid and invalid)
  - Health metric calculations
  - Quorum calculations under various failure scenarios
  - Edge cases (simultaneous disk failures, network partitions)

- **Documentation**: 
  - Add doc comments to public APIs
  - Include examples in doc comments for complex types
  - Document assumptions and invariants

### Performance Considerations
- Cache quorum calculations when possible
- Use batch health metric updates
- Implement exponential backoff for retry logic
- Consider lock-free data structures for high-frequency reads

### Configuration and Tunability
- Externalize thresholds to configuration files
- Support runtime configuration updates
- Provide CLI tools to override defaults
- Log all configuration values at startup

## Common Scenarios and Solutions

### Scenario 1: Multiple Disks Failing Simultaneously
**Guidance:**
- Evaluate quorum impact before eliminating each disk
- Check if remaining replicas can maintain quorum
- May need to transition to ReadOnly state
- Prevent cascading failures by pausing auto-elimination if critical

### Scenario 2: Transient Network Latency
**Guidance:**
- Implement grace period before elimination (e.g., 3 consecutive failures)
- Use adaptive thresholds based on network conditions
- Distinguish between transient and permanent failures
- Add circuit breaker pattern to prevent flapping

### Scenario 3: Data Rebalancing After Disk Removal
**Guidance:**
- Calculate new placement strategy
- Stream data efficiently (avoid hotspots)
- Monitor rebalancing progress
- Allow prioritization of critical data

## Testing Guidelines

### Unit Tests
```rust
#[test]
fn test_quorum_state_transition_active_to_degraded() {
    // Setup, execute, assert
}

#[test]
fn test_disk_elimination_prevents_data_loss() {
    // Verify quorum maintained after elimination
}

#[test]
#[should_panic]
fn test_invalid_state_transition_panics() {
    // Test error handling
}
```

### Integration Tests
- Simulate multi-disk failures
- Verify data consistency across rebalancing
- Test metrics collection and reporting
- Validate configuration updates

### Stress Tests
- High-frequency state transitions
- Concurrent health metric updates
- Large cluster simulations (100+ disks)

## Debugging and Monitoring

### Key Metrics to Expose
- `quorum_state_transitions_total`: Counter per state
- `disk_eliminations_total`: Counter of removed disks
- `quorum_health`: Gauge (0-1) indicating system health
- `rebalancing_progress`: Gauge (0-1) of ongoing rebalancing
- `disk_health_score`: Per-disk metric

### Logging Best Practices
- Include context (node ID, disk ID, quorum ID) in all logs
- Use structured logging with key-value pairs
- Log decisions and reasoning (why a disk was eliminated)
- Audit trail for all state changes

## API Design

### Public Interface Example
```rust
pub trait QuorumManager {
    fn get_quorum_state(&self) -> Result<QuorumState>;
    fn add_node(&self, node_id: NodeId) -> Result<()>;
    fn remove_node(&self, node_id: NodeId) -> Result<()>;
    fn get_health_report(&self) -> Result<HealthReport>;
}

pub trait DiskHealthMonitor {
    fn record_metric(&self, disk_id: DiskId, metric: HealthMetric) -> Result<()>;
    fn evaluate_health(&self, disk_id: DiskId) -> Result<HealthStatus>;
    fn request_elimination(&self, disk_id: DiskId) -> Result<()>;
}
```

## Common Pitfalls to Avoid

1. **Race Conditions**: Always consider concurrent state updates
2. **Silent Failures**: Ensure errors are logged and propagated
3. **Deadlocks**: Document lock acquisition order
4. **Data Loss**: Verify quorum before any elimination
5. **Unbounded Memory**: Implement metrics retention policies
6. **Hard Thresholds**: Use multiple indicators, not single metrics
7. **Inadequate Logging**: Include enough context for debugging

## Related Documentation
- EC Quorum Architecture Design: [link to architecture doc]
- Disk Health Monitoring Specification: [link to spec]
- Data Rebalancing Algorithm: [link to algorithm]
- Operational Runbook: [link to runbook]

## Questions or Issues?
For clarification on these guidelines, refer to:
- Project README
- Contributing guidelines
- Issue tracker with label `ec-quorum` or `disk-auto-elimination`
- Team documentation wiki
