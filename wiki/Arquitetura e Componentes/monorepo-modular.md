# Estratégia de Código: Monorepo Modular

Opta-se pela estratégia de **Monorepo** para garantir a integridade do modelo de dados (ERD) e evitar a duplicação de lógica de negócio.

## Vantagens para a solução

1. **Single Source of Truth:** O ERD (Migrations e Models) é definido uma única vez.
2. **Consistência de contratos:** Os Jobs de fila (SQS) compartilham as mesmas classes de eventos entre quem envia (Tracker) e quem recebe (Worker).
3. **Eficiência de CI/CD:** Um único pipeline gera a imagem que serve a toda a infraestrutura AWS.

## Organização do código

Para evitar que o “Monolito” se torne confuso, o projeto segue o padrão de **DDD** (e estrutura tipo Clean/Hexagonal: Domain, Application, Infrastructure, Interfaces), separando claramente:

- `app/Core`: Lógica compartilhada e Models.
- `app/Http/Controllers/Tracker`: Endpoints de alta performance.
- `app/Http/Controllers/Manager`: Endpoints do dashboard e cadastros.
- `app/Jobs`: Processamentos assíncronos (match de conversão).

Para uma defesa articulada de por que esta arquitetura **não é um monolito problemático** (estrutura de pastas + Single Image Multiple Roles no EKS), ver [Por que esta arquitetura não é um monolito problemático](arquitetura-modular-nao-monolitica.md).
