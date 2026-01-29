# Definição de Papéis e Runtime (EKS Mapping)

Origem: `eks-mapping.md`.

Estratégia de segmentação da aplicação Laravel no cluster EKS usando **Single Image, Multiple Roles**: uma única imagem Docker em **repositório privado (ECR)** assume diferentes funções baseada no comando de inicialização.

## Arquitetura de execução

A aplicação é dividida em três camadas para isolamento de falhas, escalabilidade independente e performance.

### Imagens Docker (repositório privado)

- Todas as imagens usadas pelos papéis (`Tracker`, `Worker`, `Manager`) são construídas e publicadas em um **repositório privado no Amazon ECR**.
- Os exemplos de `image:` nos manifests usam um nome genérico, mas em produção deve ser algo no formato:

  ```text
  <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/v4/laravel-app:<tag>
  ```

- O acesso a essas imagens é feito via credenciais/IAM do cluster (IRSA ou role dos nodes), **não são imagens públicas**.

### Papel: TRACKER (motor de cliques)

- **Runtime:** Laravel Octane (Swoole)
- **Comando:** `php artisan octane:start --host=0.0.0.0 --port=8000`
- **Gargalo crítico:** latência de rede e I/O de cache (Redis)
- **Escalabilidade:** HPA baseado em **RPS** e **CPU**
- **Fluxo:** recebe requisição -> consulta Redis -> dispara Job SQS (assíncrono) -> redireciona 302

### Papel: WORKER (processador de negócio)

- **Runtime:** PHP CLI (Laravel Horizon)
- **Comando:** `php artisan horizon`
- **Gargalo crítico:** conexões RDS e CPU para cálculos
- **Escalabilidade:** HPA baseado no **tamanho da fila SQS** (queue backlog)
- **Fluxo:** consome SQS -> valida ClickID -> calcula comissões -> persiste no RDS (conversions)

### Papel: MANAGER (painel administrativo)

- **Runtime:** PHP-FPM + Nginx
- **Comando:** `php-fpm` (porta 9000)
- **Gargalo crítico:** memória RAM (relatórios pesados)
- **Escalabilidade:** HPA baseado em **uso de memória**
- **Fluxo:** CRUDs -> configuração de campanhas -> relatórios/ERP

## Matriz de responsabilidades por componente

| Recurso | Tracker (Octane) | Worker (Horizon) | Manager (FPM) | Lambda (Postback) |
| :--- | :---: | :---: | :---: | :---: |
| **Leitura RDS** | Mínima | Alta | Alta | Nenhuma |
| **Escrita RDS** | Nenhuma | Alta | Média | Nenhuma |
| **Leitura Redis** | Crítica | Média | Baixa | Nenhuma |
| **Escrita SQS** | Alta | Baixa | Baixa | Crítica |
| **Consumo SQS** | Nenhum | Crítico | Nenhum | Nenhum |

## Healthcheck (liveness/readiness no EKS)

Objetivo: tornar **obrigatório** um healthcheck consistente em todas as apps para o Kubernetes usar nas probes de **liveness** e **readiness**, permitindo que o SRE use o `/health` como ponto de restauração e combine isso com HPA sem indisponibilidade.

- **Contrato de rota na aplicação**
  - Toda app deve expor `GET /health`.
  - A rota deve:
    - Retornar **HTTP 200** quando o processo está saudável.
    - Ser **extremamente leve e rápida** (sem relatórios, sem queries pesadas).
    - Opcionalmente fazer checks mínimos de dependências críticas:
      - Tracker: ping rápido no Redis.
      - Worker: usar `exec` probe com `php artisan horizon:status` (processo CLI não expõe HTTP).
      - Manager: ping leve no RDS (ex.: `SELECT 1`).

- **Liveness x Readiness (resumo)**
  - **Liveness probe**: responde se o pod **ainda está vivo**.
    - Se falhar repetidas vezes, o Kubernetes **mata o pod e recria**.
  - **Readiness probe**: responde se o pod **pode receber tráfego agora**.
    - Se falhar, o Kubernetes **tira o pod do Service** (não recebe tráfego), mas o pod continua rodando.

- **Exemplo de implementação do `/health` no Laravel**

```php
// routes/api.php (ou web.php, conforme padrão do serviço)
use Illuminate\Support\Facades\Route;
use Illuminate\Support\Facades\DB;

Route::get('/health', function () {
    // Check mínimo e rápido (opcional, ajustar por papel)
    try {
        DB::select('SELECT 1'); // Manager pode usar isso para validar conexão com RDS
    } catch (\Throwable $e) {
        return response()->json(['status' => 'error'], 500);
    }

    return response()->json(['status' => 'ok'], 200);
});
```

> Observação: para o Tracker você pode trocar o check de DB por um ping simples no Redis; para o Manager, usar o check de DB como mostrado acima.

- **Worker (Horizon) - processo CLI**

Como o Worker roda `php artisan horizon` (processo CLI sem servidor HTTP), o healthcheck usa **`exec` probe** em vez de `httpGet`:

```bash
# O comando do Horizon retorna código de saída 0 se estiver rodando
php artisan horizon:status
```

Ou, alternativamente, verificar se o processo principal está vivo:

```bash
# Verifica se o processo PHP do Horizon está rodando
pgrep -f "artisan horizon" > /dev/null
```

- **Manager (Nginx + PHP-FPM) - onde entra o FPM**

No papel **Manager**, quem “atende HTTP” é o **Nginx** (porta 80/443). O Nginx repassa as requests PHP para o **PHP-FPM** (normalmente na porta 9000, dentro do pod).
Por isso, o healthcheck recomendado é **HTTP no Nginx** (ex.: `GET /health`), pois valida o caminho completo **Nginx → FPM → Laravel** (e ainda pode checar RDS).

## Exemplo de deployment (mesma imagem, papéis diferentes)

```yaml
# Deployment do TRACKER
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-tracker
spec:
  template:
    spec:
      containers:
      - name: app
        image: <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/v4/laravel-app:latest
        command: ["php", "artisan", "octane:start", "--host=0.0.0.0"]
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3

---

# Deployment do WORKER
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-worker
spec:
  template:
    spec:
      containers:
      - name: app
        image: <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/v4/laravel-app:latest
        command: ["php", "artisan", "horizon"]
        resources:
          limits:
            cpu: "500m"
            memory: "1Gi"
        livenessProbe:
          exec:
            command:
            - php
            - artisan
            - horizon:status
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - php
            - artisan
            - horizon:status
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

---

# Deployment do MANAGER (Nginx + FPM)
# Observação: probe HTTP deve apontar para a porta do Nginx (80), e o /health deve bater no Laravel via FPM.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-manager
spec:
  template:
    spec:
      containers:
      - name: app
        image: <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/v4/laravel-app:latest
        command: ["php-fpm"]
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: "1"
            memory: "1Gi"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
```
