# Single Cluster Install

## Env Vars
```
export GLOO_GATEWAY_LICENSE_KEY=
export AGENTGATEWAY_LICENSE_KEY=
export ANTHROPIC_API_KEY=
```

## Ambient Config
```
export ISTIO_VERSION=1.27.0
export ISTIO_IMAGE=${ISTIO_VERSION}-solo
export REPO_KEY=d11c80c0c3fc
export REPO=us-docker.pkg.dev/gloo-mesh/istio-${REPO_KEY}
export HELM_REPO=us-docker.pkg.dev/gloo-mesh/istio-helm-${REPO_KEY}
export SOLO_ISTIO_LICENSE_KEY=
```

```
OS=$(uname | tr '[:upper:]' '[:lower:]' | sed -E 's/darwin/osx/')
ARCH=$(uname -m | sed -E 's/aarch/arm/; s/x86_64/amd64/; s/armv7l/armv7/')

mkdir -p ~/.istioctl/bin
curl -sSL https://storage.googleapis.com/istio-binaries-$REPO_KEY/1.27.1-solo/istioctl-1.27.1-solo-$OS-$ARCH.tar.gz | tar xzf - -C ~/.istioctl/bin
chmod +x ~/.istioctl/bin/istioctl

export PATH=${HOME}/.istioctl/bin:${PATH}
istioctl version --remote=false
```

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml 
```

```
helm upgrade --install istio-base oci://${HELM_REPO}/base \
--namespace istio-system \
--create-namespace \
--version ${ISTIO_IMAGE} \
-f - <<EOF
defaultRevision: ""
profile: ambient
EOF
```

```
helm upgrade --install istiod oci://${HELM_REPO}/istiod \
--namespace istio-system \
--version ${ISTIO_IMAGE} \
-f - <<EOF
global:
  hub: ${REPO}
  proxy:
    clusterDomain: cluster.local
  tag: ${ISTIO_IMAGE}
meshConfig:
  accessLogFile: /dev/stdout
  defaultConfig:
    proxyMetadata:
      ISTIO_META_DNS_AUTO_ALLOCATE: "true"
      ISTIO_META_DNS_CAPTURE: "true"
env:
  PILOT_ENABLE_IP_AUTOALLOCATE: "true"
  PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
pilot:
  cni:
    namespace: istio-system
    enabled: true
profile: ambient
license:
  value: ${SOLO_LICENSE_KEY}
  # Uncomment if you prefer to specify your license secret
  # instead of an inline value.
  # secretRef:
  #   name:
  #   namespace:
EOF
```

```
helm upgrade --install istio-cni oci://${HELM_REPO}/cni \
--namespace istio-system \
--version ${ISTIO_IMAGE} \
-f - <<EOF
ambient:
  dnsCapture: true
excludeNamespaces:
  - istio-system
  - kube-system
global:
  hub: ${REPO}
  tag: ${ISTIO_IMAGE}
  platform: gke
profile: ambient
cni:
  priorityClassName: ""
EOF

# Patch the DaemonSet to remove priority class (workaround for chart not respecting the value) if on GKE
kubectl patch daemonset istio-cni-node -n istio-system -p '{"spec":{"template":{"spec":{"priorityClassName":null}}}}'
```

```
helm upgrade --install ztunnel oci://${HELM_REPO}/ztunnel \
--namespace istio-system \
--version ${ISTIO_IMAGE} \
-f - <<EOF
configValidation: true
enabled: true
env:
  L7_ENABLED: "true"
hub: ${REPO}
istioNamespace: istio-system
namespace: istio-system
profile: ambient
proxy:
  clusterDomain: cluster.local
tag: ${ISTIO_IMAGE}
terminationGracePeriodSeconds: 29
variant: distroless
priorityClassName: ""
EOF

# Patch the DaemonSet to remove priority class (workaround for chart not respecting the value) if on GKE
kubectl patch daemonset ztunnel -n istio-system -p '{"spec":{"template":{"spec":{"priorityClassName":null}}}}'
```

```
kubectl get pods -n istio-system 
```

## Install Gloo Gateway

```
helm upgrade -i agentgateway-crds oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/enterprise-agentgateway-crds \
  --create-namespace \
  --namespace gloo-system \
  --version 2.1.0-beta.2
```

```
helm upgrade -i agentgateway oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/enterprise-agentgateway \
  -n gloo-system \
  --version 2.1.0-beta.2 \
  --set agentgateway.enabled=true \
  --set licensing.licenseKey=${AGENTGATEWAY_LICENSE_KEY} \
  --set licensing.glooGatewayLicenseKey=${GLOO_GATEWAY_LICENSE_KEY}
