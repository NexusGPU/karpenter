# Other Daemon Pods Support

## Summary

This design document describes the implementation of support for other daemon pod types in Karpenter's scheduling and provisioning logic. This feature allows Karpenter to recognize and properly handle daemon-like pods that are owned by custom resource types beyond the standard Kubernetes DaemonSet.

## Motivation

Karpenter currently only recognizes pods owned by DaemonSets when calculating node capacity and determining whether pods should be considered for scheduling. However, in many real-world environments, there are custom daemon-like workloads managed by operators or custom controllers that follow the same pattern as DaemonSets but are not technically DaemonSets.

### Problems

1. **Incorrect Scheduling Decisions**: Pods owned by custom daemon controllers may be treated as regular workloads, causing Karpenter to provision unnecessary nodes
2. **Resource Calculation Errors**: Custom daemon pods are not accounted for in node capacity calculations, leading to over-provisioning
3. **Inefficient Cluster Management**: Without proper daemon recognition, Karpenter cannot make optimal scaling decisions

### Goals

- Enable Karpenter to recognize custom daemon pod types through configuration
- Ensure custom daemon pods are excluded from provisioning decisions (similar to DaemonSet pods)
- Include custom daemon pod resource requirements in node capacity calculations
- Maintain backward compatibility with existing functionality
- Provide a performant solution suitable for large clusters

## Design

### Configuration

A new configuration option `OtherDaemonGVKs` is introduced to specify which GroupVersionKind (GVK) combinations should be treated as daemon pods:

```go
type Options struct {
    // ... existing fields
    OtherDaemonGVKs []string
}
```

The configuration accepts a list of GVK strings in the format `group/version/kind`:

```bash
# Environment variable
export OTHER_DAEMON_GVKS="custom.io/v1/CustomDaemon,operator.io/v1/OperatorDaemon"

# Command line flag
karpenter --other-daemon-gvks=custom.io/v1/CustomDaemon,operator.io/v1/OperatorDaemon
```

### Implementation Details

#### Pod Classification Functions

New context-aware functions are introduced to identify other daemon pods:

```go
// Context-aware version that reads configuration
func IsOwnedByOtherDaemonPods(ctx context.Context, pod *corev1.Pod) bool

// Enhanced scheduling functions
func IsProvisionableWithContext(ctx context.Context, pod *corev1.Pod) bool
func IsReschedulableWithContext(ctx context.Context, pod *corev1.Pod) bool
```

#### Cluster State Extension

The cluster state is extended to efficiently track other daemon pods:

```go
type Cluster struct {
    // ... existing fields
    otherDaemonPods sync.Map                        // other daemon pods key -> pod
    otherDaemonGVKs []schema.GroupVersionKind       // configured other daemon GVKs
}
```

#### Integration Points

1. **Pod Informer**: Automatically detects and caches other daemon pods based on configured GVKs
2. **Provisioner**: Includes other daemon pods in scheduling overhead calculations
3. **StateNode**: Tracks other daemon pod resource usage alongside DaemonSet pods
4. **Scheduler**: Considers other daemon pods in node compatibility checks

### Performance Considerations

- **O(1) Access**: Other daemon pods are cached in cluster state using sync.Map for efficient concurrent access
- **No Additional API Calls**: Leverages existing pod informer events rather than scanning all pods
- **Minimal Memory Overhead**: Reuses existing data structures and patterns from DaemonSet handling

### Backward Compatibility

- All existing APIs remain unchanged
- New context-aware functions are additive
- Legacy functions maintain their original behavior by falling back to `context.Background()`
- No impact on clusters that don't configure other daemon GVKs

## Examples

### Basic Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karpenter
spec:
  template:
    spec:
      containers:
      - name: controller
        env:
        - name: OTHER_DAEMON_GVKS
          value: "custom.io/v1/CustomDaemon,monitoring.io/v1/MonitoringAgent"
```

### Helm Configuration

```yaml
# values.yaml
controller:
  env:
    - name: OTHER_DAEMON_GVKS
      value: "custom.io/v1/CustomDaemon,monitoring.io/v1/MonitoringAgent"
```

### Custom Daemon Resource Example

```yaml
apiVersion: custom.io/v1
kind: CustomDaemon
metadata:
  name: my-custom-daemon
spec:
  selector:
    matchLabels:
      app: custom-daemon
  template:
    metadata:
      labels:
        app: custom-daemon
    spec:
      containers:
      - name: daemon
        image: custom-daemon:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

### Verification

To verify that other daemon pods are being recognized:

1. Check Karpenter logs for other daemon pod detection
2. Observe that unschedulable other daemon pods don't trigger node provisioning
3. Verify that node resource calculations include other daemon pod overhead

## Testing

### Unit Tests

- GVK parsing and validation
- Pod ownership detection with various owner reference scenarios
- Context-aware function behavior
- Backward compatibility verification

### Integration Tests

- End-to-end scheduling scenarios with other daemon pods
- Resource calculation accuracy
- Performance impact measurement
- Multiple GVK configuration testing

## Security Considerations

- GVK configuration is controlled through environment variables and command-line flags
- No additional RBAC permissions required
- Input validation prevents malformed GVK specifications
- No sensitive information exposed in logs or metrics

## Alternatives Considered

### 1. Annotation-Based Approach
Instead of configuring GVKs globally, individual pods could be marked with annotations. This was rejected because:
- Requires modification of existing pod specs
- Less performant (requires checking annotations on every pod)
- More complex for operators managing multiple daemon types

### 2. CRD-Based Configuration
A custom resource could define daemon types. This was rejected because:
- Adds complexity with additional CRD management
- Overkill for simple GVK list configuration
- Would require additional RBAC permissions

### 3. Automatic Detection
Automatically detect daemon-like behavior based on pod patterns. This was rejected because:
- Too heuristic and prone to false positives
- Difficult to define reliable detection criteria
- May not work for all daemon patterns

## Future Enhancements

1. **Dynamic Configuration**: Support for runtime configuration updates without restart
2. **Metrics Integration**: Add metrics to track other daemon pod counts and resource usage
3. **Advanced Filtering**: Support for additional pod filtering criteria beyond owner references
4. **Documentation Integration**: Enhance Karpenter documentation with daemon pod best practices

## Implementation Status

- ✅ Core functionality implemented
- ✅ Configuration parsing and validation
- ✅ Context-aware pod classification
- ✅ Cluster state integration
- ✅ Provisioner and scheduler integration
- ✅ Unit and integration tests
- ✅ Backward compatibility maintained
- ✅ Performance optimized

The feature is ready for production use and maintains full compatibility with existing Karpenter functionality.