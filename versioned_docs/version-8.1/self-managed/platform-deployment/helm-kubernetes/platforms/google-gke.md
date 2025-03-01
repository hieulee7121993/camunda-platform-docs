---
id: google-gke
title: "Google GKE"
description: "Deploy Camunda Platform 8 Self-Managed on Google GKE, a managed container service to run and scale Kubernetes applications in the cloud or on-premises."
---

Google Kubernetes Engine ([GKE](https://cloud.google.com/kubernetes-engine))
is a managed container service to run and scale Kubernetes applications in the cloud or on-premises.

Camunda Platform 8 Self-Managed can be deployed on GKE like any Kubernetes cluster using [Helm charts](../deploy.md). However, there are a few pitfalls to avoid as described below.

## GKE cluster specification

Generally speaking, the GKE cluster specification depends on your needs and workloads.
Here is a recommended start to run Camunda Platform 8:

- Instance type: `n1-standard-4` (4 vCPUs, 15 GB Memory)
- Number of nodes: `4`
- Volume type: `Performance (SSD) persistent disks`

## Pitfalls to avoid

For general deployment pitfalls, visit the [deployment troubleshooting guide](../../troubleshooting.md).

### Volume performance

To have a proper performance in Camunda Platform 8, the GKE cluster nodes should use volumes with around 1,000-3,000 IOPS. The `Performance (SSD) persistent disks` volumes deliver a consistent baseline IOPS performance but it [varies based on volume size](https://cloud.google.com/compute/docs/disks/performance#performance_factors).

It's recommended to use `Performance (SSD) persistent disks` volume type with at least `100 GB` per volume to have 3,000 IOPS.

### Zeebe Ingress

Zeebe requires an Ingress controller that supports `gRPC`, so if you are using [GKE Ingress](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress) (ingress-gce), not Nginx Ingress, you might need to do extra steps. Namely, using `cloud.google.com/app-protocols` annotation in Zeebe Service. For more details, visit the GKE guide [using HTTP/2 for load balancing with Ingress](https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-http2).
