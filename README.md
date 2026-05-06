## TicketBlaster Live – GitOps installation (OpenShift + ArgoCD)

This repository contains the **GitOps manifests** to install and operate the TicketBlaster Live demo on OpenShift using **OpenShift GitOps (ArgoCD)**.

The demo showcases a **distributed SAGA (choreography)** implemented with **Camel Quarkus** microservices and **Kafka** eventing, secured by **Keycloak**, and includes the **Red Hat Streams for Apache Kafka Console** for event visualization.

## What gets installed

- **Namespaces** for the platform and demo workloads
- **Operators** (via OLM Subscriptions), then **instances** (CRs), in a safe order using sync-waves
- **Kafka (AMQ Streams)** cluster + SAGA topics
- **Keycloak (Red Hat build of Keycloak)** realm import (`ticketblaster`) + demo users/clients
- **Knative Serving** for the Order service (so it can run continuously and consume Kafka events)
- **Kafka UIs**:
  - Red Hat Streams for Apache Kafka Console (official)
  - Kafka UI (optional / helper)
- **Applications** (Order, Allocation, Payment, Frontend) managed as ArgoCD Applications (via an ApplicationSet)

## Install (bootstrap)

1) Make sure you’re logged into the intended OpenShift cluster context.

2) Apply the root ArgoCD Application:

```bash
oc apply -f argo-apps/ticketblaster-bootstrap.yaml -n openshift-gitops
```

3) Wait until ArgoCD reports everything as `Synced` / `Healthy`.

## Where to look in the repo

- **Bootstrap**: `bootstrap/`
  - `bootstrap/namespaces.yaml`: all namespaces (wave -1)
  - `bootstrap/kustomization.yaml`: the ordered list of operator + instance applications + the appset
- **AppSet (microservices)**: `bootstrap/appsets/demo-apps.yaml`
- **Kafka cluster + topics**: `apps/kafka/cluster/`
- **Keycloak**: `apps/keycloak/`
- **Red Hat Streams for Apache Kafka Console**: `apps/amq-streams-console/`
- **Frontend overlay (dev routes)**: `apps/frontend/overlays/dev/`

Optional HA blueprints (not part of the default bootstrap):

- `apps/datagrid-ha/` (Infinispan XSite blueprint)
- `apps/keycloak-ha/` (Keycloak multi-site blueprint)

## Accessing the demo

Routes are cluster-specific. List them with:

```bash
oc get route -n frontend
oc get route -n keycloak
oc get route -n allocation
oc get route -n kafka-console
```

The Order service is exposed through the Knative ingress; the hostname is the usual `order-order...` route host.

## Smoke tests (recommended)

End-to-end validation should cover:

- Keycloak login + `/users/me`
- Happy path order creation
- **Seat conflict**: two users choose the same seat → second order `FAILED` + budget refunded
- **Payment failure**: `forceFailPayment=true` → order `FAILED` + budget refunded + seat released

## Kafka topics

The SAGA choreography uses:

- `order-events` (OrderCreated)
- `seat-events` (SeatReserved / SeatReserveFailed / SeatReleased)
- `payment-events` (PaymentCompleted / PaymentFailed)
- `compensation-events` (CompensateSeat)

Use the Kafka Console to inspect these topics and correlate by `orderId`/`sagaId`.

## Observability (logs, traces, metrics)

### Traces (Tempo)

The backend services export traces via OTLP to the in-cluster OpenTelemetry Collector, which forwards to Tempo.

- Get the Tempo (Jaeger Query) Route:

```bash
oc get route -n observability | egrep -i 'jaeger|tempo'
```

### Metrics (Prometheus)

Each backend service exposes Prometheus metrics at `/q/metrics`. This repo installs:

- a `*-metrics` `Service` that selects the Knative Service pods
- a `ServiceMonitor` that scrapes `/q/metrics`

Verify the resources exist:

```bash
oc get service,servicemonitor -n order | egrep -i 'metrics|order'
oc get service,servicemonitor -n allocation | egrep -i 'metrics|allocation'
oc get service,servicemonitor -n payment | egrep -i 'metrics|payment'
```

### PromQL examples

```promql
# Orders created by status (counter)
sum by (status) (increase(ticketblaster_saga_orders_total[5m]))

# Kafka events consumed per topic/eventType
sum by (topic, eventType) (rate(ticketblaster_kafka_events_consumed_total[5m]))
```

## Troubleshooting

- **Order timeouts / readiness failures**: check if the Knative revision is restarting (ExitCode 137 / OOM). The `apps/order/base/knative-service.yaml` resources are tuned to keep it stable.
- **Seat conflict stuck in PENDING**: verify Allocation is publishing `SeatReserveFailed` successfully (type-conversion issues in Kafka headers can break failure paths if not handled).
- **ArgoCD OutOfSync on namespaces**: OpenShift may add extra labels/annotations; `bootstrap/namespaces.yaml` uses `IgnoreExtraneous` where appropriate.

## Docs (HA blueprints)

The HA blueprint docs live under:

- `docs/failover-e2e-test.md`
- `docs/connectivity-link-dns-failover.md`
