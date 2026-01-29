# Divisão de Workers (PODs) por Fila: Track e Conversion

Documento que define como especificar e dividir os PODs do **Worker** (Laravel Horizon) considerando as duas filas SQS processadas por ele: **track** (cliques) e **conversion** (postbacks/conversões).

## Contexto das filas

| Fila | Origem | Conteúdo | Gargalo típico |
| :--- | :--- | :--- | :--- |
| **Track** | Tracker (Octane) → SQS | Jobs `LogClick` (IP, UA, ClickId) | Volume alto, escrita RDS + possível uso de Redis |
| **Conversion** | Lambda (Postback) → SQS | Dados de conversão (click_id, amount, order_id, status) | Cálculo de comissões, consistência RDS, volume geralmente menor |

Ambas são consumidas pelo mesmo papel **Worker** (`php artisan horizon`), mas têm perfis de carga e criticidade diferentes, o que justifica formas distintas de dimensionar e escalar os PODs.

## Decisão adotada: híbrido (pool base + track dedicado para pico)

A arquitetura adotada é **híbrida**: um pool base processa as duas filas; um segundo Deployment processa **apenas** a fila **track**, para ser escalado em épocas de pico de cliques (Black Friday, campanhas, etc.), sem aumentar indefinidamente o pool base.

### Quantos Deployments de worker existem

| Deployment K8s | Filas processadas | Uso |
| :--- | :--- | :--- |
| **deploy-worker** | track + conversion | Pool base: processamento contínuo das duas filas. |
| **deploy-worker-track** | apenas track | Capacidade extra de track: escalar réplicas em picos (Black Friday, etc.). |

- **Total:** 2 Deployments de worker.
- Em operação normal, o **deploy-worker** pode ter réplicas mínimas (ex.: 2); o **deploy-worker-track** pode ficar em 0 ou 1. Em pico, aumenta-se as réplicas do **deploy-worker-track** (e/ou HPA baseado no backlog da fila track).

### Como aparece no `config/horizon.php`

O Laravel Horizon usa **ambientes** (ou blocos por “perfil”) para definir quais filas cada tipo de POD consome. Exemplo de estrutura:

- **Ambiente `default`** (usado pelo **deploy-worker**): supervisors que escutam as filas **track** e **conversion**. Um único conjunto de processos trata as duas filas.
- **Ambiente `track`** (usado pelo **deploy-worker-track**): supervisors que escutam **apenas** a fila **track**. Esses PODs existem para absorver pico de cliques.

O POD escolhe o ambiente via variável de ambiente (ex.: `HORIZON_ENV=default` ou `HORIZON_ENV=track`). O comando de início do Horizon deve usar esse valor, por exemplo: `php artisan horizon` com a app lendo `HORIZON_ENV` e passando para o Horizon como ambiente (ou equivalente no `config/horizon.php`).

Exemplo mínimo em `config/horizon.php`:

```php
'environments' => [
    'default' => [
        'worker-supervisor' => [
            'connection' => 'sqs',
            'queue' => ['track', 'conversion'],
            'processes' => 3,
            // ...
        ],
    ],
    'track' => [
        'track-supervisor' => [
            'connection' => 'sqs',
            'queue' => ['track'],
            'processes' => 2,
            // ...
        ],
    ],
],
```

Assim, PODs com `HORIZON_ENV=default` processam track e conversion; PODs com `HORIZON_ENV=track` processam só track.

### Como aparece nos manifests do EKS

- **deploy-worker:** mesmo `image` e `command: ["php", "artisan", "horizon"]`; env `HORIZON_ENV=default` (ou omitido, se default for o valor padrão). Liveness/readiness com `php artisan horizon:status`.
- **deploy-worker-track:** mesma `image` e `command`; env **`HORIZON_ENV=track`**. Recursos e HPA podem ser ajustados para carga de track (ex.: mais CPU por POD). Réplicas mínimas podem ser 0 e o HPA sobe PODs quando o backlog da fila track passar de um limite (útil para Black Friday).

Exemplo YAML da abordagem adotada (dois Deployments):

```yaml
# Pool base: track + conversion
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-worker
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: app
        image: <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/v4/laravel-app:latest
        command: ["php", "artisan", "horizon"]
        env:
        - name: HORIZON_ENV
          value: "default"
        resources:
          limits:
            cpu: "500m"
            memory: "1Gi"
        livenessProbe:
          exec:
            command: ["php", "artisan", "horizon:status"]
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command: ["php", "artisan", "horizon:status"]
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
---
# Track dedicado: só fila track (escalar em pico — Black Friday, etc.)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-worker-track
spec:
  replicas: 0   # ou 1; HPA pode subir conforme backlog da fila track
  template:
    spec:
      containers:
      - name: app
        image: <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/v4/laravel-app:latest
        command: ["php", "artisan", "horizon"]
        env:
        - name: HORIZON_ENV
          value: "track"
        resources:
          limits:
            cpu: "500m"
            memory: "1Gi"
        livenessProbe:
          exec:
            command: ["php", "artisan", "horizon:status"]
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command: ["php", "artisan", "horizon:status"]
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
```

Resumo para quem implementa ou opera: **2 Deployments**; **deploy-worker** = track + conversion; **deploy-worker-track** = só track, para subir em pico de cliques.

---

## Estratégias de divisão (referência)

### 1. Pool único (um único Deployment)

Um único Deployment de Worker processa as duas filas no mesmo conjunto de PODs. O Laravel Horizon, via `config/horizon.php`, define **supervisors** e **processes** por ambiente; todos os PODs usam a mesma imagem e o mesmo comando.

- **Vantagem:** Operação simples, menos Deployments para manter.
- **Desvantagem:** Não dá para escalar track e conversion de forma independente nem dar recursos (CPU/memória) diferentes por tipo de fila.
- **Uso recomendado:** Carga equilibrada entre as duas filas e requisitos de recurso similares.

