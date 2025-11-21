# Workshop 03: Expose Model using Model as a Service (MaaS)

## Prerequisites

Before starting this workshop, ensure you have:

- Model as a Service (MaaS) enabled in your OpenShift AI cluster
- A deployed model using LLM-D (from Workshop 02)
- Access to the OpenShift AI Console
- Project namespace with appropriate permissions

## Set Environment Variables

Before starting, set the following environment variables to simplify the commands throughout this workshop:

```bash
# Set your project namespace
export PROJECT_NAME="ai-shared-project"  # Replace with your actual project name

# Set your cluster domain (automatically retrieved from cluster)
export CLUSTER_DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')

# Set your model service name (from your deployed model)
export SERVICE_NAME="gpt-oss-20b"        # Replace with your actual service name
```

**Verify your environment variables are set correctly:**

```bash
echo "PROJECT_NAME: $PROJECT_NAME"
echo "CLUSTER_DOMAIN: $CLUSTER_DOMAIN"
echo "SERVICE_NAME: $SERVICE_NAME"
```

## Step 1: Create GatewayClass

Create the GatewayClass `openshift-default` that will be used for the MaaS gateway:

```bash
oc apply -f deploy/03-maas/gateway-class.yaml
```

Verify the GatewayClass was created:

```bash
oc get gatewayclass openshift-default
```

## Step 2: Create MaaS API Namespace

Create the namespace for MaaS API components:

```bash
oc create namespace maas-api
```

Verify the namespace was created:

```bash
oc get namespace maas-api
```

## Step 3: Deploy MaaS API Objects

Deploy the MaaS API components using the official deployment overlay:

```bash
export CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')

oc apply --server-side=true \
  -f <(kustomize build "https://github.com/opendatahub-io/maas-billing.git/deployment/overlays/openshift?ref=main" | \
       envsubst '$CLUSTER_DOMAIN')
```

**Note:** This command uses kustomize to build the deployment manifests from the official MaaS billing repository. The source for installation instructions can be found at: https://opendatahub-io.github.io/maas-billing/latest/quickstart/

Verify the MaaS API objects were deployed:

```bash
oc get all -n maas-api
```

Wait for all pods to be in `Running` state:

```bash
oc get pods -n maas-api -w
```

Press `Ctrl+C` once all pods are running.

## Step 4: Configure Gateway AuthPolicy

The `gateway-auth-policy` from Workshop 02 is applied to the Gateway used for LLM-D models. We need to ensure it doesn't interfere with MaaS routes. Update the `gateway-auth-policy` to only apply to the LLM-D Gateway:

```bash
oc patch authpolicy gateway-auth-policy -n openshift-ingress --type=json -p='
[
  {
    "op": "replace",
    "path": "/spec/targetRef/kind",
    "value": "Gateway"
  },
  {
    "op": "replace",
    "path": "/spec/targetRef/name",
    "value": "openshift-ai-inference"
  }
]'
```

**Note:** This ensures the `gateway-auth-policy` only applies to the `openshift-ai-inference` Gateway used for LLM-D models, and not to the `maas-default-gateway` used by MaaS.

Verify the change:

```bash
oc get authpolicy gateway-auth-policy -n openshift-ingress -o jsonpath='{.spec.targetRef}' | jq .
```

## Step 5: Adjust Audience Policy for maas-api-auth-policy

Adjust the AuthPolicy to include the correct Kubernetes audience:

1. **Get the Kubernetes audience:**

   ```bash
   AUD="$(oc create token default --duration=10m 2>/dev/null | cut -d. -f2 | base64 -d 2>/dev/null | jq -r '.aud[0]' 2>/dev/null)"
   echo $AUD
   ```

   The output should be: `https://kubernetes.default.svc`

2. **Patch the AuthPolicy with the audience:**

   ```bash
   oc patch authpolicy maas-api-auth-policy -n maas-api --type=merge --patch-file <(echo "
   spec:
     rules:
       authentication:
         openshift-identities:
           kubernetesTokenReview:
             audiences:
               - $AUD
               - maas-default-gateway-sa")
   ```

3. **Verify the AuthPolicy was updated:**

   ```bash
   oc get authpolicy maas-api-auth-policy -n maas-api -o yaml | grep -A 5 audiences
   ```

## Step 6: Restart Required Pods

