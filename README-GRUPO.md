# FIAP Store — Global Solution (Cloud Developer II)

Plataforma Kubernetes sobre Amazon EKS que executa a FIAP Store, construída pelo grupo
como parte da Global Solution. Este README documenta **como criar e destruir** todo o
ambiente (item M1 da avaliação) e onde está cada peça da nota.

## Integrantes

<!-- Preencham antes da entrega -->
| Nome | RM |
|---|---|
| … | … |
| … | … |

## Uso de IA

<!-- Item 14 do enunciado: declarar ferramentas de IA e para quê foram usadas -->
- Claude Code (Anthropic) — geração inicial dos manifestos Kubernetes e revisão do README.

## Visão geral

```
Internet (HTTPS)
       │
  [ Traefik (LoadBalancer) ]  namespace: traefik
       │
  [ Ingress store-ingress ]  namespace: store-prod (PSA restricted)
     /api ──▶ store-api (3..6 réplicas, HPA, PDB, spread por zona)
      /  ──▶ store-front (2 réplicas)
                    │
                    ▼
                [ Redis store-redis ] ◀── KEDA (0..5 store-worker)
                    │
                    ▼
                [ Postgres store-db ] (StatefulSet + EBS gp3)
                                         namespace: store-data (PSA baseline)

  [ Job db-migrate ]   [ CronJob store-report ]   namespace: store-prod
  [ DaemonSet node-agent ]                        namespace: platform-monitoring
  [ Argo CD Application fiap-store ]              namespace: argocd
```

## Pré-requisitos locais

- `aws`, `eksctl`, `kubectl`, `helm`, `docker` com `buildx`, `git`
- Conta AWS Academy Learner Lab ativa (`~/.aws/credentials` atualizado a cada sessão; se usar um perfil
  nomeado, exporte `AWS_PROFILE` antes dos comandos)

## Criação do ambiente (do zero)

### 1. Cluster EKS

Substitua `ACCOUNT_ID` em `cluster.yaml` pelo número da conta do seu lab.

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

sed -i.bak "s/ACCOUNT_ID/$ACCOUNT_ID/g" cluster.yaml
eksctl create cluster -f cluster.yaml          # ~20 min
kubectl get nodes -L topology.kubernetes.io/zone
```

### 2. Imagens no ECR

```bash
export REGISTRY=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

for repo in store-api store-worker store-front; do
  aws ecr create-repository --repository-name $repo --region $AWS_REGION || true
done

aws ecr get-login-password --region $AWS_REGION \
  | docker login --username AWS --password-stdin $REGISTRY

docker buildx build --platform linux/amd64 --build-arg APP_VERSION=1.0 \
  -t $REGISTRY/store-api:1.0 --push store-api
docker buildx build --platform linux/amd64 --build-arg APP_VERSION=2.0 \
  -t $REGISTRY/store-api:2.0 --push store-api
docker buildx build --platform linux/amd64 \
  -t $REGISTRY/store-worker:1.0 --push store-worker
docker buildx build --platform linux/amd64 \
  -t $REGISTRY/store-front:1.0 --push store-front
```

### 3. Componentes de plataforma (Traefik, KEDA, Argo CD)

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm upgrade --install traefik traefik/traefik -n traefik --create-namespace \
  --set deployment.replicas=2 --wait
helm upgrade --install keda kedacore/keda -n keda --create-namespace --wait

kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4. Secrets (fora do Git — item M2)

```bash
kubectl create namespace store-prod
kubectl create namespace store-data

DB_PASSWORD=$(openssl rand -base64 24 | tr -d '=+/' | head -c 24)
kubectl -n store-data create secret generic store-db-credentials \
  --from-literal=password="$DB_PASSWORD"
kubectl -n store-prod create secret generic store-db-credentials \
  --from-literal=password="$DB_PASSWORD"

