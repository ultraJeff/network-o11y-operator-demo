# Network Observability Operator Demo

This repository demonstrates the **Red Hat OpenShift Network Observability Operator** for collecting, visualizing, and analyzing network traffic flows in OpenShift clusters.

## Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     All Cluster Nodes                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ eBPF Agent  │  │ eBPF Agent  │  │ eBPF Agent  │  (DaemonSet) │
│  │  (netobserv)│  │  (netobserv)│  │  (netobserv)│              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
└─────────┼────────────────┼────────────────┼─────────────────────┘
          │                │                │
          └────────────────┼────────────────┘
                           │ Flow Records
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   netobserv namespace                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                 FlowCollector                              │  │
│  │  - eBPF-based packet capture                              │  │
│  │  - DNS tracking, RTT, packet drops                        │  │
│  └─────────────────────────┬─────────────────────────────────┘  │
│                             │                                    │
│                             ▼                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    LokiStack                               │  │
│  │  - Flow log storage                                       │  │
│  │  - Query and analysis                                     │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│         OpenShift Console → Observe → Network Traffic            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  - Traffic flows table                                     │  │
│  │  - Network topology visualization                          │  │
│  │  - Dropped packets analysis                                │  │
│  │  - DNS query tracking                                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Features

| Feature | Description |
|---------|-------------|
| **eBPF-based Collection** | Lightweight, kernel-level packet capture |
| **DNS Tracking** | Track DNS queries and responses |
| **RTT Measurement** | Round-trip time for network connections |
| **Packet Drop Detection** | Identify dropped packets and reasons |
| **Network Policy Visibility** | See effect of NetworkPolicy enforcement |
| **Topology View** | Visual representation of network connections |

## Prerequisites

- OpenShift 4.14+
- Cluster admin access

## Quick Start

### 1. Install Operators

```bash
# Create namespace and install Loki Operator
oc apply -f resources/namespace.yaml
oc apply -f resources/loki-operator-subscription.yaml

# Wait for Loki Operator
until oc get csv -n openshift-operators-redhat | grep -q "loki.*Succeeded"; do
  echo "Waiting for Loki Operator..."
  sleep 10
done

# Install Network Observability Operator
oc apply -f resources/netobserv-operator-subscription.yaml

# Wait for Network Observability Operator
until oc get csv -n openshift-netobserv-operator | grep -q "Succeeded"; do
  echo "Waiting for Network Observability Operator..."
  sleep 10
done
```

### 2. Deploy MinIO Storage

```bash
oc apply -f resources/minio.yaml
oc wait --for=condition=Available deployment/minio -n netobserv --timeout=120s
oc exec -n netobserv deploy/minio -- mkdir -p /data/netobserv
```

### 3. Deploy LokiStack

```bash
oc apply -f resources/lokistack.yaml

# Wait for LokiStack
until oc get lokistack netobserv-loki -n netobserv -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q "True"; do
  echo "Waiting for LokiStack..."
  sleep 10
done
```

### 4. Deploy FlowCollector

```bash
oc apply -f resources/flowcollector.yaml
```

### 5. (Optional) Deploy Sample App with Network Policies

```bash
oc apply -k resources/sample-app/
```

## Sample Application

The included sample app demonstrates Network Policies in action:

```
┌─────────────────────────────────────────────────────────────┐
│                  netpolicy-demo namespace                    │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Frontend   │───▶│   Backend    │───▶│   Database   │  │
│  │   (nginx)    │    │   (nginx)    │    │   (nginx)    │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         ▲                   ▲                   ▲           │
│         │                   │                   │           │
│    ✅ ALLOWED          ❌ BLOCKED          ❌ BLOCKED       │
│         │                   │                   │           │
│  ┌──────────────┐    ┌─────────────────────────────────┐   │
│  │   Traffic    │    │         Attacker Pod             │   │
│  │  Generator   │    │   (tests NetworkPolicy blocks)   │   │
│  └──────────────┘    └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Network Policies

| Policy | Effect |
|--------|--------|
| `default-deny-ingress` | Block all ingress by default |
| `allow-frontend-ingress` | Allow traffic to frontend |
| `allow-backend-from-frontend` | Only frontend can reach backend |
| `allow-database-from-backend` | Only backend can reach database |

### Verify Network Policies

```bash
# Traffic generator → Frontend (ALLOWED)
oc exec -n netpolicy-demo deploy/traffic-generator -- curl -s http://frontend:8080

# Attacker → Backend (BLOCKED)
oc exec -n netpolicy-demo deploy/attacker -- curl -s --connect-timeout 3 http://backend:8080
# Should timeout/fail

# Attacker → Database (BLOCKED)
oc exec -n netpolicy-demo deploy/attacker -- curl -s --connect-timeout 3 http://database:8080
# Should timeout/fail
```

## Viewing in Console

1. Navigate to **Observe → Network Traffic**
2. Use the **Traffic flows** tab to see individual flows
3. Use the **Topology** tab to visualize connections
4. Filter by:
   - Namespace
   - Pod name
   - Direction (ingress/egress)
   - Protocol
   - Dropped packets (to see NetworkPolicy blocks)

## Directory Structure

```
resources/
├── namespace.yaml                  # netobserv namespace
├── loki-operator-subscription.yaml # Loki Operator
├── netobserv-operator-subscription.yaml # NetObserv Operator
├── minio.yaml                      # MinIO for Loki storage
├── lokistack.yaml                  # LokiStack for flow logs
├── flowcollector.yaml              # FlowCollector config
├── kustomization.yaml
└── sample-app/                     # Network Policy demo
    ├── namespace.yaml
    ├── frontend.yaml
    ├── backend.yaml
    ├── database.yaml
    ├── network-policies.yaml
    ├── traffic-generator.yaml
    ├── attacker.yaml
    └── kustomization.yaml
```

## FlowCollector Features Enabled

```yaml
spec:
  agent:
    ebpf:
      features:
      - DNSTracking      # Track DNS queries
      - FlowRTT          # Measure round-trip time
      - PacketDrop       # Detect dropped packets
```

## Documentation

- [Network Observability Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/network_observability/index)
- [Network Policy Documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

## License

Apache License 2.0

