# Definição de Papéis e Runtime (EKS Mapping)

Origem: `eks-mapping.md`.

Estratégia de segmentação da aplicação Laravel no cluster EKS usando **Single Image, Multiple Roles**: uma única imagem Docker (ECR) assume diferentes funções baseada no comando de inicialização.

## Arquitetura de execução

A aplicação é dividida em três camadas para isolamento de falhas, escalabilidade independente e performance.

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
        image: v4/laravel-app:latest
        command: ["php", "artisan", "octane:start", "--host=0.0.0.0"]
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"

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
        image: v4/laravel-app:latest
        command: ["php", "artisan", "horizon"]
        resources:
          limits:
            cpu: "500m"
            memory: "1Gi"
```
