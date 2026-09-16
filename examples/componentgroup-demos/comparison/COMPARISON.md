# Application vs ComponentGroup: Model Comparison

This document compares the legacy Application model with the new ComponentGroup model, highlighting key differences and improvements.

## High-Level Architecture

### Old Model: Application
```
Application (appstudio.redhat.com/v1alpha1)
    ├── Frontend Component (appstudio.redhat.com/v1alpha1)
    ├── Backend Component (appstudio.redhat.com/v1alpha1)
    └── [More Components...]
```
- **Single container resource**: Application groups components but doesn't track versions
- **Single version per component**: Each Component has one current state
- **Limited orchestration**: No built-in test ordering or dependencies
- **Scope**: AppStudio-specific (Hybrid Application Service)

### New Model: ComponentGroup
```
ComponentGroup (konflux-ci.dev/v1beta2)
    ├── Frontend Component Reference (version: main)
    ├── Backend Component Reference (version: v1)
    └── Nested ComponentGroup Reference
```
- **Version-aware**: Tracks multiple versions of components via ComponentVersion references
- **Flexible references**: Can reference Components, specific versions, or other ComponentGroups
- **Test orchestration**: Built-in test graph for defining test execution order and dependencies
- **Global Candidate List**: Tracks promoted versions across the entire group
- **Scope**: Konflux-native (universal CI/CD platform)

## Key Differences

| Aspect | Application (Old) | ComponentGroup (New) |
|--------|-------------------|---------------------|
| **API Version** | appstudio.redhat.com/v1alpha1 | konflux-ci.dev/v1beta2 |
| **Purpose** | Container for components | Orchestration of versioned components |
| **Component Versioning** | Single per component | Multiple versions per component |
| **Test Orchestration** | No built-in support | Via testGraph (DAG) |
| **Nested Groups** | Not supported | Yes, can reference other ComponentGroups |
| **Global Candidate List** | Not present | Yes, tracks promoted versions |
| **Use Case** | High-level app definition | Integration testing & release workflows |

## Structural Comparison

### Application (Old Model)

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: Application
metadata:
  name: sample-app
spec:
  displayName: "Sample Application"
  # Optional git repositories for app model and gitops
  appModelRepository:
    url: https://github.com/demo-org/sample-app-model
  gitOpsRepository:
    url: https://github.com/demo-org/sample-app-gitops
status:
  conditions: [...]
  devfile: "..."  # App is also a devfile container
```

**Child Components** reference the Application:
```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: Component
metadata:
  name: frontend-service
  labels:
    app.kubernetes.io/part-of: sample-app  # Reference to parent app
spec:
  componentName: frontend-service
  application: sample-app
  source:
    git:
      url: https://github.com/demo-org/frontend-service
      revision: main
```

### ComponentGroup (New Model)

```yaml
apiVersion: konflux-ci.dev/v1beta2
kind: ComponentGroup
metadata:
  name: sample-app-group
spec:
  # ComponentGroup explicitly references components and versions
  components:
    - name: frontend-service
      kind: component
      componentVersion:
        name: "main"              # Track specific versions
        revision: "main-branch"   # Fallback branch name
        context: "./"             # Context directory
    - name: backend-service
      kind: component
      componentVersion:
        name: "v1"
    - name: nested-group
      kind: componentGroup         # Can reference other ComponentGroups
  
  # Test graph defines execution order for integration tests
  testGraph:
    e2e-test:
      - name: unit-tests
        failFast: true             # Stop dependents if this fails
    integration-test:
      - name: e2e-test
        failFast: false            # Continue even if this fails
    security-scan:
      - name: unit-tests
      
status:
  # Global Candidate List: Latest promoted versions
  globalCandidateList:
    - name: frontend-service
      version: main
      url: https://github.com/demo-org/frontend-service
      lastPromotedImage: quay.io/demo-org/frontend-service@sha256:...
      lastPromotedCommit: 6a7c81802e785aa869f82301afe61f4e9775772b
      lastPromotedBuildTime: "2025-08-13T12:00:00Z"
    - name: backend-service
      version: "v1"
      url: https://github.com/demo-org/backend-service
      lastPromotedImage: quay.io/demo-org/backend-service@sha256:...
      lastPromotedCommit: 1359836353b8e249f2fbceba47d82751d7dab902
      lastPromotedBuildTime: "2025-08-13T12:30:00Z"
