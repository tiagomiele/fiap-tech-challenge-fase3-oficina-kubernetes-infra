# Observabilidade New Relic como código

A configuração fica em `observability/newrelic/` e usa o provider oficial
`newrelic/newrelic` (`~> 3.95.0`) em workspaces HCP Terraform separados dos
workspaces de infraestrutura. Nenhum recurso AWS é criado por esta
configuração e nenhuma chave é versionada.

## Componentes gerenciados

| Arquivo | Recursos |
|---|---|
| `dashboard.tf` | `newrelic_one_dashboard` com as páginas Negócio, Aplicação, Kubernetes, Serverless/API Gateway e RDS PostgreSQL |
| `alerts.tf` / `rds-alerts.tf` | política e condições NRQL de aplicação, Kubernetes, serverless, API Gateway e RDS |
| `synthetics.tf` | `newrelic_synthetics_monitor` do healthcheck e a condição associada |
| `notifications.tf` | `newrelic_notification_destination`, `newrelic_notification_channel` e `newrelic_workflow` opcionais |

Tudo é condicional:

- `observability_enabled = false` mantém a configuração válida sem conta New
  Relic (usado na validação estática do CI);
- o monitor sintético só é criado com `synthetic_monitor_enabled = true` **e**
  `health_check_url` preenchida;
- notificações só são criadas com `notification_enabled = true` e
  `notification_email` preenchido, portanto a validação não depende de ids
  reais de destino.

## Variáveis principais

| Variável | Origem | Observação |
|---|---|---|
| `newrelic_account_id` | variable `NEW_RELIC_ACCOUNT_ID` do GitHub Environment ou variável do workspace | numérico |
| `newrelic_api_key` | secret `NEW_RELIC_API_KEY` (User key `NRAK-...`) | sensível, nunca versionada |
| `environment` | `homolog` ou `production` | define limiares e nomes |
| `cluster_name` | saída `eks_cluster_name` da infraestrutura | vazio usa `oficina-<environment>` |
| `apm_application_name` | nome da aplicação no APM | vazio usa `oficina-backend-<environment>` |
| `kubernetes_namespace` / `kubernetes_deployment_name` | `oficina-homolog` ou `oficina-production` / `oficina-app` | usados nas consultas de Kubernetes |
| `lambda_function_names` / `api_gateway_name` | repositório de autenticação | a configuração central informa as quatro Lambdas e a HTTP API |
| `rds_database_identifier` | repositório de banco | vazio usa `oficina-<environment>-db` |
| `health_check_url` | URL pública do LoadBalancer | vazia desabilita o monitor sintético |
| `alert_thresholds` | mapa por ambiente | limiares configuráveis |
| `notification_enabled` / `notification_email` | opcional | workflow de notificação |

Exemplos completos em `observability/newrelic/environments/*.tfvars.example`.

## Limiares por ambiente

| Limiar | homolog | production |
|---|---|---|
| Taxa de erro (%) | 10 | 5 |
| Latência P95 (ms) | 2000 | 1200 |
| Falhas de processamento de OS | 3 | 1 |
| Falhas de integração | 3 | 2 |
| CPU do container (%) | 85 | 80 |
| Memória do container (%) | 85 | 80 |
| Reinícios de container | 3 | 2 |
| Erros de Lambda | 3 | 2 |
| Erros 5xx do API Gateway | 3 | 2 |
| Latência P95 do API Gateway (ms) | 2000 | 1200 |
| CPU do RDS (%) | 85 | 80 |
| Conexões do RDS | 70 | 60 |
| Armazenamento livre do RDS (GiB) | 3 | 5 |
| Erros PostgreSQL | 3 | 1 |
| Expiração de telemetria (s) | 1200 | 900 |
| Duração da violação (s) | 300 | 180 |

Sobrescreva com `alert_thresholds` no workspace quando necessário.

## Mapa NRQL do dashboard

### Negócio