**Como especificar:** Um único `laravel-worker` Deployment (como no `eks-mapping-roles.md`). A divisão do trabalho entre track e conversion fica apenas na configuração do Horizon (quais filas cada supervisor escuta e quantos processos).

---

### 2. Deployments dedicados (track-workers × conversion-workers)

Dois Deployments no EKS: um para PODs que processam **apenas** a fila de track e outro para PODs que processam **apenas** a fila de conversion. Mesma imagem Docker, **comando ou variáveis de ambiente diferentes** para que o Horizon (ou o entrypoint) escute só as filas desejadas.

- **Vantagem:** Escalonamento independente (HPA por backlog da fila track vs fila conversion), recursos e limites (CPU/memória) diferentes por tipo, isolamento de falhas entre as duas cargas.
- **Desvantagem:** Dois Deployments, dois HPAs e necessidade de configurar Horizon (ou wrapper) por “tipo” de worker.
- **Uso recomendado:** Volumes ou picos muito diferentes entre track e conversion, ou quando conversion exige mais CPU/memória por job.

**Como especificar:**

- **No Kubernetes:** dois Deployments, por exemplo `laravel-worker-track` e `laravel-worker-conversion`, com:
  - **Env** (ou ConfigMap/Secret) indicando o “perfil” do worker, por exemplo:
    - `WORKER_QUEUES=track` → só processa fila(s) de track
    - `WORKER_QUEUES=conversion` → só processa fila(s) de conversion
  - **Recursos** (requests/limits) e **HPA** distintos para cada Deployment (ex.: HPA do track baseado em métrica da fila track; do conversion na fila conversion).
- **No Laravel/Horizon:**
  - Opção A: `config/horizon.php` define ambientes (ex.: `track`, `conversion`); o comando ou um script de entrypoint lê `WORKER_QUEUES` e inicia o Horizon no ambiente correspondente, por exemplo:
    - `php artisan horizon --env=track`
    - `php artisan horizon --env=conversion`
  - Opção B: Um único ambiente no Horizon com vários supervisors; o entrypoint do container usa `WORKER_QUEUES` para gerar ou escolher um `horizon.php` que ativa apenas os supervisors daquelas filas (track ou conversion).

Exemplo mínimo de **divisão em dois Deployments**:

```yaml
# Deployment: Workers só para fila TRACK
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-worker-track
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: app
        image: <ECR_URI>/v4/laravel-app:latest
        command: ["php", "artisan", "horizon"]
        env:
        - name: HORIZON_ENV
          value: "track"   # ou WORKER_QUEUES=track, conforme config da app
        resources:
          limits:
            cpu: "500m"
            memory: "1Gi"
        livenessProbe:
          exec:
            command: ["php", "artisan", "horizon:status"]
          # ... (initialDelaySeconds, periodSeconds, etc.)
---
# Deployment: Workers só para fila CONVERSION
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-worker-conversion
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: app
        image: <ECR_URI>/v4/laravel-app:latest
        command: ["php", "artisan", "horizon"]
        env:
        - name: HORIZON_ENV
          value: "conversion"
        resources:
          limits:
            cpu: "1000m"
            memory: "1Gi"
        livenessProbe:
          exec:
            command: ["php", "artisan", "horizon:status"]
          # ...
```

O valor de `HORIZON_ENV` (ou equivalente) deve ser lido pela aplicação para escolher qual bloco de configuração do Horizon usar (qual conjunto de filas/supervisors).

---

### 3. Híbrido (pool compartilhado + fila crítica dedicada) — **adotado**

Um Deployment processa as duas filas (track + conversion) e um segundo Deployment processa **apenas** uma fila (no nosso caso, **track**), para dar capacidade extra em picos sem duplicar todo o pool.

- **Como especificar:** Pool base com Horizon em ambiente `default` (track + conversion); Deployment dedicado com `HORIZON_ENV=track` (só track). Ver seção “Decisão adotada” acima.
- **Uso adotado:** Picos de **cliques** (Black Friday, campanhas): sobe réplicas do **deploy-worker-track**. O pool base (**deploy-worker**) segue atendendo track e conversion em regime normal.

---

## Resumo: como especificar a divisão

| Abordagem | Onde se especifica | O que definir |
| :--- | :--- | :--- |
| **Pool único** | `config/horizon.php` | Supervisors/processes por ambiente; todos os PODs iguais. |
| **Deployments dedicados** | K8s: Deployments + env (ex. `HORIZON_ENV` / `WORKER_QUEUES`) | Dois Deployments, recursos e HPA por fila. App: Horizon por ambiente ou por conjunto de filas. |
| **Híbrido** *(adotado)* | K8s: **deploy-worker** + **deploy-worker-track**; `config/horizon.php` ambientes `default` e `track` | Pool base (track + conversion); Deployment extra só track para pico (Black Friday, etc.). |

## Escalonamento (HPA) por fila

Quando houver Deployments dedicados, o HPA pode ser baseado em métricas por fila:

- **Worker track:** métrica de backlog da fila SQS de track (ex.: `ApproximateNumberOfMessagesVisible`), ou custom métrica derivada dela.
- **Worker conversion:** idem para a fila SQS de conversion.

Assim, os PODs de track e de conversion escalam de forma independente conforme o tamanho da respectiva fila.

---

**Referências:** [EKS Mapping (papéis e runtime)](./eks-mapping-roles.md), [Build e Deploy](./build-e-deploy.md), [Sequência Clique (Tracker)](../Fluxos/sequencia-clique-tracker.md), [Sequência Conversão (Postback)](../Fluxos/sequencia-conversao-postback.md).
