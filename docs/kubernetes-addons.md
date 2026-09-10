# Add-ons e escalabilidade

## Add-ons gerenciados pelo Terraform

- Amazon VPC CNI;
- CoreDNS;
- kube-proxy.

## Add-ons instalados por Helm após o cluster existir

| Add-on | Chart | Versão | Namespace |
|---|---|---|---|
| Metrics Server | `metrics-server/metrics-server` | `3.13.1` | `kube-system` |
| New Relic | `newrelic/nri-bundle` | `8.0.10` | `newrelic` |

As versões ficam em `kubernetes/addons/versions.env` e são usadas tanto pelo
deploy quanto pela validação offline. Os values não contêm segredos:

- `kubernetes/addons/metrics-server.values.yaml`;
- `kubernetes/addons/nri-bundle.values.yaml` (base);
- `kubernetes/addons/nri-bundle.values.homolog.yaml` e
  `nri-bundle.values.production.yaml` (overlays por ambiente).

O provider Kubernetes não é usado no Terraform da infraestrutura: isso evitaria
o `plan` inicial, já que o endpoint do cluster ainda não existe. A instalação é
feita por script idempotente.

## Scripts

| Script | Função |
|---|---|
| `scripts/check-aws-session.sh` | valida `aws sts get-caller-identity` e falha explicitamente em `ExpiredToken` |
| `scripts/deploy-cluster-addons.sh` | `helm upgrade --install` do Metrics Server e do `nri-bundle`, criando o Secret da license key |
| `scripts/verify-cluster-addons.sh` | rollout, pods, HPA e namespace `newrelic` sem exposição de segredos |
| `scripts/lint-cluster-addons.sh` | `helm template` offline com valores de placeholder |

Execução manual (equivalente ao workflow `Deploy`):

```bash
export CLUSTER_NAME=oficina-homolog
export ENVIRONMENT=homolog
export AWS_REGION=us-west-2
export NEW_RELIC_LICENSE_KEY=...   # nunca versionar
./scripts/deploy-cluster-addons.sh
./scripts/verify-cluster-addons.sh
```

Reexecutar os scripts é seguro: `helm upgrade --install` e `kubectl apply`
convergem para o mesmo estado.

## Escalabilidade

1. Metrics Server precisa estar pronto antes de o HPA reportar métricas;
2. Deployment, Service e HPA são aplicados pelo repositório da aplicação;
3. o HPA usa CPU e memória, e o dashboard de Kubernetes acompanha réplicas
   atuais, desejadas e máximo;
4. a condição `hpa_no_maximo` alerta quando o HPA satura.

## Compatibilidade de versões

O cluster usa Kubernetes 1.34. O workflow instala `kubectl v1.34.1` e
`helm v3.18.4`, mantendo a diferença de versão dentro da janela suportada.
