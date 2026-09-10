# Arquitetura Kubernetes

## Escopo

Este repositório é proprietário da rede compartilhada e do cluster EKS. Seus outputs não sensíveis são consumidos explicitamente pelos repositórios de banco e autenticação.

O diagrama desta visão está em `docs/diagrams/infrastructure-observability.mmd`.
O diagrama completo da solução permanece no repositório da aplicação e não é
duplicado aqui.

## Componentes

- VPC em duas zonas de disponibilidade;
- subnets públicas e privadas;
- Internet Gateway e um NAT Gateway compatível com o orçamento do Learner Lab;
- cluster EKS com logs de control plane;
- managed node group privado reutilizando a `LabRole`;
- add-ons VPC CNI, CoreDNS e kube-proxy;
- outputs necessários ao RDS e ao deploy da aplicação;
- Metrics Server e `nri-bundle` instalados por Helm depois da criação do
  cluster;
- dashboards, alertas e monitor sintético do New Relic versionados em
  `observability/newrelic`.

## Estados

Homologação e produção utilizam estados HCP Terraform separados. O state nunca é armazenado no Git e não reutiliza o state combinado da Fase 2.

A observabilidade tem workspaces próprios, também separados por ambiente, para
que a API key do New Relic não circule no state da infraestrutura AWS.

## Camadas de execução

| Camada | Mecanismo | Momento |
|---|---|---|
| Rede e EKS 1.34 | Terraform na raiz | primeiro |
| Metrics Server e `nri-bundle` | Helm idempotente (`scripts/deploy-cluster-addons.sh`) | depois do cluster |
| Aplicação, Service e HPA | repositório do backend | depois dos add-ons |
| Dashboards, alertas e sintético | Terraform em `observability/newrelic` | por último |

## Dependências

A infraestrutura do banco consumirá VPC e subnets privadas. A autenticação serverless consumirá rede e endpoints necessários. A aplicação publicará sua imagem separadamente.