# aguarde o Traefik publicar o hostname do balanceador
HOST=""
while [ -z "$HOST" ]; do
  HOST=$(kubectl -n traefik get svc traefik \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  [ -z "$HOST" ] && sleep 10
done

openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -subj "/CN=fiap-store" -addext "subjectAltName=DNS:$HOST" \
  -keyout /tmp/tls.key -out /tmp/tls.crt
kubectl -n store-prod create secret tls store-tls \
  --cert=/tmp/tls.crt --key=/tmp/tls.key
rm /tmp/tls.key /tmp/tls.crt
```

### 5. Mapear usuário IAM para o grupo `fiap:devs`

Edite o ConfigMap `aws-auth` no namespace `kube-system` adicionando o usuário/role IAM
do desenvolvedor ao grupo `fiap:devs`:

```bash
kubectl edit configmap aws-auth -n kube-system
```

```yaml
# exemplo de entrada em mapUsers (role do lab também pode ir em mapRoles)
- userarn: arn:aws:iam::ACCOUNT_ID:user/dev.aluno
  username: dev.aluno
  groups:
    - fiap:devs
```

### 6. Sincronizar via Argo CD

O overlay `k8s/overlays/eks` já aponta para o ECR da conta do lab (`701096295882`, `us-east-1`)
e a Application aponta para este repositório. Em outra conta, troque o número em
`k8s/overlays/eks/kustomization.yaml`.

```bash
kubectl apply -f k8s/argocd/application.yaml
kubectl -n argocd get applications -w     # esperar Synced / Healthy
```

- Repositório sincronizado: https://github.com/gym187/fiap-store (path `k8s/overlays/eks`, branch `main`)
- Application: `fiap-store` no namespace `argocd` (`prune` e `selfHeal` ligados)

Argo CD URL:

```bash
kubectl -n argocd port-forward svc/argocd-server 8081:443 &
# usuario: admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### 7. Verificar

```bash
./avaliacao/verifica-gs.sh "https://$HOST"
```

## Destruição do ambiente

Ordem obrigatória — ignorar provoca volumes/load balancers órfãos (custo):

```bash
# 1. remover a Application do Argo CD (senão ele recria tudo)
kubectl -n argocd delete application fiap-store

# 2. apagar os namespaces da aplicação
kubectl delete namespace store-prod store-data platform-monitoring

# 3. aguardar os PVs sumirem (o PVC apagado remove o volume EBS)
kubectl get pv -w       # Ctrl+C quando a lista estiver vazia

# 4. remover o balanceador (Traefik)
helm uninstall traefik -n traefik
kubectl delete namespace traefik keda argocd

# 5. destruir o cluster
eksctl delete cluster -f cluster.yaml --disable-nodegroup-eviction

# 6. conferir no console: nenhum volume EBS, nenhum load balancer, nenhum ENI pendente
```

## Mapa pontuação → arquivos

| Item | Onde |
|---|---|
| A1 cluster EKS | `cluster.yaml` |
| A2 imagens ECR | `k8s/overlays/eks/kustomization.yaml` (campo `images`) |
| A3 sem tag `latest` | todos os manifestos usam tag explícita |
| B1–B7 | `k8s/base/store-prod/store-api.yaml`, `hpa-pdb.yaml` |
| C1 namespaces | `k8s/base/namespaces.yaml` |
| C2 services internos | `k8s/base/store-data/*.yaml` (sem NodePort/LB) |
| C3 Ingress | `k8s/base/store-prod/ingress.yaml` |
| C4 TLS | Secret `store-tls` criado via `kubectl` (passo 4) + Ingress |
| C5 deny ingress em store-data | `k8s/base/store-data/networkpolicies.yaml` |
| C6 NetworkPolicy efetiva | `k8s/base/store-data/networkpolicies.yaml` + `store-prod/networkpolicies.yaml` |
| C7 Quota + LimitRange | `k8s/base/store-prod/quota-limitrange.yaml` |
| D1–D2 Postgres + EBS | `k8s/base/store-data/store-db.yaml`, `storage-class.yaml` |
| D3 ConfigMap | `k8s/base/store-prod/configmap.yaml` |
| D4 Downward API | `k8s/base/store-prod/store-api.yaml` (bloco `env`) |
| D5 DaemonSet | `k8s/base/platform-monitoring/daemonset.yaml` |
| D6 Redis | `k8s/base/store-data/store-redis.yaml` |
| E1 Job db-migrate | `k8s/base/store-prod/db-migrate.yaml` (Argo hook) |
| E2 CronJob | `k8s/base/store-prod/store-report.yaml` |
| E3 Secret da senha | referenciado em `store-api.yaml` (volume `db-secret`) |
| F1 SA + ClusterRole | `k8s/base/store-prod/serviceaccounts.yaml`, `rbac.yaml` |
| F2 token desligado | front e worker com `automountServiceAccountToken: false` |
| F3 grupo fiap:devs | `k8s/base/store-prod/rbac.yaml` |
| F4 PSA restricted | `k8s/base/namespaces.yaml` (label em store-prod) |
| F5 securityContext | em todos os Deployments/Job/CronJob |
| G1 KEDA | `k8s/base/store-prod/keda-scaledobject.yaml` |
| G2 Argo CD | `k8s/argocd/application.yaml` |
| M1 README criar/destruir | este arquivo |
| M3 scan Trivy | `avaliacao/trivy/` (gerado pelo grupo após build) |

## Troubleshooting rápido

Ver seção 9 do enunciado (`docs/GS-Enunciado-Alunos.md`). Armadilhas mais prováveis:

- **Pod recusado por PSA**: completar o `securityContext` do container exatamente como está nos manifestos existentes.
- **PVC `Pending`**: StorageClass `gp3` não marcada default — confira a annotation.
- **NetworkPolicy não bloqueia**: addon `vpc-cni` precisa de `enableNetworkPolicy: true` (já está no `cluster.yaml`).
- **HPA `<unknown>`**: addon `metrics-server` não subiu; sem `resources.requests` no Deployment não há como calcular %.
- **Argo CD em loop**: `replicas` declarado em Deployment escalado por HPA/KEDA — nos nossos manifestos `store-api` e `store-worker` **não declaram `replicas`** justamente por isso.
