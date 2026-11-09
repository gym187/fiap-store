# FIAP Store — manifestos da plataforma

Estrutura dos manifestos que o Argo CD sincroniza:

```
k8s/
├── base/                     # manifestos independentes do ambiente
│   ├── namespaces.yaml       # store-prod (PSA restricted) + store-data (baseline)
│   ├── storage-class.yaml    # gp3 (default) via ebs.csi.aws.com
│   ├── store-data/           # Postgres (StatefulSet) + Redis + NetworkPolicies
│   ├── store-prod/           # API, worker, front, HPA, PDB, Ingress, Job, CronJob, KEDA
│   └── platform-monitoring/  # DaemonSet node-agent (item D5)
├── overlays/
│   ├── eks/                  # injeta imagens do ECR (ACCOUNT_ID + AWS_REGION)
│   └── minikube/             # imagePullPolicy: Never + sem StorageClass default
└── argocd/
    └── application.yaml      # Argo CD Application (aponta para overlays/eks)
```

## Preparação antes do primeiro sync

Antes de aplicar os manifestos **é obrigatório** criar o Secret `store-db-credentials`
nos dois namespaces (não vai para o Git) e o Secret TLS do Ingress:

```bash
# 1. namespaces (o resto do Argo CD cria, mas esses dois precisam existir antes do Secret)
kubectl create namespace store-prod
kubectl create namespace store-data

# 2. senha do banco — mesma senha nos dois namespaces
DB_PASSWORD=$(openssl rand -base64 24 | tr -d '=+/' | head -c 24)
kubectl -n store-data create secret generic store-db-credentials \
  --from-literal=password="$DB_PASSWORD"
kubectl -n store-prod create secret generic store-db-credentials \
  --from-literal=password="$DB_PASSWORD"

# 3. TLS do Ingress (hostname é o balanceador do Traefik)
HOST=$(kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -subj "/CN=fiap-store" -addext "subjectAltName=DNS:$HOST" \
  -keyout /tmp/tls.key -out /tmp/tls.crt
kubectl -n store-prod create secret tls store-tls \
  --cert=/tmp/tls.crt --key=/tmp/tls.key
rm /tmp/tls.key /tmp/tls.crt
```

## Overlay do EKS: ajustar ACCOUNT_ID

Antes do primeiro commit no repo que o Argo CD vai sincronizar, substitua em
`overlays/eks/kustomization.yaml`:

```
ACCOUNT_ID       -> número da conta AWS do seu lab
AWS_REGION       -> us-east-1 (ou a região configurada)
```

Ou use `sed` no bootstrap:

```bash
sed -i "s/ACCOUNT_ID/$ACCOUNT_ID/g; s/AWS_REGION/$AWS_REGION/g" overlays/eks/kustomization.yaml
```

## Aplicar manualmente (sem Argo CD) — para teste local

```bash
# minikube
kubectl apply -k k8s/overlays/minikube

# EKS (depois do substituir ACCOUNT_ID / AWS_REGION)
kubectl apply -k k8s/overlays/eks
```

## Habilitar o Argo CD apontando para o repositório

```bash
# aponte o repoURL do argocd/application.yaml para o seu fork
kubectl apply -f k8s/argocd/application.yaml
kubectl -n argocd get applications
```
