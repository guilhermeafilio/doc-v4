# Estratégia de Build e Deployment

A solução utiliza uma abordagem de **Imagem Única (Single Source of Truth)** com **Segmentação de Serviços**.

## O artefato (Docker image)

Uma única imagem Docker é gerada no pipeline de CI/CD e enviada para o **Amazon ECR**. Esta imagem contém:

- Código fonte completo (Monorepo).
- Extensões PHP necessárias para FPM e Octane (Swoole), PHP CLI.
- Binários do Nginx e dependências de runtime.

## Separação de instâncias no EKS

No cluster EKS, dividimos a aplicação em duas frentes de deploy distintas para isolamento de recursos e escalabilidade:

### Aplicação TRACKER (Runtime: Octane)

Focada exclusivamente em redirecionamento de links e rastreamento de cliques.

- **Service Type:** High-Traffic Edge API.
- **Entrypoint:** `php artisan octane:start`.
- **Isolamento:** Possui seu próprio conjunto de Pods e HPA. Erros no painel administrativo não afetam a capacidade de redirecionamento.
- **Conexões:** Otimizada para conexões persistentes com Redis.

### Aplicação MANAGEMENT (Runtime: Monolito)

Focada em gestão de dados, interface de usuário e processamento de regras de negócio.

Esta unidade de deploy gerencia dois processos internos:

1. **Manager (Web):** Interface via `php-fpm`.
2. **Worker (Queue):** Processamento de filas via `php artisan horizon`.

- **Entrypoint:** `php-fpm` (pods web) ou `php artisan horizon` (pods de fila).
- **Conexões:** Conecta-se ao RDS para operações de alta consistência e manipulação do ERD.

**Workers (filas):** A decisão de arquitetura é **híbrida** (ver [Divisão de Workers por Fila](./worker-divisao-filas.md)): existem **dois** Deployments de worker:
- **deploy-worker:** pool base que processa as filas **track** e **conversion** (`HORIZON_ENV=default`).
- **deploy-worker-track:** PODs que processam **apenas** a fila **track** (`HORIZON_ENV=track`), para escalar em épocas de pico de cliques (Black Friday, campanhas). Em regime normal pode ter réplicas mínimas (ex.: 0 ou 1); em pico sobe-se as réplicas ou usa-se HPA baseado no backlog da fila track.

## Matriz de deployment

| Componente | Imagem Docker | Deployment K8s | Configuração PHP | Objetivo |
| :--- | :--- | :--- | :--- | :--- |
| **Tracker** | `app:latest` | `deploy-tracker` | Octane/Swoole | Clique -> Redirecionamento |
| **Manager** | `app:latest` | `deploy-manager` | PHP-FPM | CRUDs / Dashboard |
| **Worker (base)** | `app:latest` | `deploy-worker` | PHP-CLI (Horizon env `default`) | Processar filas track + conversion |
| **Worker (track pico)** | `app:latest` | `deploy-worker-track` | PHP-CLI (Horizon env `track`) | Processar só fila track; escalar em pico (Black Friday, etc.) |

## Benefícios da separação

- **Independência de escalonamento:** Escalar o `Tracker` sem escalar o `Manager`; escalar workers de **track** (deploy-worker-track) em pico sem aumentar o pool base (deploy-worker).
- **Segurança de memória:** Problemas no Octane não derrubam fila (Worker) nem acesso (Manager).
- **Updates granulares:** Rolling update do Tracker sem indisponibilizar o painel.
- **Pico de cliques:** Black Friday e campanhas: sobe réplicas do **deploy-worker-track** (ou HPA na fila track) sem alterar o processamento de conversion.
