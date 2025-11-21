# Workshop 03: Expose Model using Model as a Service (MaaS)

## Prerequisites

Before starting this workshop, ensure you have:

- **OpenShift cluster** (4.19.9+) with `oc`/`kubectl` access
- **RHOAI 3.0+** or **ODH 3.0+** installed
- **RHCL 1.2+** installed (must be in `kuadrant-system` namespace)
- **Cluster admin** or equivalent permissions
- **Required tools:**
  - `oc` (OpenShift CLI)
  - `kubectl`
  - `jq`
  - `kustomize` (v5.7.0+)
- A deployed model using LLM-D (from Workshop 02)
- Access to the OpenShift AI Console
- Project namespace with appropriate permissions

**⚠️ IMPORTANT:** MaaS must be enabled in the `OdhDashboardConfig` custom resource. If MaaS is not available in the dashboard, you need to enable it first (see Step 0 below).

**Reference:** For more detailed installation instructions, refer to the [official MaaS documentation](https://opendatahub-io.github.io/maas-billing/latest/quickstart/).

## Set Environment Variables

Before starting, set the following environment variables to simplify the commands throughout this workshop:

```bash
# Set your project namespace
export PROJECT_NAME="ai-shared-project"  # Replace with your actual project name

# Set your cluster domain (automatically retrieved from cluster)
export CLUSTER_DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')

# Set your model service name (from your deployed model)
export SERVICE_NAME="llama-31-8b"        # Replace with your actual service name
```

**Verify your environment variables are set correctly:**

```bash
echo "PROJECT_NAME: $PROJECT_NAME"
echo "CLUSTER_DOMAIN: $CLUSTER_DOMAIN"
echo "SERVICE_NAME: $SERVICE_NAME"
```

## Step 0: Enable MaaS in Dashboard Configuration (if not already enabled)

If MaaS is not available in the OpenShift AI dashboard, you need to enable it in the `OdhDashboardConfig` custom resource:

1. **Edit the OdhDashboardConfig:**

   ```bash
   oc edit odhdashboardconfig odh-dashboard-config -n redhat-ods-applications
   ```

2. **Add or update the `modelAsService` field in the `spec.dashboardConfig` section:**

   ```yaml
   spec:
     dashboardConfig:
       modelAsService: true
   ```

3. **Restart the dashboard deployment to ensure the changes take effect:**

   ```bash
   oc rollout restart deployment/rhods-dashboard -n redhat-ods-applications
   oc rollout status deployment/rhods-dashboard -n redhat-ods-applications --timeout=120s
   ```

4. **Verify the dashboard is running:**

   ```bash
   oc get pods -n redhat-ods-applications | grep rhods-dashboard
   ```

**Note:** After enabling MaaS, refresh your browser and you should see the MaaS option available in the dashboard.

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

oc apply --server-side=true --force-conflicts \
  -f <(kustomize build "https://github.com/opendatahub-io/maas-billing.git/deployment/overlays/openshift?ref=main" | \
       envsubst '$CLUSTER_DOMAIN')
```

**Note:** This command uses kustomize to build the deployment manifests from the official MaaS billing repository. The source for installation instructions can be found at: https://opendatahub-io.github.io/maas-billing/latest/quickstart/

**⚠️ IMPORTANT:** The `--force-conflicts` flag is required because the MaaS deployment may try to modify the `gateway-auth-policy` that was configured in Workshop 02. This is expected and safe - the conflicts will be resolved by forcing the server-side apply.

**Alternative:** If you prefer an automated deployment, you can use the official deployment script from the [MaaS billing repository](https://github.com/opendatahub-io/maas-billing):
```bash
git clone https://github.com/opendatahub-io/maas-billing.git
cd maas-billing
./deployment/scripts/deploy-openshift.sh
```

### 3.1 Verify MaaS Deployment

The deployment creates several core resources. Verify they were created successfully:

1. **Check namespaces:**

   ```bash
   oc get ns | grep -E "maas-api|kuadrant|kserve|opendatahub"
   ```

2. **Check Gateway status:**

   ```bash
   oc get gateway -n openshift-ingress maas-default-gateway
   ```

   The Gateway should show `PROGRAMMED: True` when ready.

3. **Check HTTPRoutes:**

   ```bash
   oc get httproute maas-api-route -n maas-api
   ```

4. **Check policies:**

   ```bash
   oc get authpolicy -A | grep maas
   oc get tokenratelimitpolicy -A | grep maas
   oc get ratelimitpolicy -A | grep maas
   ```

5. **Check MaaS API pods and service:**

   ```bash
   oc get pods -n maas-api
   oc get svc -n maas-api
   ```

   Wait for all pods to be in `Running` state:

   ```bash
   oc get pods -n maas-api -w
   ```

   Press `Ctrl+C` once all pods are running.

6. **Check Kuadrant operators:**

   ```bash
   oc get pods -n kuadrant-system | grep -E "kuadrant|authorino|limitador"
   ```

## Step 4: Configure Gateway AuthPolicy

**⚠️ IMPORTANT:** After deploying MaaS in Step 3, the `gateway-auth-policy` may have been modified by the MaaS deployment. We need to ensure it only applies to the LLM-D Gateway (`openshift-ai-inference`) and not to the MaaS Gateway (`maas-default-gateway`).

The `gateway-auth-policy` from Workshop 02 is applied to the Gateway used for LLM-D models. Update the `gateway-auth-policy` to only apply to the LLM-D Gateway:

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

**Note:** If you see conflicts when applying the MaaS deployment in Step 3, this is expected. The MaaS deployment may try to modify the `gateway-auth-policy`, but we need to keep it pointing to `openshift-ai-inference` only. The `--force-conflicts` flag in Step 3 handles this, and this step (Step 4) ensures the final configuration is correct.

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

   - **Option 2:** Use the IP directly with the Host header (note: you must use `-k` flag to bypass SSL certificate validation since the certificate is issued for the hostname, not the IP):
     ```bash
     # Get the Gateway IP (you can also use the IPs from nslookup: 3.137.2.235 or 18.224.50.87)
     GATEWAY_IP=$(nslookup $(oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.status.addresses[0].value}') 2>/dev/null | grep -A 1 "Name:" | tail -1 | awk '{print $2}')
     TOKEN_RESPONSE=$(curl -sk \
       -H "Host: maas.${CLUSTER_DOMAIN}" \
       -H "Authorization: Bearer $(oc whoami -t)" \
       -H "Content-Type: application/json" \
       -X POST \
       -d '{"expiration": "10m"}' \
       "https://${GATEWAY_IP}/maas-api/v1/tokens")
     ```
     **Note:** The `-k` flag bypasses SSL certificate validation. This is safe for testing but not recommended for production.

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

## Step 8: Update Existing Models to Use MaaS Gateway (Optional)

If you have existing models deployed using LLM-D that you want to expose through MaaS, you need to update the `LLMInferenceService` to reference the `maas-default-gateway`.

**Note:** This step is only needed if you want to migrate existing models to use MaaS. New models deployed through the dashboard with MaaS enabled will automatically use the MaaS gateway.

### 8.1 Update LLMInferenceService Gateway Reference

Update your existing `LLMInferenceService` to use the `maas-default-gateway`:

```bash
oc patch llminferenceservice ${SERVICE_NAME} -n ${PROJECT_NAME} --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/gateway/refs/-",
    "value": {
      "name": "maas-default-gateway",
      "namespace": "openshift-ingress"
    }
  }
]'
```

**Alternative:** You can also edit the `LLMInferenceService` directly:

```bash
oc edit llminferenceservice ${SERVICE_NAME} -n ${PROJECT_NAME}
```

Add or update the `gateway.refs` section:

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: ${SERVICE_NAME}
spec:
  gateway:
    refs:
      - name: maas-default-gateway
        namespace: openshift-ingress
```

### 8.2 Verify Model is Accessible via MaaS

After updating the gateway reference, verify your model is accessible through the MaaS gateway:

```bash
# Test model endpoint through MaaS
curl -X GET "https://maas.${CLUSTER_DOMAIN}/${PROJECT_NAME}/${SERVICE_NAME}/v1/models" \
  -H "Authorization: Bearer ${TOKEN}"
```

**Note:** Replace `${TOKEN}` with the token obtained in Step 7.

## Verification Checklist

After completing all steps, verify your MaaS deployment:

- [ ] GatewayClass `openshift-default` created
- [ ] Namespace `maas-api` created
- [ ] MaaS API objects deployed
- [ ] Gateway `maas-default-gateway` shows `PROGRAMMED: True`
- [ ] HTTPRoute `maas-api-route` created
- [ ] AuthPolicy `maas-api-auth-policy` configured with correct audience
- [ ] `gateway-auth-policy` updated to only apply to `openshift-ai-inference` Gateway
- [ ] Required pods restarted (`odh-model-controller`, `kuadrant-operator-controller-manager`)
- [ ] MaaS API pods running in `maas-api` namespace
- [ ] Token obtained successfully from MaaS API
- [ ] (Optional) Existing models updated to use `maas-default-gateway`