| Widget | NRQL |
|---|---|
| OS criadas (24h) | `SELECT count(*) FROM OrdemServicoCriada WHERE environment = '<env>' SINCE 24 hours ago` |
| Volume diário | `SELECT count(*) FROM OrdemServicoCriada ... TIMESERIES 1 day SINCE 14 days ago` |
| Tempo médio por etapa | `SELECT average(duracaoMilissegundos) / 1000 FROM OrdemServicoStatusAlterado WHERE statusAnterior IN ('EM_DIAGNOSTICO', 'EM_EXECUCAO', 'AGUARDANDO_PAGAMENTO') FACET statusAnterior` |
| Transições de status | `SELECT count(*) FROM OrdemServicoStatusAlterado FACET statusAnterior, novoStatus TIMESERIES 1 day` |
| Falhas de processamento | `SELECT count(*) FROM OrdemServicoProcessamentoFalhou FACET codigoErro TIMESERIES` |
| Falhas de integração | `SELECT count(*) FROM IntegracaoExternaFalhou FACET operacao, codigoErro` |

### Aplicação

| Widget | NRQL |
|---|---|
| Throughput | `SELECT rate(count(*), 1 minute) FROM Transaction WHERE appName = '<app>' TIMESERIES` |
| Latência média e P95 | `SELECT average(duration) * 1000, percentile(duration, 95) * 1000 FROM Transaction ... TIMESERIES` |
| 4xx e 5xx | `SELECT filter(count(*), WHERE numeric(http.statusCode) >= 400 AND numeric(http.statusCode) < 500), filter(count(*), WHERE numeric(http.statusCode) >= 500) FROM Transaction ...` |
| Taxa de erro | `SELECT percentage(count(*), WHERE error IS true) FROM Transaction ...` |
| Uptime do healthcheck | `SELECT percentage(count(*), WHERE result = 'SUCCESS') FROM SyntheticCheck WHERE monitorName = '<cluster>-healthcheck'` |
| Logs correlacionados | `SELECT count(*) FROM Log WHERE cluster_name = '<cluster>' AND namespace_name = '<namespace>' AND level IN ('ERROR', 'WARN') FACET level` |

### Kubernetes

| Widget | NRQL |
|---|---|
| CPU por pod | `SELECT average(cpuUsedCores / cpuLimitCores) * 100 FROM K8sContainerSample WHERE clusterName = '<cluster>' AND namespaceName = '<ns>' FACET podName` |
| Memória por pod | `SELECT average(memoryWorkingSetUtilization) FROM K8sContainerSample ... FACET podName` |
| Réplicas | `SELECT latest(podsDesired), latest(podsAvailable) FROM K8sDeploymentSample WHERE deploymentName = '<deployment>'` |
| HPA | `SELECT latest(currentReplicas), latest(desiredReplicas), latest(maxReplicas) FROM K8sHpaSample ...` |
| Reinícios | `SELECT sum(restartCountDelta) FROM K8sContainerSample ...` |
| Pods e status | `SELECT latest(status), latest(isReady) FROM K8sPodSample ... FACET podName` |
| Eventos do cluster | `SELECT count(*) FROM InfrastructureEvent WHERE clusterName = '<cluster>' FACET event.reason` |

### Serverless e API Gateway

| Widget | NRQL |
|---|---|
| Invocações | `SELECT count(*) FROM Transaction WHERE appName IN (...) AND aws.lambda.arn IS NOT NULL FACET appName` |
| Duração média e P95 | `SELECT average(duration) * 1000, percentile(duration, 95) * 1000 FROM Transaction ...` |
| Erros e cold starts | `SELECT filter(count(*), WHERE aws.lambda.coldStart IS true), filter(count(*), WHERE error IS true) FROM Transaction ...` |
| Latência do gateway | `SELECT average(numeric(responseLatency)), percentile(numeric(responseLatency), 95), average(numeric(integrationLatency)) FROM Log WHERE logtype = 'api-gateway-access' AND environment = '<env>'` |
| Status do gateway | `SELECT count(*), filter(count(*), WHERE numeric(status) >= 400 ...), filter(count(*), WHERE numeric(status) >= 500) FROM Log ...` |
| Erros por função | `SELECT count(*) FROM TransactionError ... FACET appName, error.message` |

As Lambdas Java usam o modo APM serverless atual do agente New Relic, que publica invocações como `Transaction` e falhas como `TransactionError`. Os widgets do API Gateway usam os access logs técnicos encaminhados pela Lambda do repositório Auth. O payload é sanitizado por allowlist e não contém headers, tokens ou body; por isso o dashboard não depende de uma integração externa de métricas AWS.

