# Scan de vulnerabilidades (Trivy) — item M3

Comando usado (severidades HIGH e CRITICAL, imagens do ECR do grupo):

```bash
trivy image --severity HIGH,CRITICAL --format table \
  -o avaliacao/trivy/<imagem>-<tag>.txt <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/<imagem>:<tag>
```

| Imagem | HIGH | CRITICAL | Relatório |
|---|---:|---:|---|
| `store-api:1.0` | 52 | 0 | `store-api-1.0.txt` |
| `store-api:2.0` | 52 | 0 | `store-api-2.0.txt` |
| `store-worker:1.0` | 49 | 0 | `store-worker-1.0.txt` |
| `store-front:1.0` | 75 | 4 | `store-front-1.0.txt` |

## Justificativa dos CRITICAL (`store-front`)

Os quatro vêm de pacotes Alpine da imagem base do nginx, não do código da loja. O código não pode
ser alterado (regra do enunciado), por isso não reconstruímos a base.

| CVE | Pacote | Por que o risco é aceito / mitigado |
|---|---|---|
| CVE-2026-31789 | `libcrypto3`, `libssl3` | Afeta só sistemas de **32 bits**; os nós EKS são `amd64`. Além disso, o nginx do `store-front` não termina TLS (o TLS fica no Traefik). |
| CVE-2026-42945 | `nginx` | Mitigado pela plataforma: contêiner não-root (UID 101), `readOnlyRootFilesystem`, `capabilities.drop: ALL`, `allowPrivilegeEscalation: false`, seccomp `RuntimeDefault`, sem token da API do Kubernetes e NetworkPolicy impedindo acesso ao `store-data`. Correção: atualizar a base para `nginx 1.28.3-r1`. |
| CVE-2026-111497 | `brotli-libs` | Decoder Brotli; o nginx serve arquivos estáticos e não descomprime conteúdo enviado pelo cliente. Correção: `brotli-libs 1.2.1-r0` na próxima atualização da base. |
