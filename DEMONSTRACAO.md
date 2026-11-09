# FIAP Store — Demonstração e resolução

**Global Solution — Cloud Native** · Professor Fabio Bara · Turma **2TCNPZ**

| Nome | RM |
|---|---|
| Bruno Henrique | RM566277 |
| Guylherme Miguel | RM562374 |

## Vídeo

▶️ **https://youtu.be/p-7EineDkBE**

O vídeo percorre os 12 comandos abaixo. Para cada um, este documento traz o comando, a saída
real obtida no cluster EKS e a explicação do que ela comprova.

Preparação usada em todos os comandos:

```bash
export AWS_PROFILE=fiap-lab
export HOST=a8674d7a8f7bc449eb24bb1ad983a114-316955218.us-east-1.elb.amazonaws.com
```

---

## 1. Loja em HTTPS e Painel da plataforma

```bash
curl -sk -o /dev/null -w 'HTTP %{http_code}\n' https://$HOST/
```

```
HTTP 200
```

**Resolução:** a loja responde em HTTPS pelo Load Balancer da AWS criado pelo Traefik. O Ingress
`store-ingress` encaminha `/api` para a `store-api` e `/` para o `store-front`, usando o Secret
TLS `store-tls`. No navegador, o **Painel da plataforma** (autodiagnóstico feito de dentro do
cluster) mostra todos os itens verdes: réplicas, zonas, Downward API, menor privilégio, Secret,
banco, worker, CronJob, Redis e `securityContext`.

## 2. Cluster EKS em várias zonas

```bash
kubectl get nodes -L topology.kubernetes.io/zone
```

```
NAME                             STATUS   ROLES    AGE   VERSION               ZONE
ip-192-168-20-215.ec2.internal   Ready    <none>   79m   v1.35.9-eks-6e7947a   us-east-1a
ip-192-168-54-124.ec2.internal   Ready    <none>   79m   v1.35.9-eks-6e7947a   us-east-1b
ip-192-168-65-120.ec2.internal   Ready    <none>   79m   v1.35.9-eks-6e7947a   us-east-1c
ip-192-168-89-182.ec2.internal   Ready    <none>   79m   v1.35.9-eks-6e7947a   us-east-1c
```

**Resolução:** 4 nós `t3.medium` prontos em **3 zonas de disponibilidade** (item A1). O cluster
foi criado com `eksctl create cluster -f cluster.yaml`, usando a `LabRole` (o Learner Lab não
permite criar roles IAM). São 4 nós porque cada `t3.medium` aceita no máximo 17 Pods.

## 3. Visão geral da aplicação

```bash
kubectl get pods -n store-prod -o wide
kubectl get pods,pvc -n store-data
```

```
NAME                           READY   STATUS      RESTARTS   AGE    IP               NODE
store-api-649bb69bd9-hrslj     1/1     Running     0          29m    192.168.64.86    ip-192-168-65-120.ec2.internal
store-api-649bb69bd9-lgswl     1/1     Running     0          72m    192.168.73.74    ip-192-168-89-182.ec2.internal
store-api-649bb69bd9-sx57s     1/1     Running     0          127m   192.168.54.145   ip-192-168-54-124.ec2.internal
store-front-56dbc9fb5f-mtwql   1/1     Running     0          127m   192.168.47.193   ip-192-168-54-124.ec2.internal
store-front-56dbc9fb5f-w4sjf   1/1     Running     0          127m   192.168.35.127   ip-192-168-54-124.ec2.internal
store-report-29904055-k28jb    0/1     Completed   0          14m    192.168.73.235   ip-192-168-89-182.ec2.internal
store-report-29904060-9647w    0/1     Completed   0          9m6s   192.168.92.113   ip-192-168-89-182.ec2.internal
store-report-29904065-bvcps    0/1     Completed   0          4m6s   192.168.79.240   ip-192-168-89-182.ec2.internal

NAME                               READY   STATUS    RESTARTS   AGE
pod/store-db-0                     1/1     Running   0          22m
pod/store-redis-798765665d-dg59c   1/1     Running   0          127m

NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
persistentvolumeclaim/data-store-db-0   Bound    pvc-64dca4d8-ae74-4a30-8f15-180b9343e171   5Gi        RWO            gp3
```

