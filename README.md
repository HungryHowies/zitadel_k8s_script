# zitadel_k8s_script

```
vi install.sh
```
Add to file
```
#!/usr/bin/env bash
set -euo pipefail

## openebs create directory
sudo mkdir -p /var/openebs/local
sudo chmod 777 /var/openebs/local

##  Home Directory for Zitadel
sudo mkdir -p /opt/zitadel-k8s
sudo chmod 755 /opt/zitadel-k8s

# Disable all interactive apt prompts & service restart questions
export DEBIAN_FRONTEND=noninteractive
export NEEDRESTART_MODE=a

# =========================
# Colors & helpers
# =========================
CYAN="\e[1;36m"
GREEN="\e[1;32m"
RED="\e[1;31m"
RESET="\e[0m"

step() { echo -e "\n${CYAN}==> $1${RESET}"; }
ok()   { echo -e "${GREEN}✔ $1${RESET}"; }
fail() { echo -e "${RED}✖ $1${RESET}"; exit 1; }

# =========================
# User input
# =========================
step "Collecting configuration"

read -rp "Timezone [America/Chicago]: " TIMEZONE
TIMEZONE=${TIMEZONE:-America/Chicago}

read -rp "Kind cluster name [zitadel-cluster]: " CLUSTER_NAME
CLUSTER_NAME=${CLUSTER_NAME:-zitadel-cluster}

read -rp "Zitadel domain [zitadel.local]: " ZITADEL_DOMAIN
ZITADEL_DOMAIN=${ZITADEL_DOMAIN:-zitadel.local}

read -rp "Host IP for /etc/hosts (example 192.168.1.210): " HOST_IP
[[ -z "$HOST_IP" ]] && fail "Host IP is required"

read -rp "Postgres password [zitadelpass]: " POSTGRES_PASS
POSTGRES_PASS=${POSTGRES_PASS:-zitadelpass}

# -------------------------
# Zitadel Admin Login Info
# -------------------------
ZITADEL_ADMIN_USER="zitadel-admin@zitadel.${ZITADEL_DOMAIN}"
ZITADEL_ADMIN_PASS="Password1!"

#--------------------
# Service prompt stop
#--------------------
INSTALL_DIR="/opt/zitadel-k8s"
mkdir -p /etc/needrestart
cat > /etc/needrestart/needrestart.conf <<'EOF'
$nrconf{restart} = 'a';
$nrconf{kernelhints} = -1;
EOF

# =========================
# Base system
# =========================
step "System update & base packages"
apt-get update
apt-get -y \
  -o Dpkg::Options::="--force-confdef" \
  -o Dpkg::Options::="--force-confold" \
  upgrade

timedatectl set-timezone "$TIMEZONE"

apt-get install -y \
  -o Dpkg::Options::="--force-confdef" \
  -o Dpkg::Options::="--force-confold" \
  net-tools plocate vim git sendmail curl jq nodejs \
  ca-certificates gnupg


mkdir -p "$INSTALL_DIR"
ok "Base system ready"

# =========================
# Docker
# =========================
step "Installing Docker"

install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" \
> /etc/apt/sources.list.d/docker.list

apt-get update
apt-get install -y \
  -o Dpkg::Options::="--force-confdef" \
  -o Dpkg::Options::="--force-confold" \
  docker-ce docker-ce-cli containerd.io

systemctl enable --now docker
usermod -aG docker "$SUDO_USER" || true
ok "Docker installed"

# =========================
# kubectl / kind / helm
# =========================
step "Installing kubectl, kind, helm"

curl -LO https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
install -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl

curl -Lo kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
install -m 0755 kind /usr/local/bin/kind && rm kind

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
ok "Kubernetes tooling installed"

# =========================
# KIND cluster (safe)
# =========================
step "Checking KIND cluster"

if kind get clusters | grep -qx "$CLUSTER_NAME"; then
  echo "Cluster '$CLUSTER_NAME' already exists:"
  echo "1) Reuse existing"
  echo "2) Delete and recreate"
  echo "3) Abort"
  read -rp "Choose [1-3]: " CHOICE

  case "$CHOICE" in
    1) ok "Reusing existing cluster" ;;
    2)
      kind delete cluster --name "$CLUSTER_NAME"
      kind create cluster --name "$CLUSTER_NAME"
      ;;
    3) fail "Aborted by user" ;;
    *) fail "Invalid choice" ;;
  esac
else
  kind create cluster --name "$CLUSTER_NAME"
fi

ok "KIND cluster ready"

# =========================
# Ingress + Nginx fixes
# =========================
step "Installing ingress-nginx & fixing NodePort + hostNetwork"

# Apply default manifests
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Patch Service to NodePort
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  -p '{"spec": {"type": "NodePort"}}'

# Patch Deployment to use hostNetwork
kubectl patch deployment ingress-nginx-controller -n ingress-nginx \
  --patch '{"spec": {"template": {"spec": {"hostNetwork": true, "dnsPolicy": "ClusterFirstWithHostNet"}}}}'

ok "Ingress-nginx patched with NodePort and hostNetwork"

# =========================
# OpenEBS + StorageClass
# =========================
step "Installing OpenEBS"

kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml

cat > "$INSTALL_DIR/csi-local-sc.yaml" <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-local
provisioner: openebs.io/local
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF

kubectl apply -f "$INSTALL_DIR/csi-local-sc.yaml"

kubectl patch ds openebs-ndm -n openebs --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/volumes/1/hostPath/type","value":"DirectoryOrCreate"}]'

kubectl delete pod -n openebs -l name=openebs-ndm || true
ok "Storage configured"

# =========================
# Postgres
# =========================
step "Deploying Postgres"

kubectl create namespace zitadel 2>/dev/null || true

cat > "$INSTALL_DIR/postgres.yaml" <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
   name: zitadel-postgres-data-csi
   namespace: zitadel
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
  storageClassName: csi-local
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zitadel-postgres-data-csi
  namespace: zitadel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zitadel-postgres
  template:
    metadata:
      labels:
        app: zitadel-postgres
    spec:
      securityContext:
        fsGroup: 999
        fsGroupChangePolicy: "OnRootMismatch"
      containers:
        - name: postgres
          image: postgres:14.10-alpine
          env:
            - name: POSTGRES_DB
              value: zitadel
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: ${POSTGRES_PASS}
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-data
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: zitadel-postgres-data-csi
---
apiVersion: v1
kind: Service
metadata:
   name: zitadel-postgres-csi
   namespace: zitadel
spec:
  selector:
    app: zitadel-postgres
  ports:
    - port: 5432
EOF

kubectl apply -f "$INSTALL_DIR/postgres.yaml"

# =========================
# Patch Postgres (TCP-safe) with progress
# =========================
step "Patching Postgres roles"

echo "Waiting for Postgres pod to be created..."
while true; do
    POSTGRES_POD=$(kubectl get pod -n zitadel -l app=zitadel-postgres \
        -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)
    if [[ -n "$POSTGRES_POD" ]]; then
        echo "Found Postgres pod: $POSTGRES_POD"
        break
    fi
    sleep 2
done

echo "Waiting for pod $POSTGRES_POD to be Ready..."
kubectl wait --for=condition=Ready pod/"$POSTGRES_POD" -n zitadel --timeout=3600s

echo "Waiting for Postgres to accept connections..."
while ! kubectl exec -n zitadel "$POSTGRES_POD" -- pg_isready -h localhost -U postgres >/dev/null 2>&1; do
    echo -n "."
    sleep 2
done
echo " Postgres is ready!"

kubectl exec -n zitadel "$POSTGRES_POD" -- \
  sh -c "psql -h localhost -U postgres -P pager=off -v ON_ERROR_STOP=1 <<'SQL'
DO \$\$
BEGIN
  IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname='zitadel') THEN
    CREATE ROLE zitadel LOGIN;
  END IF;
END
\$\$;
ALTER USER postgres PASSWORD '${POSTGRES_PASS}';
ALTER USER zitadel PASSWORD '${POSTGRES_PASS}';
GRANT CONNECT, CREATE ON DATABASE zitadel TO zitadel;
ALTER ROLE zitadel WITH SUPERUSER;
SQL"

ok "Postgres patched"

#####################
####### Master Key
#####################

step "Creating Zitadel masterkey"

# Generate EXACTLY 32 CHARACTERS (what Zitadel actually validates)
MASTERKEY=$(openssl rand -base64 24 | tr -d '\n' | cut -c1-32)

# Hard validation
if [ -z "$MASTERKEY" ]; then
  echo "Failed to generate masterkey"
  exit 1
fi

if [ "$(echo -n "$MASTERKEY" | wc -c)" -ne 32 ]; then
  echo " Masterkey is not 32 characters"
  exit 1
fi

# Ensure namespace exists
kubectl get ns zitadel >/dev/null 2>&1 || kubectl create ns zitadel

# Create / update secret (idempotent)
kubectl create secret generic zitadel-masterkey \
  --from-literal=masterkey="$MASTERKEY" \
  -n zitadel \
  --dry-run=client -o yaml | kubectl apply -f -

# Verify secret exists
kubectl get secret zitadel-masterkey -n zitadel >/dev/null

ok "Zitadel masterkey created"

# =========================
# Zitadel Helm values
# =========================
step "Writing Zitadel Helm values"

cat > "$INSTALL_DIR/zitadel-values.yaml" <<EOF
zitadel:
  masterkey: "$MASTERKEY"
  configmapConfig:
    ExternalDomain: $ZITADEL_DOMAIN
    ExternalPort: 8443
    ExternalSecure: true
    TLS:
      Enabled: false
      External: true
    Database:
      Postgres:
        Host: zitadel-postgres-csi
        Port: 5432
        Database: zitadel
        User:
          Username: zitadel
          SSL:
            Mode: disable
        Admin:
          Username: zitadel
          SSL:
            Mode: disable
  secretConfig:
    Database:
      Postgres:
        User:
          Password: ${POSTGRES_PASS}
        Admin:
          Password: ${POSTGRES_PASS}

ingress:
  enabled: true
  hosts:
    - host: $ZITADEL_DOMAIN
      paths:
        - path: /
          pathType: Prefix
EOF
ok "Zitadel Helm values written"

# =========================
# Host file
# =========================
step "setting /etc/hosts"

# Detect host IP automatically
HOST_IP=$(hostname -I | awk '{print $1}')
[[ -z "$HOST_IP" ]] && fail "Could not detect host IP"

# Ensure domain is set
ZITADEL_DOMAIN=${ZITADEL_DOMAIN:-zitadel.local}

# Update /etc/hosts if missing
if ! grep -q "$ZITADEL_DOMAIN" /etc/hosts; then
    echo "$HOST_IP $ZITADEL_DOMAIN" | sudo tee -a /etc/hosts
    ok "/etc/hosts updated with $HOST_IP $ZITADEL_DOMAIN"
else
    ok "/etc/hosts already contains $ZITADEL_DOMAIN"
fi
#----------
# PVC name
#----------
PVC_NAME="zitadel-postgres-data-csi"
NAMESPACE="zitadel"

# Get the PV name bound to this PVC
PV_NAME=$(kubectl get pvc "$PVC_NAME" -n "$NAMESPACE" -o jsonpath='{.spec.volumeName}')

# Get the host path of the PV
PV_PATH=$(kubectl get pv "$PV_NAME" -o jsonpath='{.spec.local.path}')

# Fix permissions inside KIND node container
NODE_CONTAINER="${CLUSTER_NAME}-control-plane"

docker exec "$NODE_CONTAINER" bash -c "
  cd \"$PV_PATH\"
  mkdir -p .velero
  chmod 755 .
  chmod 755 .velero
"

echo "OpenEBS PV permissions fixed at: $PV_PATH/.velero"


# =========================
# Install / Upgrade Zitadel
# =========================
step "Installing Zitadel"
helm repo add zitadel https://charts.zitadel.com
helm repo update

helm upgrade --install zitadel zitadel/zitadel \
  -n zitadel \
  -f "$INSTALL_DIR/zitadel-values.yaml"

kubectl wait --for=condition=Available deployment/zitadel \
  -n zitadel --timeout=600s

# =========================
# Disable login_v2 (DB)
# =========================
step "Disabling login_v2"

INSTANCE_ID=$(kubectl exec -n zitadel "$POSTGRES_POD" -- \
  sh -c "psql -h localhost -U postgres -d zitadel \
  -t -A -P pager=off \
  -c \"SELECT id FROM projections.instances LIMIT 1;\"")

kubectl exec -n zitadel "$POSTGRES_POD" -- \
  sh -c "psql -h localhost -U postgres -d zitadel \
  -v ON_ERROR_STOP=1 -P pager=off <<SQL
UPDATE projections.instance_features2
SET value='{\"required\": false}'
WHERE instance_id='${INSTANCE_ID}' AND key='login_v2';
SQL"

kubectl delete pod -n zitadel -l app.kubernetes.io/name=zitadel-login || true

ok "login_v2 disabled"

step "Creating systemd service for port-forwarding ingress-nginx"

PORT_FORWARD_FILE="/etc/systemd/system/k8s-port-forward.service"

cat > "$PORT_FORWARD_FILE" <<EOF
[Unit]
Description=Port forward ingress-nginx
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8443:443 --address 0.0.0.0
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd and enable the service
systemctl daemon-reload
systemctl enable k8s-port-forward
systemctl restart k8s-port-forward

ok "Port-forward service created and started"


# =========================
# DONE
# =========================
echo -e "\n${GREEN} Zitadel installation complete${RESET}"
echo
echo -e "${CYAN}Access URL:${RESET}"
echo "  https://${ZITADEL_DOMAIN}:8443"
echo
echo -e "${CYAN}Zitadel Admin Login:${RESET}"
echo "  User: ${ZITADEL_ADMIN_USER}"
echo "  Pass: ${ZITADEL_ADMIN_PASS}"
echo
```