### RDS PostgreSQL

A Lambda agendada do repositório Database publica `OficinaRdsSample` a cada cinco minutos. A página apresenta CPU, conexões, armazenamento e memória livres, latências, IOPS e contagens agregadas de erros/consultas lentas. Nenhum SQL ou texto bruto de log é encaminhado.

## Mapa das condições de alerta

| Condição | Base NRQL | Operador e limiar |
|---|---|---|
| `healthcheck_indisponivel` | `SyntheticCheck` com `result != 'SUCCESS'` | `>= synthetic_failures` por 5 min |
| `taxa_de_erro_alta` | `percentage(count(*), WHERE error IS true)` em `Transaction` | `> error_rate_percent` |
| `latencia_p95_alta` | `percentile(duration, 95) * 1000` em `Transaction` | `> latency_p95_ms` |
| `falha_processamento_os` | `count(*)` em `OrdemServicoProcessamentoFalhou` | `>= os_processing_failures` |
| `falha_integracao_recorrente` | `count(*)` em `IntegracaoExternaFalhou` | `>= integration_failures` por 10 min |
| `pod_indisponivel` | `podsDesired - podsAvailable` em `K8sDeploymentSample` | `> 0` por 5 min |
| `reinicios_repetidos` | `sum(restartCountDelta)` em `K8sContainerSample` | `>= container_restarts` por 10 min |
| `cpu_alta` | `cpuUsedCores / cpuLimitCores * 100` | `> container_cpu_percent` |
| `memoria_alta` | `memoryWorkingSetUtilization` | `> container_memory_percent` |
| `hpa_no_maximo` | `desiredReplicas - maxReplicas` em `K8sHpaSample` | `>= 0` por 10 min |
| `falha_lambda_recorrente` | `count(*)` em `TransactionError` filtrado pelas aplicações Lambda | `>= lambda_errors` por 10 min |
| `api_gateway_5xx` / `api_gateway_latencia` | access logs sanitizados em `Log` | limites de 5xx e P95 por ambiente |
| `rds_cpu_alta` / `rds_conexoes_altas` | `OficinaRdsSample` | acima do limite por ambiente |
| `rds_armazenamento_baixo` | `OficinaRdsSample.freeStorageBytes` | abaixo do limite em GiB |
| `rds_erros_postgresql` | última contagem sanitizada | `>= rds_postgres_errors` |
| `telemetria_ausente` | `count(*)` em `K8sClusterSample` com `expiration_duration` | `< 1` e violação na expiração do sinal |
| `telemetria RDS ausente` | `count(*)` em `OficinaRdsSample` com `expiration_duration` | abre violação se a coleta agendada parar |

O nome final de cada condição é `oficina-<environment> — <condição>`.

## Workspaces HCP Terraform

| Workspace sugerido | Diretório | Variable do GitHub Environment |
|---|---|---|
| `oficina-newrelic-homolog` | `observability/newrelic` | `TF_WORKSPACE_OBSERVABILITY_HOMOLOG` |
| `oficina-newrelic-production` | `observability/newrelic` | `TF_WORKSPACE_OBSERVABILITY_PRODUCTION` |

No workspace, defina **Terraform Working Directory** como
`observability/newrelic` e mantenha *Auto apply* desativado.

Execução local (somente plan):

```bash
cd observability/newrelic
export TF_CLOUD_ORGANIZATION=<organização>
export TF_WORKSPACE=oficina-newrelic-homolog
terraform init -input=false -lockfile=readonly
terraform plan -input=false
```

Validação estática, sem conta e sem credenciais:

```bash
cd observability/newrelic
terraform fmt -check -recursive
terraform init -backend=false -input=false -lockfile=readonly
terraform validate
```

## Monitor sintético

Enquanto o LoadBalancer não existir, `synthetic_monitor_enabled` permanece `false`. Depois do deploy da aplicação, reexecute `scripts/configure-environment.ps1` no repositório do backend. O script descobre a URL pública de `/actuator/health`, sincroniza HCP Terraform e GitHub Environment e habilita o monitor. Depois, revise o plan e autorize separadamente o apply do workspace de observabilidade.

Nenhuma URL é presumida ou documentada como verificada antes disso.