**Resolução:** a aplicação fica em dois namespaces. No `store-prod`, a `store-api` tem 3 réplicas
em nós diferentes, o `store-front` tem 2, e as execuções `Completed` são do CronJob
`store-report` (a cada 5 minutos). O `store-worker` não aparece porque está em **zero réplicas**:
só sobe quando há fila (ver comando 9). No `store-data` ficam o Postgres (`store-db-0`, um
StatefulSet) e o Redis. O PVC está `Bound` num volume **EBS gp3** provisionado pelo driver
`ebs.csi.aws.com` (itens D1, D2 e D6). Banco e Redis só têm Services `ClusterIP` (item C2).

## 4. Alta disponibilidade: HPA e PDB

```bash
kubectl get hpa,pdb -n store-prod
```

```
NAME                                                        REFERENCE                 TARGETS             MINPODS   MAXPODS   REPLICAS
horizontalpodautoscaler.autoscaling/keda-hpa-store-worker   Deployment/store-worker   <unknown>/5 (avg)   1         5         0
horizontalpodautoscaler.autoscaling/store-api               Deployment/store-api      cpu: 4%/70%         3         6         3

NAME                                   MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
poddisruptionbudget.policy/store-api   2               N/A               1
```

**Resolução:** o HPA da `store-api` mantém de 3 a 6 réplicas e **lê a CPU** (`4%/70%`, não
`<unknown>`), graças ao addon `metrics-server` e aos `requests` declarados (item B5). O PDB exige
no mínimo 2 réplicas disponíveis, então numa manutenção só uma réplica sai por vez (item B6). O
`keda-hpa-store-worker` é criado pelo KEDA. O `<unknown>` dele é normal: com o worker em zero, não
há métrica a calcular, e quem controla a escala a partir de zero é o próprio KEDA.

## 5. Self-healing

```bash
kubectl -n store-prod delete $(kubectl -n store-prod get pod -l app=store-api -o name | head -1)
kubectl get pods -n store-prod -l app=store-api
```

```
pod "store-api-649bb69bd9-hrslj" deleted from store-prod namespace
NAME                         READY   STATUS    RESTARTS   AGE
store-api-649bb69bd9-kfdbq   1/1     Running   0          17s
store-api-649bb69bd9-lgswl   1/1     Running   0          72m
store-api-649bb69bd9-sx57s   1/1     Running   0          128m
```

**Resolução:** o Pod apagado foi substituído na hora pelo ReplicaSet (o novo tem 17 s). O Pod
novo só recebe tráfego depois que a readiness em `/ready` confirma o acesso ao banco, e as outras
duas réplicas atendem nesse intervalo. Além disso, o `preStop: sleep 10` e o Middleware de retry
do Traefik evitam erros 502 durante a troca. A loja não sai do ar (desafio 1).

## 6. Rede: o front não alcança o banco

```bash
kubectl -n store-prod exec deploy/store-front -- nc -zv -w3 store-db.store-data.svc.cluster.local 5432
kubectl -n store-prod exec deploy/store-api -- python -c "import socket;socket.create_connection(('store-db.store-data.svc.cluster.local',5432),3);print('conectou')"
```

```
nc: store-db.store-data.svc.cluster.local (192.168.61.137:5432): Operation timed out
command terminated with exit code 1
---
conectou
```

**Resolução:** do `store-front`, a conexão com o Postgres dá **timeout**: a NetworkPolicy descarta
o pacote. Da `store-api`, a conexão funciona (itens C5 e C6, desafio 9). No `store-data`, a
`default-deny-ingress` nega toda entrada e políticas específicas liberam o banco só para
`store-api`, `store-worker`, `db-migrate` e `store-report`. O Redis fica liberado para a API, o
worker e o operador do KEDA. As políticas são aplicadas de fato porque o addon `vpc-cni` foi
criado com `enableNetworkPolicy: "true"`.

## 7. Pod Security Admission

```bash
kubectl -n store-prod run priv --image=nginx:1.27 \
  --overrides='{"spec":{"containers":[{"name":"priv","image":"nginx:1.27","securityContext":{"privileged":true}}]}}'
```

