# Test Graph Demo

This directory contains files demonstrating the **testGraph** feature of ComponentGroup, which enables orchestration of IntegrationTestScenarios with defined execution order and dependencies.

## Overview

The testGraph allows you to:
- Define which tests run first (no dependencies)
- Define which tests run after other tests complete
- Control failure behavior (failFast vs continue-on-failure)
- Maximize parallelism while respecting dependencies

## Files in This Directory

### Infrastructure
- **namespace.yaml** — Kubernetes namespace for the demo
- **frontend-component.yaml** — Frontend service component (konflux-ci.dev)
- **backend-component.yaml** — Backend service component (konflux-ci.dev)
- **componentgroup.yaml** — ComponentGroup with testGraph definition
- **sample-build-pipelinerun.yaml** — Completed build PLR that triggers testing

### Test Pipeline
- **test-pipeline.yaml** — Tekton Task and Pipeline definitions
  - **run-test-task** — Single task that waits 10s then succeeds
  - **integration-test-pipeline** — Pipeline that executes the test task

## Test Graph Structure

The ComponentGroup defines this test execution order:

```
┌─────────────────────────────────────────────────────────┐
│              unit-tests (no dependencies)               │
│                                                         │
│  Starts at t=0, runs for 10s                           │
└─────────────────────────────────────────────────────────┘
         ↓                        ↓
    (failFast: true)      (failFast: false)
         ↓                        ↓
┌──────────────────┐  ┌──────────────────────────────────┐
│   e2e-test       │  │   security-scan                  │
│                  │  │                                  │
│ Starts at t=10s  │  │ Starts at t=10s                 │
│ Runs for 10s     │  │ Runs for 10s                    │
│ If fails: blocks │  │ If fails: continues             │
│ integration-test │  │                                  │
└──────────────────┘  └──────────────────────────────────┘
         ↓
    (failFast: false)
         ↓
┌──────────────────────────────────────────────────────────┐
│             integration-test                             │
│                                                         │
│  Starts at t=20s (after e2e-test)                       │
│  Runs for 10s                                           │
│  Doesn't block other tests if it fails                  │
└──────────────────────────────────────────────────────────┘

Total execution time: ~30 seconds
```

## ComponentGroup testGraph Definition

```yaml
spec:
  testGraph:
    # e2e-test depends on unit-tests
    e2e-test:
      - name: unit-tests
        failFast: true      # If unit-tests fails, skip e2e-test AND integration-test
    
    # integration-test depends on e2e-test
    integration-test:
      - name: e2e-test
        failFast: false     # If e2e-test fails, still run integration-test
    
    # security-scan depends on unit-tests (parallel with e2e-test)
    security-scan:
      - name: unit-tests
        failFast: false     # If unit-tests fails, still run security-scan
```

## How to Use

### 1. Create the namespace
```bash
kubectl apply -f namespace.yaml
```

### 2. Create the components
```bash
kubectl apply -f frontend-component.yaml
kubectl apply -f backend-component.yaml
```

### 3. Create the test infrastructure
```bash
kubectl apply -f test-pipeline.yaml
```

### 4. Create the ComponentGroup
```bash
kubectl apply -f componentgroup.yaml
```

### 5. Trigger test execution by creating a build PipelineRun
```bash
kubectl apply -f sample-build-pipelinerun.yaml
```

The integration service will:
1. Detect the completed build PipelineRun
2. Match it to the ComponentGroup
3. Create IntegrationTestScenario PipelineRuns for: unit-tests, e2e-test, security-scan, and integration-test
4. Execute them in the order defined by testGraph
5. Report results back to the ComponentGroup status

## Expected Behavior

When you watch the namespace:
```bash
kubectl get -n new-model-demo pipelinerun -w
```

You should see:
- **t=0**: unit-tests PipelineRun created and running
- **t=10s**: 
  - unit-tests completes (Succeeded)
  - e2e-test and security-scan PipelineRuns created
- **t=20s**: 
  - e2e-test and security-scan complete
  - integration-test PipelineRun created
- **t=30s**: 
  - integration-test completes

## Key Concepts

### failFast Behavior

**failFast: true**
- If the parent test fails, ALL dependent tests are skipped
- Useful for "blocking" tests like unit tests or smoke tests
- Saves execution time when critical tests fail
- **Important:** If a child has multiple parents and ANY parent with `failFast: true` fails, the child is cancelled regardless of other parents' status or finish order

**failFast: false**
- If the parent test fails, dependent tests still run
- Useful for gathering comprehensive test results
- Supports parallel test execution in different failure scenarios
- **Order-independent:** Whether the parent fails first or succeeds first doesn't matter; the child will run

### Parallel Execution

Tests with no dependencies run in parallel. In this example:
- `unit-tests`: runs alone (first)
- `e2e-test` and `security-scan`: run in parallel (both depend only on unit-tests)
- `integration-test`: runs alone (depends on e2e-test)

### Independent Branches

You can have multiple independent test branches:
```yaml
testGraph:
  smoke-tests: []           # Runs first
  unit-tests: []            # Runs first (parallel with smoke-tests)
  e2e-tests:
    - name: smoke-tests
  security-scan:
    - name: unit-tests
```

In this case:
- smoke-tests and unit-tests run in parallel at t=0
- e2e-tests and security-scan run in parallel at t=10 (each with their own dependency)

## Integration with ComponentGroup

The testGraph is defined at the ComponentGroup level, meaning:
- All tests for the ComponentGroup follow the same execution order
- Changes to testGraph apply to all future Snapshots
- Test results are tracked per Snapshot in the integration service

## Viewing Results

After tests complete, check the integration service for:
- **ComponentGroup Status** — Overall test results
- **Snapshot Status** — Per-snapshot test results
- **PipelineRun Status** — Individual test execution details

Example:
```bash
kubectl get componentgroup -n new-model-demo sample-app-group -o yaml
kubectl get snapshot -n new-model-demo -o yaml
kubectl get pipelinerun -n new-model-demo -o yaml
```
