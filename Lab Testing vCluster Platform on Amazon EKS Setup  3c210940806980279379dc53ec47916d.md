# Lab Testing: vCluster Platform on Amazon EKS: Setup with Traefik and then Migration to Envoy Gateway

This runbook walks through provisioning an EKS cluster, installing vCluster Platform behind Traefik, creating a tenant cluster, verifying access, and then migrating the exposure layer from Traefik to Envoy Gateway (Gateway API).

## Variables used throughout

Set these once and reuse them in the commands below.

```bash
export CLUSTER_NAME="vcluster-traefik-lab"
export AWS_REGION="eu-north-1"
export AWS_ACCOUNT_ID="<your-account-id>"
```

---

## 1. Provision the EKS cluster

### 1.1 Create the cluster

```bash
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --version 1.33 \
  --nodegroup-name standard-nodes \
  --node-type t3.large \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3 \
  --managed \
  --with-oidc
```

**Purpose:** Creates the EKS control plane and a managed node group. `--with-oidc` associates an IAM OIDC provider with the cluster, which is required for IRSA (IAM Roles for Service Accounts), needed in the next step for the EBS CSI driver. Core add-ons (`vpc-cni`, `coredns`, `kube-proxy`, `metrics-server`) are installed automatically by `eksctl` when no config file is used.

### 1.2 Install the EBS CSI driver

Tenant clusters and vCluster Platform provision persistent volumes for their data stores. On EKS, dynamic EBS provisioning requires the EBS CSI driver, since the in-tree AWS EBS plugin is no longer available on recent Kubernetes versions.

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --role-name AmazonEKS_EBS_CSI_DriverRole_${CLUSTER_NAME} \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

**Purpose:** Creates the IAM role the EBS CSI driver will assume, scoped to the `ebs-csi-controller-sa` service account via IRSA.

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole_${CLUSTER_NAME} \
  --force
```

**Purpose:** Installs the EBS CSI driver as a managed EKS add-on, using the IAM role created above.

```bash
eksctl get addon --cluster $CLUSTER_NAME --region $AWS_REGION
```

**Purpose:** Confirms all add-ons (`vpc-cni`, `coredns`, `kube-proxy`, `metrics-server`, `aws-ebs-csi-driver`) show `ACTIVE` before proceeding.

### 1.3 Set a default StorageClass

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
parameters:
  type: gp3
EOF
```

**Purpose:** The cluster’s default `gp2` StorageClass uses the deprecated in-tree provisioner (`kubernetes.io/aws-ebs`), which does not work once the in-tree plugin is removed. This creates a new default StorageClass backed by the EBS CSI driver, so that any PersistentVolumeClaim without an explicit StorageClass provisions correctly.

```bash
kubectl get storageclass
```

**Purpose:** Confirms `gp3 (default)` is present with provisioner `ebs.csi.aws.com`.

---

## 2. Install Traefik

```bash
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
```

**Purpose:** Adds the official Traefik Helm repository.

```bash
helm install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --set service.type=LoadBalancer \
  --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb \
  --set providers.kubernetesCRD.enabled=true \
  --set providers.kubernetesIngress.enabled=true \
  --set ingressRoute.dashboard.enabled=true
```

**Purpose:** Installs Traefik with a `LoadBalancer`-type Service. The `aws-load-balancer-type: nlb` annotation is required, because on EKS, a plain `LoadBalancer` Service defaults to a Classic Load Balancer unless this annotation requests a Network Load Balancer. An NLB is needed here because it operates at Layer 4 and can pass TCP/TLS traffic through unmodified, which the next steps depend on.

```bash
kubectl get pods -n traefik
kubectl get svc -n traefik
```

**Purpose:** Confirms the Traefik pod is `Running` and retrieves the NLB’s external hostname (`EXTERNAL-IP` column). Note this hostname down, since it is used in the steps below.

---

## 3. Install vCluster Platform

```bash
vcluster platform start --values platform.yaml
```

**Purpose:** Deploys vCluster Platform via the vCluster CLI, using a custom values file (license token, feature flags, etc.). By default this also creates a `loft.host` tunnel for quick access. For a customer environment where exposure goes through the customer’s own ingress (Traefik), this tunnel is not the production access path, and it can be disabled on future installs with `--no-tunnel` if not needed.

```bash
kubectl get svc -n vcluster-platform
```

**Purpose:** Confirms the Platform’s core service (`loft`) is running, listening on port `443`. This is the backend that Traefik will route to.

### 3.1 Expose vCluster Platform through Traefik

vCluster Platform terminates its own TLS (it serves HTTPS on its own certificate). Because of this, the exposure must use **TLS passthrough**, meaning Traefik must forward the encrypted connection to the Platform pod without decrypting it itself. If Traefik tried to terminate TLS and forward plain HTTP, the Platform pod would reject it, since it only speaks HTTPS.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: vcluster-platform-tcp
  namespace: vcluster-platform