```
Error from server (Forbidden): pods "priv" is forbidden: violates PodSecurity "restricted:latest":
privileged (container "priv" must not set securityContext.privileged=true),
allowPrivilegeEscalation != false (container "priv" must set securityContext.allowPrivilegeEscalation=false),
unrestricted capabilities (container "priv" must set securityContext.capabilities.drop=["ALL"]),
runAsNonRoot != true (pod or container "priv" must set securityContext.runAsNonRoot=true),
seccompProfile (pod or container "priv" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

**Resolução:** o namespace `store-prod` tem o label `pod-security.kubernetes.io/enforce=restricted`,
então a API recusa o Pod e lista exatamente as violações (item F4, desafio 6). Os nossos
Deployments atendem todas essas regras: `runAsNonRoot`, `seccompProfile: RuntimeDefault`,
`readOnlyRootFilesystem`, `allowPrivilegeEscalation: false` e `capabilities.drop: ["ALL"]`
(item F5). O nginx do front recebe um `emptyDir` em `/tmp`, porque precisa gravar arquivos
temporários.

## 8. RBAC do time de desenvolvimento

```bash
kubectl auth can-i list pods -n store-prod --as=dev --as-group=fiap:devs
kubectl auth can-i get secrets -n store-prod --as=dev --as-group=fiap:devs
```

```
yes
no
```

**Resolução:** o grupo `fiap:devs` **lê Pods** (e logs), mas **não lê Secrets**. Ele também não
apaga Pods nem faz `exec`, como confere o item F3 do script. Na prática, a exigência
"desenvolvedores olham, não mexem" é uma Role só com `get`, `list` e `watch` em `pods` e
`pods/log`, ligada ao grupo por RoleBinding. A `store-api` usa uma ServiceAccount própria, que lê
`nodes` (ClusterRole) e não lê Secrets (item F1). O `store-front` e o `store-worker` rodam com
`automountServiceAccountToken: false` (item F2).

## 9. KEDA: escala a partir de zero

```bash
curl -sk -X POST "https://$HOST/api/pedidos/lote?quantidade=50"
kubectl get pods -n store-prod -l app=store-worker -w
```

```
{"criados":50,"primeiro":201,"ultimo":250}
-- t=0s
No resources found
-- t=15s
store-worker-687bff4f9c-4spdb 0/1 ContainerCreating
store-worker-687bff4f9c-6sg6g 1/1 Running
store-worker-687bff4f9c-dvt52 0/1 ContainerCreating
store-worker-687bff4f9c-v9pnn 0/1 ContainerCreating
-- t=30s
store-worker-687bff4f9c-4spdb 1/1 Running
store-worker-687bff4f9c-6sg6g 1/1 Running
store-worker-687bff4f9c-dvt52 1/1 Running
store-worker-687bff4f9c-v9pnn 1/1 Running
store-worker-687bff4f9c-xhbdk 1/1 Running
-- depois do cooldown (~60 s sem fila):
No resources found in store-prod namespace.
{'pendentes': 0, 'processados': 250}
```

**Resolução:** o worker começa em **0 réplicas**. Com 50 pedidos na fila `pedidos` do Redis, o
ScaledObject do KEDA (`listLength: 5`, mínimo 0, máximo 5) sobe o worker até **5 réplicas**, que
processam tudo. Depois do `cooldownPeriod` de 60 s sem fila, o worker volta a **zero** (item G1,
desafio 4). O Deployment `store-worker` não declara `replicas`, para o Argo CD não disputar o
número de réplicas com o KEDA.

## 10. Persistência do banco

```bash
kubectl -n store-data delete pod store-db-0
kubectl get pod store-db-0 -n store-data
curl -sk "https://$HOST/api/pedidos?limite=1"
```

```
pod "store-db-0" deleted from store-data namespace
NAME         READY   STATUS    RESTARTS   AGE
store-db-0   1/1     Running   0          12s

{"resumo":{"pendentes":0,"processados":250}, ...}
```

**Resolução:** o StatefulSet recriou o `store-db-0` com o mesmo nome e remontou o **mesmo volume
EBS**, e os 250 pedidos continuam no banco (desafio 5). Nos ~6 s em que o Postgres está subindo,
a `/ready` da API responde 503 e as réplicas saem temporariamente do balanceamento. Esse é o
comportamento esperado da readiness. O `PGDATA` fica num subdiretório do volume, por causa do
`lost+found` na raiz do disco EBS.

## 11. GitOps: drift desfeito pelo Argo CD

```bash
kubectl -n store-prod delete service store-front
kubectl get svc store-front -n store-prod
kubectl -n argocd get application fiap-store
```

```
service "store-front" deleted from store-prod namespace
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
store-front   ClusterIP   10.100.9.229   <none>        8080/TCP   5s

