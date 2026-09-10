# New Relic no Kubernetes

A integração é instalada com o Helm chart oficial `newrelic/nri-bundle`
(versão fixada em `kubernetes/addons/versions.env`), sempre depois de o cluster
EKS existir.

## Componentes habilitados

| Subchart | Função |
|---|---|
| `newrelic-infrastructure` | nodes, namespaces, deployments, pods e containers |
| `kube-state-metrics` | estado dos objetos (réplicas, restarts, HPA) |
| `nri-kube-events` | eventos do Kubernetes (OOMKilled, FailedScheduling, scaling) |
| `newrelic-logging` | encaminhamento dos logs de `stdout` |
| `newrelic-prometheus-agent` | métricas Prometheus dos alvos anotados |
| `nri-metadata-injection` | injeta metadados Kubernetes nos pods da aplicação |

O control plane do EKS é gerenciado pela AWS e permanece desabilitado no
agente. Pixie, eBPF, `nri-prometheus` e operators ficam desativados por custo e
compatibilidade com o AWS Academy.

## Correlação de entidades

- `nri-metadata-injection` injeta as variáveis
  `NEW_RELIC_METADATA_KUBERNETES_CLUSTER_NAME`, `..._NAMESPACE_NAME`,
  `..._DEPLOYMENT_NAME`, `..._POD_NAME` e `..._CONTAINER_NAME` nos pods, ligando
  a entidade APM ao cluster, namespace, deployment e pod;
- `global.cluster` recebe o nome do cluster EKS;
- `global.customAttributes` adiciona `project`, `environment` e `repository` a
  todas as métricas e entidades do cluster;
- `kube-state-metrics` expõe as labels `app.kubernetes.io/name`,
  `app.kubernetes.io/part-of` e `app.kubernetes.io/component`;
- os logs encaminhados carregam `cluster_name`, `namespace_name`, `pod_name` e
  `container_name`.

Com isso, dashboards e alertas filtram por cluster, namespace, deployment, pod
e ambiente usando os mesmos atributos.

## Logs sem duplicidade

A aplicação escreve logs JSON em `stdout` e o agente Java decora cada linha com
`trace.id` e `span.id`. O encaminhamento é feito **apenas** pelo
`newrelic-logging` (Fluent Bit) do cluster.

Regra: o agente Java deve manter
`NEW_RELIC_APPLICATION_LOGGING_FORWARDING_ENABLED=false`. Se o agente também
encaminhar, cada linha é ingerida duas vezes, dobrando custo e distorcendo as
consultas de erro.

## Segurança da license key

- fornecida pelo secret `NEW_RELIC_LICENSE_KEY` do GitHub Environment;
- convertida em Secret Kubernetes `newrelic-license` (chave `licenseKey`) no
  namespace `newrelic`, no momento do deploy;
- referenciada por `global.customSecretName` e `global.customSecretLicenseKey`;
- nunca aparece em `values.yaml`, em state do Terraform, em saída de comando ou
  em logs de pipeline;
- o CI recusa qualquer valor em `global.licenseKey` ou `global.insightsKey`.

## Observabilidade como código

Dashboards, política de alertas, condições e monitor sintético são gerenciados
por Terraform em `observability/newrelic/`. Consulte
[Observabilidade como código](observability-as-code.md).