```

```
kubectl get pods -n gloo-system
```

## Install Kagent

```
kubectl create namespace kagent
kubectl label namespaces kagent istio.io/dataplane-mode=ambient
```

## Azure Entra ID 

```
export AZURE_ENTRA_ID_CLIENT_SECRET="_I6gY3AyzgjpR_crRbdf"
export AZURE_ENTRA_ID_CLIENT_ID="bf96-494b-c04376f66e0a"
export AZURE_ENTRA_ID_TENANT_ID="2205-4189-77519ff5064f"
export AZURE_ENTRA_ID_FRONTEND_CLIENT_ID="61a1-41ba-92017db1bafd"
export AZURE_ENTRA_ID_ISSUER="https://login.microsoftonline.com/8635e970-3333-2222-444-77519ff5064f/v2.0"
export KAGENT_ENT_VERSION=0.1.10-2026-01-06-main-ba15b13f
```

## Version of KAGENT

```
export KAGENT_ENT_VERSION=0.1.10-2026-01-06-main-ba15b13f
```

## Azure AD Groups
```
   e8f557b6-dd93-4df1-8206-70ecd738b274: "global.Admin"   
   8c867323-7002-4f90-bd03-00b90a655d5c: "global.Reader"   
   048b8653-3980-42e3-814b-dfe5f0499cc9: "global.Writer"   
```


## Install Kagent Management

```
helm upgrade -i kagent-mgmt \
oci://us-docker.pkg.dev/developers-369321/solo-enterprise-public-nonprod/charts/management \
-n kagent --create-namespace \
--version $KAGENT_ENT_VERSION \
-f - <<EOF
imagePullSecrets: []
cluster: kagent-ent-kind-entra-id
global:
  imagePullPolicy: IfNotPresent
oidc:
  issuer: https://login.microsoftonline.com/8635e970-2205-4189-bc77-77519ff5064f/v2.0
  additionalScopes:
    - api://d1e81942-bf96-494b-b2b9-c04376f66e0a/user_impersonation
rbac:
  roleMapping:
    roleMapper: "has(claims.Groups) ? claims.Groups.transformList(i, v, v in rolesMap, rolesMap[v]) : (has(claims.groups) ? claims.groups.transformList(i, v, v in rolesMap, rolesMap[v]) : [])"
    roleMappings:
      e8f557b6-dd93-2222-8206-70ecd738b274: "global.Admin"   
      8c867323-7002-2222-bd03-00b90a655d5c: "global.Reader"   
      048b8653-3980-2222-814b-dfe5f0499cc9: "global.Writer"   
service:
  type: LoadBalancer
  clusterIP: ""
ui:
  backend:
    oidc:
      clientId: ${AZURE_ENTRA_ID_CLIENT_ID}
      secret: ${AZURE_ENTRA_ID_CLIENT_SECRET}
  frontend:
    oidc:
      clientId: ${AZURE_ENTRA_ID_FRONTEND_CLIENT_ID}
clickhouse:
  enabled: true
tracing:
  verbose: true
EOF
```

```
# Generate the RSA key
openssl genrsa -out /tmp/key.pem 2048
kubectl create secret generic jwt -n kagent --from-file=jwt=/tmp/key.pem
```

```
helm upgrade -i kagent-crds \
oci://us-docker.pkg.dev/developers-369321/kagent-enterprise-public-nonprod/charts/kagent-enterprise-crds \
--version $KAGENT_ENT_VERSION \
-n kagent
```

```
helm upgrade -i kagent \
oci://us-docker.pkg.dev/developers-369321/kagent-enterprise-public-nonprod/charts/kagent-enterprise \
--version $KAGENT_ENT_VERSION \
-n kagent \
-f - <<EOF
oidc:
  clientId: ${AZURE_ENTRA_ID_CLIENT_ID}
  issuer: ${AZURE_ENTRA_ID_ISSUER}
  secretRef: kagent-backend-secret
  secret: ${AZURE_ENTRA_ID_CLIENT_SECRET}
rbac:
  roleMapping:
    roleMapper: "has(claims.Groups) ? claims.Groups.transformList(i, v, v in rolesMap, rolesMap[v]) : (has(claims.groups) ? claims.groups.transformList(i, v, v in rolesMap, rolesMap[v]) : [])"
    roleMappings:
      e8f557b6-dd93-2222-8206-70ecd738b274: "global.Admin"   
      8c867323-7002-2222-bd03-00b90a655d5c: "global.Reader"   
      048b8653-3980-2222-814b-dfe5f0499cc9: "global.Writer"  
providers:
  default: openAI
  openAI:
    apiKey: ${OPENAI_API_KEY}
otel:
  tracing:
    enabled: true
    exporter:
      otlp:
        endpoint: solo-enterprise-ui.kagent.svc.cluster.local:4317
        insecure: true
controller:
  image:
    repository: kagent-enterprise-public-nonprod
kmcp:
  enabled: true
  image:
    repository: kagent-enterprise-public-nonprod
EOF
```



## Access the UI
kubectl port-forward -n kagent svc/solo-enterprise-ui 4000:80

## Open http://localhost:4000 in your browser


```
kubectl get pods -n kagent
```
