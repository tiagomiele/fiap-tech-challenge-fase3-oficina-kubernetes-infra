# Validação do repositório de infraestrutura Kubernetes

Todas as validações abaixo são gratuitas: não criam recursos, não exigem
credenciais AWS e não acessam a conta New Relic.

## Terraform da infraestrutura

```bash
terraform fmt -check -recursive
terraform init -backend=false -input=false -lockfile=readonly
terraform validate
tflint --init && tflint --recursive
```

## Terraform da observabilidade

```bash
cd observability/newrelic
terraform fmt -check -recursive
terraform init -backend=false -input=false -lockfile=readonly
terraform validate
```

`terraform validate` não avalia recursos, então a configuração é verificada sem
account id nem API key. Para um `plan` local sem conta New Relic, use
`observability_enabled = false`.

## Charts Helm

```bash
./scripts/lint-cluster-addons.sh
```

O script renderiza `metrics-server` e `nri-bundle` para homologação e produção
com valores de placeholder, e falha se o nome do cluster não for propagado ou
se o Secret externo da license key não for referenciado.

## YAML e scripts

```bash
yamllint -c .yamllint.yml kubernetes .github/workflows
shellcheck scripts/*.sh
```

## Segurança

O CI executa:

- Trivy (`scan-type: config`) nas duas configurações Terraform;
- Gitleaks no histórico do repositório;
- verificação de chaves New Relic em texto claro;
- verificação de que `global.licenseKey` e `global.insightsKey` estão vazios.

## Plan remoto

O plan real depende das credenciais temporárias do AWS Academy e do workspace
HCP Terraform. Execute o workflow manual **Terraform plan** escolhendo o stack
(`infrastructure` ou `observability`) e o ambiente, e confirme:

- uma VPC com DNS habilitado;
- duas subnets públicas e duas privadas em zonas distintas;
- Internet Gateway e um NAT Gateway;
- EKS 1.34 e managed node group reutilizando somente a `LabRole`;
- nodes nas subnets privadas;
- logs do control plane habilitados;
- nenhuma criação de IAM role ou Access Entry;
- outputs de VPC, subnets e security group do EKS.

Para o stack de observabilidade, confirme o dashboard de quatro páginas, a
política de alertas, as doze condições e a ausência do monitor sintético
enquanto `health_check_url` estiver vazia.

O único NAT Gateway é uma decisão de custo para o Learner Lab. Produção real
exigiria um NAT por zona ou VPC endpoints equivalentes.

O apply não faz parte da validação estática e exige o workflow `Deploy` com
flags explícitas e aprovação do GitHub Environment.
