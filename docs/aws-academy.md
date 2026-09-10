# AWS Academy

## Restrições consideradas

- credenciais temporárias;
- reutilização obrigatória da `LabRole` quando IAM estiver bloqueado;
- possível bloqueio de criação de EKS Access Entry;
- recursos destruídos ou indisponíveis após expiração do laboratório;
- necessidade de renovar credenciais temporárias em todos os executores.

## Estratégia

- renovar uma única vez por sessão com o script central, que atualiza AWS CLI, HCP e GitHub;
- validar `aws sts get-caller-identity` no início da pipeline;
- reutilizar a `LabRole` para o cluster e o managed node group, sem criar IAM roles;
- manter Terraform idempotente e reproduzível;
- separar estados de homologação e produção;
- executar destroy após coleta das evidências;
- não aplicar automaticamente quando a sessão estiver expirada;
- usar um único NAT Gateway como compromisso de custo do laboratório;
- manter bootstrap de administrador do criador do cluster sem Access Entry;
- manter endpoint público autenticado para operadores e runners externos ao VPC; a exceção Trivy está registrada em `.trivyignore` e os CIDRs devem ser restringidos quando a origem for estável.
