# Melhoras práticas de DevOps

## Índice

- [Melhoras práticas de DevOps](#melhoras-práticas-de-devops)
  - [Índice](#índice)
  - [Introdução](#introdução)
  - [IaC](#iac)
    - [IaC e CI/CD](#iac-e-cicd)
    - [Segurança do State File Terraform](#segurança-do-state-file-terraform)
  - [Ambientes](#ambientes)
  - [Containers](#containers)
    - [Versionamento e tags](#versionamento-e-tags)
    - [Opções para runtime de containers](#opções-para-runtime-de-containers)
    - [Segurança de containers](#segurança-de-containers)
    - [Registry](#registry)
  - [Testes automatizados](#testes-automatizados)
  - [Segredos](#segredos)
  - [DR (Disaster Recovery)](#dr-disaster-recovery)
    - [Bancos de dados](#bancos-de-dados)
    - [File systems](#file-systems)
    - [Object storage](#object-storage)
    - [Drills de recuperação](#drills-de-recuperação)
  - [Escalabilidade](#escalabilidade)
    - [Escalabilidade de bancos de dados](#escalabilidade-de-bancos-de-dados)
  - [Monitoração](#monitoração)
    - [Métricas](#métricas)
    - [Logs](#logs)
    - [Traces](#traces)
    - [Alertas](#alertas)
    - [Dashboard](#dashboard)
    - [Stack convencional](#stack-convencional)


## Introdução

Este é um resumo do meu aprendizado de melhores práticas para DevOps, baseado na minha experiência prática e aprendizado de campo.

## IaC

Toda a infraestrutura deve poder ser recriada do zero a partir de código, garantindo que seja idêntica à existente.

Terraform é a ferramenta recomendada. Motivo: maior ecossistema, com maior cobertura de diferentes providers.

O código IaC deve ser mantido em um repositório Git. Isto garante rastreabilidade de mudanças.

Opção: GitOps. Caso seja usado Kubernetes, parte do IaC deve residir em arquivos mantidos em Git, que descrevem o estado desejado em YAML, e sincronizados com o cluster através de ferramentas de GitOps como Argo CD ou Flux.

Todo o código IaC (Terraform, manifestos GitOps, etc.) deve ser mantido em repositórios Git separados do código da aplicação.

Isto garante apartar acessos uma vez que os desenvolvedores podem ou não ter acesso à configuração de IaC dos diferentes ambientes. Além disso, workflows (exemplo, GitHub Actions) podem ter efeitos colaterais indesejados se executados em IaC ou código da aplicação.

### IaC e CI/CD

Para ferramentas como Terraform, desejável que os scripts sejam executados via esteiras CI/CD. Isto garante consistência (sempre mesmo ambiente de execução) e rastreabilidade (resultado fica registrado no workflow). É possível usar ferramentas de simulação como `terraform plan`/`validate` para execução em PRs antes do merge e execução do `terraform apply` no branch principal.

Em GitOps (Argo CD etc.) o fluxo de CI/CD começa no repositório da aplicação, onde o deploy executa um `git push` dos manifestos para o repositório de GitOps e o Argo CD sincroniza os manifestos com o cluster.

### Segurança do State File Terraform

O Terraform guarda num state file o estado atual para comparação com o existente quando calcula as mudanças a serem aplicadas. Este arquivo pode frequentemente conter dados sensíveis como senhas, tokens, credenciais etc. É fundamental que a esteira de CI/CD configure o Terraform para armazenar o state file de forma segura. Recomendação: usar backend S3. As credenciais podem ser configuradas via GitHub Secrets, GitLab Secrets, etc. conforme [Segredos](#segredos).

## Ambientes

Além do ambiente de produção, deve ser criado um ambiente de Staging. A principal finalidade do ambiente de Staging é aplicar mudanças a um ambiente idêntico ao de produção para garantir as mudanças não quebrem o ambiente de produção. Conforme [IaC](#iac), em todos os ambientes as mudanças devem transitar pelo Git.

O ambiente de Staging deve ser mantido sempre idêntido ao de produção, com a exceção de diferenças de dimensionamento (por exemplo, menor CPU, memória, armazenamento, etc.) mas sempre com os mesmos componentes e versões. Em particular, por exemplo, se for usada replicação ou load balancing em produção, o ambiente de Staging deve ser configurado da mesma forma.

Não se deve usar o ambiente de Staging para mudanças frequentes como num ambiente de desenvolvimento. Para isto, criar um ambiente de Dev. Neste ambiente deverão ser feitas as mudanças de dia a dia que fazem parte do processo de desenvolvimento do produto.

Ao aplicar mudanças no ambiente de Staging, deve-se executar uma suite de testes funcionais que cubram todas as jornadas dos diferentes perfis de usuários, testes de performance etc. para garantir que as mudanças não quebrem a aplicação. Caso o teste passe, a mudança deve ser promovida para o ambiente de produção em janelas de horário pre-estabelecidas. Caso o teste falhe, a mudança no ambiente de Staging deve ser revertida para que ele permaneça igual ao de produção.

Para respositórios Git de [IaC](#iac), não usar branches separados para ambiente. Segmentar ambientes por pastas ou por repositórios separados. Branches são usados para merges e não há motivo para merge entre ambientes. Deve-se usar um branch único (trunk based). Branches efêmeros podem ser criados apenas para PRs de infraestrutura e processos de aprovação mas a aplicação da definição IaC ao ambiente deve ser feita sempre a partir do branch principal (`main`).

## Containers

Todo o código deve ser containerizado. Isto garante que ele seja executado de forma consistente, seja localmente ou nos diferentes ambientes. A definição do Dockerfile deve ser mantida junto com o código da aplicação.

### Versionamento e tags

Via CI/CD (GitHub Actions, GitLab CI, etc.), automatizar a criação de tags de imagens quando tags do repositório Git forem criadas. Assim as imagens ficam com tags sincronizadas com o código.

Usar o conceito de Semantic Versioning (<https://semver.org/>) para a criação de tags do repositório.

Usar a convenção <https://www.conventionalcommits.org/> para commit messages. É possível, via GitHub Actions, automatizar a criação de tags de imagens e Changelogs quando commits forem criados. Exemplos:

- <https://github.com/marketplace/actions/semver-conventional-commits>
- <https://github.com/marketplace/actions/changelog-from-conventional-commits>

Nas definições de [IaC](#iac), as imagens devem ser referenciados por um tag fixo (pinned) e não por um tag dinâmico (latest). É possível usar latest em ambiente de Dev se desejado. Nos ambientes de Staging e Produção, as imagens devem ser referenciados pelo mesmo tag, exceto quando for necessário atualizar a imagem para uma versão mais recente (primeiro em Staging, depois em Produção).

### Opções para runtime de containers

Existem duas opções para runtime de containers:

- No cloud provider, exemplos: AWS ECS, Google Cloud Run, etc.
- Via um cluster Kubernetes, exemplo: AWS EKS, Google Kubernetes Engine, etc.

A escolha deve ser feita considerando os skills da equipe de DevOps. A gestão de um cluster Kubernetes, mesmo quando a partir de ofertas gerenciadas como EKS e GKE requer uma maturidade maior do time. Motivos:

- A configuração default do cluster é entregue sem hardening necessário para a produção como segmentação de rede, limitação de escopo de service accounts, etc.
- Gestão de upgrades do cluster e seus diferentes componentes (ingress, cert-manager, etc.) de forma continuada.
- Criação da infraestrutura interna de monitoração, logging, etc.

### Segurança de containers

Usar uma ferramenta como, por exemplo, <https://trivy.dev/> para scanear as imagens quanto a vulnerabilidades embedded na esteira de CI/CD, junto com a criação da imagem a partir do Dockerfile.

Usar bases de SO "magras" como Alpine Linux ou distroless para reduzir a superfície de ataque da aplicação.

### Registry

Imagens geradas precisam de um registry para armazenamento. Usar o registry do GitHub, GitLab, etc. é mais prático. Proteger o registry com autenticação por tokens, que devem ser sincronizados por [Segredos](#segredos).

## Testes automatizados

Para cada API endpoint devem ser criados testes unitários que cubram todos os métodos HTTP (GET, POST, PUT, DELETE, etc.) e todos as possíveis combinações de parâmetros de entrada, incluindo casos de dados inválidos, inexistentes ou defaults.

Criar também testes funcionais fim a fim (E2E) onde a entrada de dados é feita simulando as diferentes jornadas dos diferentes perfis de usuários, a partir de interação headless com o navegador e garantindo que os dados persistidos no banco de dados são os esperados de acordo com cada caso.

Incluir os testes automatizados na esteira de CI/CD para garantir que eles passam antes da promoção para o ambiente de Staging.

Tais testes também devem ser usados em aprovações de PRs, idealmente de forma também automática.

## Segredos

Nunca manter segredos em código fonte. Opções para manutenção de segredos:

- Sistema de gerenciamento de segredos, como, por exemplo, AWS Secrets Manager, etc.
- Segredos no GitHub Secrets, GitLab Secrets, etc.: ideal para segredos de IaC como credenciais de cloud provider ou de acesso ao registry de imagens de containers.
- Segredos criptografados em GitOps, exemplo: [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)

Via [IaC](#iac), configurar os containers para que os segredos sejam injetados como variáveis de ambiente e acessa-los dessa forma via código da aplicação.

Montar um esquema de rotação dos segredos, usando a automação do IaC para sincronizar com as aplicações.

## DR (Disaster Recovery)

Dados persistentes podem vir de bancos de dados, file systems ou object storage.

### Bancos de dados

Backups full de bancos de dados devem ser feitos segundo um agendamento automático e uma política de retenção também automatizada. Por exemplo:

- 3 últimos backups diários
- 2 últimos backups semanais
- 2 últimos backups mensais
- 5 últimos backups anuais

O destino dos backups deve ser um storage com presença em ao menos uma localidade geográfica diferente do banco de dados. Por exemplo, um bucket S3 multi-zonas.

O conteúdo dos backups deve ser protegido por credenciais separadas daquelas usadas dentro do ambiente de bancos de dados. Por exemplo, com o uso de [versionamento em buckets S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html), pode-se configurar uma credencial com acesso somente à versão corrente (para o agente de backup residente no banco ou no ambiente de produção) e outra, separada e inacessível pelo ambiente de produção, com acesso a versões anteriores (para a recuperação).

Desejável complementar com backups de logs, por exemplo, com log shipping a cada 5 minutos. Com isso o RPO (Recovery Point Objective) pode ser reduzido para minutos.

### File systems

Não é desejável o uso de file systems para dados persistentes, pois compromete arquiteturas modernas de software como [12 Factor App](https://12factor.net/). Se inevitável, seguir conceitos semelhantes aos de [Bancos de dados](#bancos-de-dados). Para arquivos de mídia como imagens, vídeos, etc., usar object storage.

### Object storage

Configurar object storage como multi-zonas para garantir alta disponibilidade. Usar [versionamento](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html) para garantir que os dados persistidos sejam protegidos contra perda ou corrupção e configurar credenciais separadas para o ambiente de execução e para o ambiente de backup, de modo que o ambiente de produção não tenha acesso a versões anteriores do bucket, garantindo maior proteção para eventual recuperação.

### Drills de recuperação

Deve-se executar periodicamente (sugestão: trimestral ou semestral) um drill de recuperação. Pode-se usar o ambiente de Staging para isto. Executar os testes funcionais de promoção à produção para validar a recuperação.

## Escalabilidade

Deve-se usar a arquitetura [12 Factor App](https://12factor.net/) para garantir que a aplicação seja escalável. Os seguintes componentes são usados para montar esta arquitetura:

- Load balancer: distribui acesso entre réplicas
- Autoscaling: usa um trigger como fila, requests por segundo, etc. para aumentar/diminuir automaticamente o número de réplicas

### Escalabilidade de bancos de dados

Para bancos de dados, usar réplicas de leitura para distribuir a carga. Quando se tornar necessário, usar componentes de otimização de conexão, como [PgBouncer](https://www.pgbouncer.org/) para PostgreSQL ou [ProxySQL](https://proxysql.com/) para MySQL.

Tais ferramentas devem ser configuradas via [IaC](#iac) usando os recursos do provedor de cloud ou, se rodando bancos de dados em clusters Kubernetes, via GitOps, por intermédio de controladores como [CloudNativePG](https://cloudnative-pg.io/).

## Monitoração

O stack de monitoração deve incluir:

- Métricas
- Logs
- Traces
- Alertas
- Dashboards

### Métricas

Métricas são indicadores como CPU, memória, requests por segundo, etc. que são exportadas em amostras temporais para posterior análise, visualização em gráficos e disparos de alertas.

Deve-se configurar, via [IaC](#iac), a exposição de métricas usando capacidades nativas do provedor de cloud juntamente com métricas expostas pela aplicação. Métricas devem ser expostas em um padrão como Prometheus ou OpenTelemetry.

### Logs

Configurar a aplicação para exportar logs via `stdout` e `stderr` em consonância com [12 Factor App](https://12factor.net/logs). A infraestrutura deve automaticamente capturar e processar tais logs de forma transparente para a aplicação.

### Traces

Traces são usados para mapear requsições à medida que elas fluem através dos diferentes componentes da aplicação e bancos de dados.

Deve-se configurar, via [IaC](#iac), a captura e processamento de traces usando capacidades nativas do provedor de cloud juntamente com traces expostas pela aplicação. Traces devem ser expostas em um padrão como OpenTelemetry.

### Alertas

A configuração de alertas deve ser feita via [IaC](#iac) usando capacidades nativas do provedor de cloud juntamente com alertas expostos pela aplicação. 

Tal configuração deve incluir os canais desejados para recebimento de alertas de acordo com a ferramenta em uso pela empresa, como Slack, Teams, etc. e devem ser disponibilizados também pelo Dashboard de monitoração.

### Dashboard

Um dashboard de monitoração deve ser disponibilizado para visualização dos diferentes indicadores de monitoração. O padrão universal é o [Grafana](https://grafana.com/).

### Stack convencional

Recomenda-se usar as ferramentas mais maduras e consolidadas. Um exemplo de stack "cloud-native" para uso em Kubernetes:

- Prometheus para métricas
- Alertmanager para alertas
- Grafana para dashboards
- Loki para logging
- Tempo para traces

Provedores de cloud geralmente oferecem soluções equivalentes e integradas dentro de seu próprio ecossistema. Por exemplo, o runtime de containers ECS da AWS oferecem integração nativa de logs via CloudWatch e métricas/traces via OpenTelemetry.

Configurar todo o wiring de observabilidade entre os componentes da aplicação via [IaC](#iac).

Para Kubernetes, recomenda-se usar Helm Charts com bundles que incluem os componentes já de forma integrada. Exemplos (componentes podem ser usados em conjunto ou isoladamente):

- [Kube Prometheus Stack](https://github.com/prometheus-community/helm-charts/charts/kube-prometheus-stack)
- [Grafana K8s Monitoring](https://github.com/grafana/k8s-monitoring-helm)