Restart the following pods to ensure they pick up the new configuration:

1. **Restart odh-model-controller in redhat-ods-applications:**

   ```bash
   oc rollout restart deployment/odh-model-controller -n redhat-ods-applications
   oc rollout status deployment/odh-model-controller -n redhat-ods-applications --timeout=120s
   ```

2. **Restart kuadrant-operator-controller-manager in kuadrant-system:**

   ```bash
   oc rollout restart deployment/kuadrant-operator-controller-manager -n kuadrant-system
   oc rollout status deployment/kuadrant-operator-controller-manager -n kuadrant-system --timeout=120s
   ```

Verify both pods are running:

```bash
oc get pods -n redhat-ods-applications | grep odh-model-controller
oc get pods -n kuadrant-system | grep kuadrant-operator-controller-manager
```

## Step 7: Test the Configuration

Test the MaaS API configuration by obtaining an authentication token:

1. **Set the MaaS API host:**

   ```bash
   CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
   HOST="https://maas.${CLUSTER_DOMAIN}"
   ```

2. **Request a token from the MaaS API:**

   ```bash
   TOKEN_RESPONSE=$(curl -sSk \
     -H "Authorization: Bearer $(oc whoami -t)" \
     -H "Content-Type: application/json" \
     -X POST \
     -d '{"expiration": "10m"}' \
     "${HOST}/maas-api/v1/tokens")
   ```

   **Note:** If you encounter a "Could not resolve host" error, this is a DNS resolution issue from your local machine. You can resolve it by:

   - **Option 1:** Add an entry to `/etc/hosts` (replace with the actual IP from your cluster):
     ```bash
     # Get the Gateway IP
     GATEWAY_IP=$(oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.status.addresses[0].value}' | nslookup | grep -A 1 "Name:" | tail -1 | awk '{print $2}')
     echo "$GATEWAY_IP maas.${CLUSTER_DOMAIN}" | sudo tee -a /etc/hosts
     ```

   - **Option 2:** Use the IP directly with the Host header:
     ```bash
     GATEWAY_IP=$(oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.status.addresses[0].value}' | nslookup | grep -A 1 "Name:" | tail -1 | awk '{print $2}')
     TOKEN_RESPONSE=$(curl -sSk \
       -H "Host: maas.${CLUSTER_DOMAIN}" \
       -H "Authorization: Bearer $(oc whoami -t)" \
       -H "Content-Type: application/json" \
       -X POST \
       -d '{"expiration": "10m"}' \
       "https://${GATEWAY_IP}/maas-api/v1/tokens")
     ```

   - **Option 3:** Test from within the cluster (this always works):
     ```bash
     oc run test-maas-token --image=curlimages/curl:latest --rm -i --restart=Never -- \
       curl -sSk -H "Authorization: Bearer $(oc create token default --duration=10m 2>/dev/null)" \
       -H "Content-Type: application/json" -X POST -d '{"expiration": "10m"}' \
       "https://maas.${CLUSTER_DOMAIN}/maas-api/v1/tokens"
     ```

3. **Extract the token from the response:**

   ```bash
   TOKEN=$(echo $TOKEN_RESPONSE | jq -r .token)
   echo $TOKEN
   ```

   **Note:** Save this token value, as you will need it to authenticate requests to your MaaS endpoints.

   **Important Notes:**
   - The URL may be displayed as "http" in some interfaces, but "https" access works correctly.
   - All tokens you create are active (there's currently no way to revoke them).
   - When you enable MaaS for a model served using LLM-D, the direct HTTPRoute to the model stays valid. This means:
     - `https://maas.<domain>/maas-api/v1/models` leads to the MaaS Gateway Pod.
     - `https://maas.<domain>/<namespace>/<model-id>/v1/models` leads to the LLM-D instance directly, **bypassing MaaS**.
   - **Security Warning:** If you checked the MaaS checkbox but did not check "Require authentication", your endpoint is freely accessible, bypassing any Authentication Policies you may have in place on MaaS.

4. **Verify the token was obtained successfully:**

   ```bash
   if [ -n "$TOKEN" ] && [ "$TOKEN" != "null" ]; then
     echo "Token obtained successfully"
   else
     echo "Error: Failed to obtain token"
     echo "Response: $TOKEN_RESPONSE"
   fi
   ```