última operação: 2026-11-09T17:12:34Z successfully synced (all tasks run) | Synced Healthy
```

**Resolução:** o Service apagado na mão voltou em segundos (`AGE 5s`). A Application `fiap-store`
do Argo CD tem `selfHeal: true`: ao detectar que o cluster divergiu do Git, ela sincroniza
automaticamente (desafio 8). Com `prune: true`, o que é removido do repositório também sai do
cluster. A Application continua **Synced** e **Healthy** (item G2). O rollout 1.0 → 2.0 e o
rollback também foram feitos **por commit** neste repositório (desafio 3, item B7).

## 12. Script oficial de correção

```bash
./avaliacao/verifica-gs.sh https://$HOST
```

```
===================================================================================
 FIAP Store - Global Solution - verificacao automatica
 Data:     09/11/2026 14:12:41
 Contexto: user3940975=Guylherme_Ivanildo_Miguel@fiap-store.us-east-1.eksctl.io
 Servidor: https://6E0EADCB0857725C43378C07911B0F03.gr7.us-east-1.eks.amazonaws.com
 Conta AWS (credenciais atuais): 701096295882
===================================================================================

A. Cluster e imagens (cap. 2 e 6)
  [OK]   A1   Cluster EKS com 3+ nos prontos em 2+ zonas                      3/3
         -> EKS: sim | nos prontos: 4 | zonas: 3
  [OK]   A2   store-api, store-worker e store-front com imagens do ECR        3/3
         -> 3 de 3 imagens vem do ECR | registro: 701096295882.dkr.ecr.us-east-1.amazonaws.com
  [OK]   A3   Nenhuma imagem com tag latest ou sem tag                        2/2

B. Workloads e alta disponibilidade (cap. 3 e 4)
  [OK]   B1   store-api com 3+ replicas prontas                               2/2
         -> replicas prontas: 3
  [OK]   B2   store-api: readiness em /ready e liveness em /health            2/2
         -> readiness: /ready | liveness: /health
  [OK]   B3   requests e limits de CPU e memoria nos 3 Deployments            2/2
  [OK]   B4   Replicas da store-api em 2+ zonas de disponibilidade            3/3
         -> zonas: us-east-1a us-east-1b us-east-1c
  [OK]   B5   HPA da store-api: minimo 3 e metrica de CPU sendo lida          3/3
         -> alvo: store-api | min: 3 | CPU atual: 4%
  [OK]   B6   PodDisruptionBudget protegendo a store-api                      2/2
         -> minAvailable: 2 | maxUnavailable: -
  [OK]   B7   Rolling update com maxUnavailable 0 e 2+ revisoes               2/2
         -> maxUnavailable: 0 | ReplicaSets: 5

C. Rede e namespaces (cap. 5)
  [OK]   C1   Namespaces store-prod e store-data                              1/1
  [OK]   C2   store-db e store-redis so com Services internos                 2/2
  [OK]   C3   Ingress store-ingress: /api -> store-api e / -> store-front     3/3
         -> rotas: /api store-api;/ store-front;
  [OK]   C4   TLS no Ingress e loja respondendo em HTTPS                      2/2
         -> secret: store-tls (kubernetes.io/tls) | GET https://a8674d7a8f7bc449eb24bb1ad983a114-316955218.us-east-1.elb.amazonaws.com/api/produtos -> 200
  [OK]   C5   NetworkPolicy que nega toda entrada em store-data               2/2
         -> politica: default-deny-ingress
  [OK]   C6   NetworkPolicy efetiva: front NAO alcanca o banco; api alcanca   2/2
         -> front->db: bloqueado | api->db: conecta
  [OK]   C7   ResourceQuota e LimitRange em store-prod                        2/2
         -> ResourceQuota: 1 | LimitRange: 1