spec:
  entryPoints:
    - websecure
  routes:
    - match: HostSNI(\`*\`)
      services:
        - name: loft
          port: 443
  tls:
    passthrough: true
EOF
```

**Purpose:** `websecure` is Traefik’s built-in HTTPS entrypoint (port 443). `HostSNI(\`*`)`matches any TLS SNI hostname, used here because there is no dedicated domain in this environment. In production this should be scoped to the actual domain.`tls.passthrough: true`is the setting that prevents Traefik from decrypting the traffic, forwarding it unmodified to the`loft` service.

```bash
kubectl get ingressroutetcp -n vcluster-platform
```

**Purpose:** Confirms the route object was created successfully.

---

## 4. Create a tenant cluster

A tenant cluster (`traefik-test`) was created from the vCluster Platform UI (“Create Tenant Cluster” button, Shared Nodes tenancy model) to validate the platform end to end.

---

## 5. Verify access

### 5.1 Via curl

```bash
curl -k https://<TRAEFIK_NLB_HOSTNAME>/version
```

**Purpose:** Confirms the request reaches vCluster Platform through the full path, from client to NLB to Traefik (passthrough) to Platform. A successful response returns Platform version metadata (`"apiVersion":"version.loft.sh"`). The `-k` flag skips certificate validation, since the Platform serves a self-signed certificate in this environment (no custom domain configured).

### 5.2 Via browser

Navigate to:

```
https://<TRAEFIK_NLB_HOSTNAME>
```

**Purpose:** Confirms the full UI loads and login succeeds. A certificate warning is expected here (self-signed certificate) and can be bypassed for lab/testing purposes. For a production deployment with a real domain, this is resolved by issuing a trusted certificate for the Platform (e.g., via cert-manager) rather than relying on the self-signed one.

---

## 6. Migrate exposure from Traefik to Envoy Gateway

This section installs Envoy Gateway **in parallel** with Traefik, using a separate namespace and a separate load balancer, so the existing Traefik path keeps serving traffic uninterrupted while the new path is validated. Cutover (DNS change, decommissioning Traefik) is a separate decision once the new path is confirmed working.

### 6.1 Install Envoy Gateway and Gateway API CRDs

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.9.0 \
  -n envoy-gateway-system --create-namespace
```

**Purpose:** This single Helm chart installs both the Gateway API CRDs (the schema for `Gateway`, `GatewayClass`, `TLSRoute`, etc.) and the Envoy Gateway controller (the component that watches those objects and provisions actual Envoy Proxy instances).

```bash
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

**Purpose:** Waits for the controller to be ready before creating any Gateway API objects.

### 6.2 Configure NLB provisioning

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: nlb-envoy-proxy
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
EOF
```

**Purpose:** Defines the Kubernetes Service annotations to apply to the Envoy Proxy’s Service when it’s created. Same reasoning as with Traefik applies here: without this, EKS provisions a Classic Load Balancer by default, which does not support the TLS passthrough this setup needs.

### 6.3 Create the GatewayClass

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: nlb-envoy-proxy
    namespace: envoy-gateway-system
EOF
```

**Purpose:** Registers a `GatewayClass` named `eg`, tied to the Envoy Gateway controller, and referencing the `EnvoyProxy` configuration created above (so any `Gateway` using this class gets the NLB annotation).

```bash
kubectl get gatewayclass eg
```

**Purpose:** Confirms `ACCEPTED: True`.

### 6.4 Create the Gateway (TLS passthrough listener)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: vcluster-platform-gw
  namespace: vcluster-platform
spec:
  gatewayClassName: eg
  listeners:
    - name: tls-passthrough
      protocol: TLS
      port: 443
      tls:
        mode: Passthrough
      allowedRoutes:
        kinds:
          - kind: TLSRoute
EOF
```

**Purpose:** Creating this object is what actually triggers the Envoy Gateway controller to provision a real Envoy Proxy pod and its Service (a new, independent Network Load Balancer). `protocol: TLS` with `tls.mode: Passthrough` is the Gateway API equivalent of the `IngressRouteTCP` passthrough setting used with Traefik, required for the same reason (vCluster Platform terminates its own TLS).

```bash
kubectl get gateway vcluster-platform-gw -n vcluster-platform
```

**Purpose:** Confirms `PROGRAMMED: True` and retrieves the new NLB’s hostname under `ADDRESS`.

### 6.5 Create the TLSRoute

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: TLSRoute
metadata:
  name: vcluster-platform-tlsroute
  namespace: vcluster-platform
spec:
  parentRefs:
    - name: vcluster-platform-gw
      sectionName: tls-passthrough
  hostnames:
    - "<NEW_NLB_HOSTNAME>"
  rules:
    - backendRefs:
        - name: loft
          port: 443
EOF
```

**Purpose:** Routes traffic arriving at the Gateway’s `tls-passthrough` listener to the `loft` Service. Unlike Traefik’s `IngressRouteTCP`, the Gateway API `TLSRoute` **requires** a specific hostname value (matched against the TLS SNI); a bare wildcard (`*`) is not accepted. In the absence of a dedicated domain, the NLB’s own auto-generated hostname is used here, since that is what a client’s TLS request will present as SNI when connecting directly to it. In production, this should be the customer’s real domain.

```bash
kubectl get tlsroute -n vcluster-platform
```

**Purpose:** Confirms the route was created.

### 6.6 Verify the new path

```bash
curl -k https://<NEW_NLB_HOSTNAME>/version
```

**Purpose:** Confirms the new path (client → new NLB → Envoy Gateway, passthrough → vCluster Platform) works end to end, independently of the existing Traefik path.

---