# Shared Services

This directory contains services deployed on the management cluster and exposed to HyperShift hosted clusters.

## Architecture

```
Management Cluster                    Hosted Clusters
┌─────────────────────┐              ┌─────────────────────┐
│ shared-services ns  │              │ shared-services ns  │
│ ┌─────────────────┐ │   Route      │ ┌─────────────────┐ │
│ │    Pyroscope    │─┼──────────────┼─│ ExternalName    │ │
│ │    (server)     │ │              │ │ (DNS pointer)   │ │
│ └─────────────────┘ │              │ └─────────────────┘ │
└─────────────────────┘              │ ┌─────────────────┐ │
                                     │ │   ConfigMap     │ │
                                     │ │ (endpoints)     │ │
                                     │ └─────────────────┘ │
                                     └─────────────────────┘
```

## Deployed Services

| Service | Port | Purpose | Status |
|---------|------|---------|--------|
| Pyroscope | 4040 | Continuous profiling | Implemented |
| Prometheus | 9090 | Metrics aggregation | Planned |
| Jaeger | 16686 | Distributed tracing | Planned |
| Registry | 5000 | Image caching | Planned |

## Usage

### For Application Developers

1. Ensure `shared-services` namespace exists in your hosted cluster
2. Reference the ConfigMap in your deployment:

```yaml
envFrom:
  - configMapRef:
      name: shared-services-endpoints
```

3. Configure your application SDK to use `PYROSCOPE_SERVER_ADDRESS`

### Adding New Services

1. Create manifests in `platform/shared-services/<service>/`
2. Add Route for external access
3. Update `shared-services-config/endpoints-configmap.yaml`
4. Add ExternalName service to `shared-services-config/`
5. Update ApplicationSet patches in `applications/shared-services-config-applicationset.yaml`

## Files

* `namespace.yaml` - Namespace definition
* `pyroscope/` - Pyroscope deployment, service, route, and config
