# Node Capacity-Aware Admission for Kueue

A distributed systems project that enhances the Kubernetes-native Kueue scheduler by introducing node capacity-aware admission to prevent workloads that cannot fit on any cluster node from reserving ClusterQueue quota.

## Overview

Kueue is a Kubernetes-native job scheduler that admits workloads based on available ClusterQueue quota before they are scheduled onto cluster nodes.

In the baseline scheduler, workloads whose CPU or memory requests exceeded the allocatable capacity of every node could still be admitted if sufficient ClusterQueue quota was available. Although these workloads could never be scheduled, they continued to reserve ClusterQueue quota and prevented runnable workloads from being admitted.

This project introduces a node capacity-aware admission enhancement that evaluates whether a workload can fit on at least one node before reserving ClusterQueue quota. By filtering infeasible workloads early in the admission pipeline, the scheduler preserves quota for runnable workloads and avoids unnecessary Pod creation.

## Problem Statement

The baseline admission process considered only ClusterQueue quota during admission.

As a result:

- Workloads that could never fit on any node were still admitted.
- These workloads reserved ClusterQueue quota.
- Kubernetes repeatedly attempted to schedule the resulting Pods, producing `FailedScheduling` events.
- Runnable workloads that could have been scheduled were blocked from admission because ClusterQueue quota had already been reserved by infeasible workloads.

## Solution

The scheduler was enhanced to perform node capacity feasibility evaluation before reserving ClusterQueue quota.

For every workload, the scheduler:

1. Computes the CPU and memory requested by each PodSet.
2. Compares those requests against the allocatable resources of every cluster node.
3. Determines whether at least one node has sufficient allocatable CPU and memory to accommodate the workload.
4. Skips workloads that cannot fit on any node before ClusterQueue quota reservation.

This preserves ClusterQueue quota for runnable workloads while preventing unnecessary Pod creation.

## Scheduler flow diagram

<img width="1182" height="2260" alt="Architecture diagram drawio" src="https://github.com/user-attachments/assets/5bfce42d-9ffb-4082-8ea1-2a6f0a6d6205" />



## Implementation Highlights

- Added node capacity feasibility evaluation before ClusterQueue quota reservation.
- Compared PodSet CPU and memory requests against node allocatable resources.
- Skipped workloads that could not fit on any node during admission.
- Added unit tests covering:
  - Feasible workloads
  - CPU-infeasible workloads
  - Memory-infeasible workloads
  - Clusters with no available nodes
- Benchmarked the scheduler across multiple workload mixes containing runnable and infeasible workloads.

## Benchmark Results

### Runnable Workload Admissions

<p align="center">
  <img src="images/Runnable Workload Admissions.png" width="700">
</p>

### Infeasible Workload Admissions

<p align="center">
  <img src="images/Infeasible Workload Admissions.png" width="700">
</p>

### ClusterQueue Quota Reserved by Infeasible Workloads

<p align="center">
  <img src="images/ClusterQueue Quota Reserved by Infeasible Workloads.png" width="700">
</p>

### Runnable Workload Admission Rate

<p align="center">
  <img src="images/Runnable Workload Admission Rate.png" width="700">
</p>

### Key Results

- Increased runnable workload admission rates from **4–20%** to **100%** across all benchmark scenarios.
- Eliminated ClusterQueue quota reservation by workloads that could not fit on any node.
- Increased ClusterQueue quota available for runnable workloads from **4 CPU** to **60 CPU** in the representative workload mix (30 infeasible / 30 runnable).
- Avoided unnecessary Pod creation and the resulting scheduling attempts for workloads that exceeded the capacity of every node.

For the complete benchmark methodology, workload configuration, experimental setup, and detailed results, see:

**[`docs/benchmark.md`](docs/benchmark.md)**

## Repository Structure

```
.
├── README.md
├── docs
│   ├── benchmark.md
│  
├── images
│   └── benchmark-results.png
```

## Technologies

- Go
- Kubernetes
- Kueue
- Kind
- Docker
- Kubectl

## Based On

This project extends the open-source Kueue scheduler.

Original project:
https://github.com/kubernetes-sigs/kueue
