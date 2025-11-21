# OpenShift AI Workshop - Initial Setup

## Prerequisites

Before starting, ensure you have the following operators installed:

- Node Discovery Feature Operator
- NVIDIA GPU Operator
- Red Hat Build of Leader Worker Set Operator
- Red Hat Build of Kueue Operator

## Step 1: Enable User Workload Monitoring

Enable UserWorkloadMonitoring to capture KServe metrics:

```bash
oc apply -f deploy/00-initial-config/cluster-monitoring-config.yaml
```

## Step 2: Enable GPU Support

Visit the repository for GPU installation instructions:

**Repository:** https://github.com/pkstaz/install-rhoai-workshop

```bash
git clone https://github.com/pkstaz/install-rhoai-workshop.git
cd install-rhoai-workshop
```

Follow the instructions in the repository to enable GPU support.

## Step 3: Install OpenShift AI Operator

1. **Install OpenShift AI Operator**
   - Navigate to Operator Hub in the OpenShift Console
   - Search for "OpenShift AI" or "Red Hat OpenShift AI"
   - Install the operator

2. **Create Data Science Cluster (DSC)**
   - After the operator is installed, create a default Data Science Cluster object
   - This can be done through the OpenShift Console or via CLI

## Step 4: Create Hardware Profile

Create the hardware profile for GPU-enabled model deployments:

```bash
oc apply -f deploy/00-initial-config/hardware-profile.yaml
```

**Note:** 
- The hardware profile uses the `infrastructure.opendatahub.io/v1` API version.
- If you see a warning about missing `kubectl.kubernetes.io/last-applied-configuration` annotation, this is expected and can be safely ignored. The annotation will be added automatically.

## Verification

Verify the hardware profile was created successfully:

```bash
oc get hardwareprofile -n redhat-ods-applications
```

You should see the `gpu-profile` listed.

