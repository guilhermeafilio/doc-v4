# Estratégia de Código: Monorepo Modular

Origem: `repository.md`.

Optamos pela estratégia de **Monorepo** para garantir a integridade do modelo de dados (ERD) e evitar a duplicação de lógica de negócio.

## Vantagens para a solução

1. **Single Source of Truth:** O ERD (Migrations e Models) é definido uma única vez.
2. **Consistência de contratos:** Os Jobs de fila (SQS) compartilham as mesmas classes de eventos entre quem envia (Tracker) e quem recebe (Worker).
3. **Eficiência de CI/CD:** Um único pipeline gera a imagem que serve a toda a infraestrutura AWS.

## Organização do código

Para evitar que o “Monolito” se torne confuso, o projeto segue o padrão de **DDD**, separando claramente:

- `app/Core`: Lógica compartilhada e Models.
- `app/Http/Controllers/Tracker`: Endpoints de alta performance.
- `app/Http/Controllers/Manager`: Endpoints do dashboard.
- `app/Jobs`: Processamentos assíncronos (match de conversão).
