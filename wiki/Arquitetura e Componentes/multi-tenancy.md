# Estratégia de Multi-tenancy (Isolation Layer)

Origem: `multi-tenancy.md`.

Como a aplicação garante isolamento lógico entre diferentes clientes (tenants), usando `Enterprise` como âncora principal.

## Definição do tenant

Abordagem de **Banco de Dados Compartilhado com Esquema de Isolamento por Coluna (Shared Schema)**.

- **Entidade raiz:** `Enterprise`
- **Chave de isolamento:** `enterprise_id` (presente em tabelas de recursos como `Campaigns`, `LinkShortener` e `Conversions`)

## Componentes (app/Infrastructure/MultiTenancy)

### Contexto global (`TenantContext.php`)

Serviço Singleton que armazena a `Enterprise` ativa durante o ciclo da requisição.

```php
namespace App\Infrastructure\MultiTenancy;

use App\Infrastructure\Persistence\Eloquent\Models\EnterpriseModel;

class TenantContext
{
    private ?EnterpriseModel $tenant = null;

    public function set(EnterpriseModel $tenant): void
    {
        $this->tenant = $tenant;
    }

    public function get(): ?EnterpriseModel
    {
        return $this->tenant;
    }

    public function hasTenant(): bool
    {
        return !is_null($this->tenant);
    }
}
```

### Trait de identificação (`BelongsToTenant.php`)

Trait aplicado aos Models Eloquent em `app/Infrastructure/Persistence/Eloquent/Models/`.

```php
namespace App\Infrastructure\MultiTenancy\Traits;

use App\Infrastructure\MultiTenancy\Scopes\TenantScope;
use Illuminate\Database\Eloquent\Model;

trait BelongsToTenant
{
    public static function bootBelongsToTenant(): void
    {
        // Auto-filtro: injeta o Global Scope em todas as consultas SQL
        static::addGlobalScope(new TenantScope);

        // Auto-injeção: preenche o enterprise_id automaticamente ao criar registros
        static::creating(function (Model $model) {
            if (app()->has('tenant.context') && app('tenant.context')->hasTenant()) {
                $model->enterprise_id = app('tenant.context')->get()->id;
            }
        });
    }
}
```

### Escopo global (`TenantScope.php`)

Intercepta queries e adiciona a cláusula de barreira.

```php
namespace App\Infrastructure\MultiTenancy\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        if (app()->has('tenant.context') && app('tenant.context')->hasTenant()) {
            $tenantId = app('tenant.context')->get()->id;

            // Adiciona: WHERE {tabela}.enterprise_id = {tenant_id_ativo}
            $builder->where($model->getTable() . '.enterprise_id', '=', $tenantId);
        }
    }
}
```

## Fluxo de resolução (identification)

### No Manager (painel administrativo)

- Resolvido via **subdomínio**
- Gerenciado pelo middleware `ResolveTenant`

### No Tracker (clique)

- Resolvido via **slug do link** (ex.: `afil.io/abc`)
- Busca no **Redis (cache)**; em *cache miss*, consulta o **RDS**
- Ao localizar o link, o `enterprise_id` é injetado no `TenantContext`, garantindo que logs (SQS) e validações respeitem o dono do link

## Regras de integridade

### Barreira de segurança

Nenhuma consulta aos models de recursos deve ser executada sem `TenantContext` definido, exceto em rotas públicas de autenticação.

### Auditoria vs isolamento

- `person_id` identifica o **autor da ação**
- A segurança entre clientes é garantida pela coluna `enterprise_id`

### Performance

Tabelas que utilizam `BelongsToTenant` devem possuir **índice** na coluna `enterprise_id`.