D. Dados, configuracao e daemons (cap. 7)
  [OK]   D1   StatefulSet store-db pronto com PVC Bound                       3/3
         -> prontas: 1 | PVC: data-store-db-0|Bound|gp3
  [OK]   D2   Volume do banco provisionado pelo driver EBS CSI                1/1
         -> StorageClass: gp3 | provisioner: ebs.csi.aws.com
  [OK]   D3   ConfigMap store-api-config usado pela store-api                 2/2
         -> ConfigMaps referenciados: store-api-config
  [OK]   D4   Downward API: nome do Pod, namespace e no na store-api          2/2
         -> campos: metadata.name metadata.namespace spec.nodeName
  [OK]   D5   DaemonSet proprio com um Pod pronto em cada no                  3/3
         -> DaemonSets: platform-monitoring/node-agent
  [OK]   D6   Redis store-redis pronto                                        1/1

E. Jobs e Secrets (cap. 8)
  [OK]   E1   Job db-migrate concluido                                        2/2
         -> execucoes bem-sucedidas: 1
  [OK]   E2   CronJob store-report com execucao bem-sucedida                  3/3
         -> agenda: */5 * * * * | ultimo sucesso: 2026-11-09T17:10:06Z
  [OK]   E3   Senha do banco vinda do Secret store-db-credentials             3/3
         -> Secrets referenciados: store-db-credentials

F. RBAC e seguranca (cap. 9 e 10)
  [OK]   F1   store-api: ServiceAccount propria, le nodes e NAO le Secrets    3/3
         -> SA: store-api | get nodes: yes | list secrets: no
  [OK]   F2   Token da API desligado em store-front e store-worker            2/2
  [OK]   F3   Grupo fiap:devs: le Pods e logs; sem delete, Secrets ou exec    3/3
         -> list pods: yes | logs: yes | delete: no | secrets: no | exec: no
  [OK]   F4   Pod Security restricted (enforce) em store-prod                 3/3
         -> enforce: restricted
  [OK]   F5   securityContext completo nos 3 Deployments                      3/3

G. Serverless e GitOps (cap. 8 e 11)
  [OK]   G1   KEDA: store-worker escala a partir de zero                      4/4
         -> alvo: store-worker | min: 0 | trigger: redis | pronto: True
  [OK]   G2   Argo CD: Application Synced + Healthy com prune e selfHeal      4/4
         -> fiap-store|Synced|Healthy|true|true;

===================================================================================
 Pontuacao automatica: 80 / 80
===================================================================================
```

**Resolução:** o script oficial, que só faz leituras no cluster, confirmou **80 de 80** pontos
automáticos. Os itens manuais estão no repositório: M1 no [`README.md`](README.md) (criar e
destruir o ambiente), M2 nos Secrets criados via `kubectl` e nunca versionados, e M3 em
[`avaliacao/trivy/`](avaliacao/trivy/README.md).

---

## Problemas encontrados e como resolvemos

| Problema | Causa | Solução |
|---|---|---|
| `store-api` nunca ficava pronta (`/ready` com timeout) | NetworkPolicy de egress com `podSelector: {}` em `store-prod` liberava só o DNS e bloqueava API → banco | Removida. O egress restrito ficou só no `store-front`, e o banco foi liberado também para worker, `db-migrate` e `store-report` |
| `openssl` falhava ao gerar o certificado TLS | O hostname do ELB tem 72 caracteres e o campo CN aceita no máximo 64 | CN curto (`fiap-store`) e hostname só no `subjectAltName`, que é o campo usado pelo Traefik (SNI) |
| Erros 502 esporádicos ao trocar Pods | O Traefik ainda enviava requisições para um Pod encerrando, e o VPC CNI derruba conexões quando aplica a política num Pod novo | `preStop: sleep 10` na API e no front, e Middleware `retry` do Traefik no Ingress |
| Réplicas da API concentradas numa zona após o rollout | O `topologySpreadConstraints` contava também os Pods da versão antiga | `matchLabelKeys: [pod-template-hash]` |
| Nós não voltaram após a sessão do Learner Lab expirar | As instâncias novas subiram com a `LabRole` bloqueada e o `nodeadm` desistiu do bootstrap (403) | `aws ec2 reboot-instances` depois de reabrir o lab: o bootstrap rodou de novo e os nós entraram |
