# aks-fleet-manager-demo
Demo showing capabilities of AKS Fleet Manager (KubeFleet)

## Overview

This demo showcases the use of AKS Fleet Manager to deploy a sample application across multiple clusters with a controlled rollout strategy.

## Demo Application

The demo includes a simple application called "myapp" that runs a pause container in its own namespace. This application is used to demonstrate:

1. **ClusterResourcePlacement**: Distribution of resources across all clusters in the fleet
2. **Rollout Strategy**: Progressive deployment with timing and approval gates

## Prerequisites

- AKS Fleet Manager hub cluster already created
- Three member clusters labeled appropriately:
  - `dev` cluster with label `environment=dev`
  - `test` cluster with label `environment=test`
  - `prod` cluster with label `environment=prod`

## Manifests

### Application Manifests

- `manifests/namespace.yaml` - Creates the "myapp" namespace
- `manifests/deployment.yaml` - Deploys a pause container in the myapp namespace

### Fleet Manager Manifests

- `manifests/clusterresourceplacement.yaml` - ClusterResourcePlacement object that distributes the myapp namespace and its resources across all clusters
- `manifests/rollout-strategy.yaml` - Defines a staged rollout strategy with the following workflow:
  - **Stage 1 (dev)**: Deploy to dev cluster → Wait 1 hour
  - **Stage 2 (test)**: Deploy to test cluster → Wait 24 hours + Require approval
  - **Stage 3 (prod)**: Deploy to prod cluster
- `manifests/resourceoverride.yaml` - ResourceOverride object that applies environment-specific configuration:
  - **dev**: 1 replica, ENVIRONMENT=development
  - **test**: 2 replicas, ENVIRONMENT=testing
  - **prod**: 3 replicas, ENVIRONMENT=production

## Deployment Instructions

### Step 1: Label your clusters

Ensure your member clusters are properly labeled:

```bash
# Label dev cluster
kubectl label memberclusters <dev-cluster-name> environment=dev

# Label test cluster
kubectl label memberclusters <test-cluster-name> environment=test

# Label prod cluster
kubectl label memberclusters <prod-cluster-name> environment=prod
```

### Step 2: Deploy the application manifests to the hub

Apply the application manifests to your Fleet hub cluster:

```bash
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/deployment.yaml
```

### Step 3: Apply environment-specific configuration (optional)

Apply the ResourceOverride to customize configuration per environment:

```bash
kubectl apply -f manifests/resourceoverride.yaml
```

This will apply environment-specific overrides:
- Dev clusters will get 1 replica with ENVIRONMENT=development
- Test clusters will get 2 replicas with ENVIRONMENT=testing
- Prod clusters will get 3 replicas with ENVIRONMENT=production

### Step 4: Create ClusterResourcePlacement

Apply the ClusterResourcePlacement to distribute resources:

```bash
kubectl apply -f manifests/clusterresourceplacement.yaml
```

This will create a placement that selects the myapp namespace and uses a PickAll policy to target all available clusters.

### Step 5: Create rollout strategy (optional)

For a controlled progressive rollout, apply the staged update strategy:

```bash
kubectl apply -f manifests/rollout-strategy.yaml
```

This creates:
- A `ClusterStagedUpdateStrategy` defining the rollout stages
- A `ClusterStagedUpdateRun` that executes the rollout

## Monitoring the Deployment

### Check ClusterResourcePlacement status

```bash
kubectl get clusterresourceplacement myapp-placement -o yaml
```

### Check rollout progress

```bash
kubectl get clusterstagedupdaterun myapp-rollout -o yaml
```

### Verify deployment on member clusters

Switch context to each member cluster and verify:

```bash
kubectl get namespaces myapp
kubectl get deployments -n myapp
kubectl get pods -n myapp
```

## Approving the Rollout

After the test stage completes its 24-hour wait, the rollout will pause waiting for approval before proceeding to prod.

### Find the approval request

First, list all pending approval requests:

```bash
kubectl get clusterapprovalrequests
```

### Approve via CLI

Approve the request to allow the rollout to proceed to production:

```bash
kubectl patch clusterapprovalrequest <approval-request-name> --type=merge -p '{"spec":{"state":"Approved"}}'
```

Replace `<approval-request-name>` with the name from the previous command.

### Verify approval

Check the status of the rollout to confirm it has proceeded:

```bash
kubectl describe clusterstagedupdaterun myapp-rollout
```

## Cleanup

To remove the demo resources:

```bash
# Delete rollout strategy (if applied)
kubectl delete -f manifests/rollout-strategy.yaml

# Delete resource override (if applied)
kubectl delete -f manifests/resourceoverride.yaml

# Delete cluster resource placement
kubectl delete -f manifests/clusterresourceplacement.yaml

# Delete application manifests
kubectl delete -f manifests/deployment.yaml
kubectl delete -f manifests/namespace.yaml
```

## Architecture

```
Fleet Hub Cluster
├── myapp namespace
│   └── myapp deployment (pause container)
├── ClusterResourcePlacement
│   └── Distributes myapp namespace to all clusters
├── ResourceOverride
│   └── Applies environment-specific config (replicas, env vars)
└── ClusterStagedUpdateStrategy
    └── Controls rollout: dev → test → prod

Member Clusters (dev, test, prod)
└── Receive myapp namespace and deployment with environment-specific overrides
```

## Notes

- The pause container (`registry.k8s.io/pause:3.9`) is a minimal container that does nothing but sleep, making it ideal for demos
- ClusterResourcePlacement with `PickAll` policy ensures all clusters receive the resources
- ResourceOverride allows environment-specific configuration using JSON patch operations
- The rollout strategy provides a real-world example of progressive deployment with gates
- Approval gates require manual intervention to proceed to production
- **API Version Note**: The ClusterStagedUpdateStrategy and ResourceOverride use v1alpha1 APIs, indicating these features are in alpha and may change in future Fleet Manager releases