```

## Feature Comparison

### 1. Component Versioning

**Application (Old)**
- A Component can only have one "current" version
- Versioning managed externally (git branches, tags, etc.)
- No CRD-level version tracking

**ComponentGroup (New)**
- ComponentVersion references allow tracking multiple versions of a component
- Each version can have its own:
  - Branch (`revision`)
  - Context directory (`context`)
  - Promoted image in Global Candidate List
- Supports version selection at ComponentGroup creation time

### 2. Test Orchestration

**Application (Old)**
- No built-in test ordering
- All tests run in parallel against the application
- Dependencies managed outside the resource (docs, conventions, etc.)

**ComponentGroup (New)**
- `testGraph`: Declarative test dependency graph
- Tests are IntegrationTestScenario CRs
- Test execution follows DAG defined in spec:
  ```yaml
  testGraph:
    test-b:
      - name: test-a        # test-b depends on test-a
        failFast: true      # stop other dependents if test-a fails
    test-c:
      - name: test-a
      - name: test-b
  ```
- Supports failure handling strategies (failFast)

### 3. Scalability

**Application (Old)**
- Single flat structure
- All components at the same level
- Implicit ordering and relationships

**ComponentGroup (New)**
- Nested ComponentGroups possible (one ComponentGroup can reference another)
- Hierarchical organization of components
- Explicit relationships via references
- Global Candidate List enables complex release workflows

### 4. Integration Service Workflows

**Application (Old)**
- Watches Application and Component CRs
- Creates Snapshot for each build/change
- Tests run against the full Application

**ComponentGroup (New)**
- Watches ComponentGroup and Component CRs
- Creates Snapshot from Global Candidate List versions
- Orchestrates tests via testGraph
- Supports promotion-triggered Snapshot creation
- Better for multi-version and multi-team scenarios

## Migration Path

When migrating from Application to ComponentGroup:

1. **Create ComponentGroup** with same components as Application
2. **Add ComponentVersion references** pointing to component versions
3. **Define testGraph** if tests have ordering requirements
4. **Populate Global Candidate List** with currently promoted versions
5. **Deprecate Application** once ComponentGroup is managing tests/releases

## Use Cases

### Application (Old Model) Works Best For:
- Simple, single-version applications
- AppStudio-managed applications
- Team applications with straightforward testing
- Basic component grouping without complex test dependencies

### ComponentGroup (New Model) Works Best For:
- Multi-component systems with version matrix testing
- Complex test dependencies and ordering
- Release workflows requiring specific component versions
- Nested/hierarchical application structures
- Teams using Konflux as a universal CI/CD platform
- Applications requiring promotion-based snapshot creation

## Example: Test Graph Execution Order

Given this testGraph:
```yaml
testGraph:
  e2e-test:
    - name: unit-tests
  integration-test:
    - name: e2e-test
  security-scan:
    - name: unit-tests
```

**Execution timeline:**
```
Time 0: [unit-tests] starts
Time 5: [e2e-test] starts (after unit-tests finishes)
        [security-scan] starts (after unit-tests finishes)
Time 10: [integration-test] starts (after e2e-test finishes)
Time 15: All tests complete
```

The testGraph creates a clear execution sequence while maximizing parallelism.

## Global Candidate List Lifecycle

The Global Candidate List in ComponentGroup status tracks:
- **Latest promoted image**: Image digest or tag of latest successful build
- **Last promoted commit**: Git commit used for that image
- **Build timestamp**: When that image was built
- **Component URL**: Git repository for the component

This enables:
1. **Reproducible snapshots**: Create integration test snapshots from known-good versions
2. **Release workflows**: Know exactly which versions are ready for release
3. **Audit trails**: Track when and what was promoted
4. **Rollback capability**: Can reference any version in the GCL history
