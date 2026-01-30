# Home

Esta Wiki centraliza a documentação técnica do **V4** (arquitetura, fluxos, contratos e dados). Baseado no estudo feito a partir da V3.

### Arquitetura e componentes
- [Monorepo modular](./Arquitetura%20e%20Componentes/monorepo-modular.md)
- [EKS roles (Tracker/Worker/Manager)](./Arquitetura%20e%20Componentes/eks-mapping-roles.md)
- [Build e deploy](./Arquitetura%20e%20Componentes/build-e-deploy.md)
- [Divisão de workers por fila (track / conversion)](./Arquitetura%20e%20Componentes/worker-divisao-filas.md)
- [Arquitetura modular (não monolítica)](./Arquitetura%20e%20Componentes/arquitetura-modular-nao-monolitica.md)
- [Multi-tenancy](./Arquitetura%20e%20Componentes/multi-tenancy.md)

### Fluxos
- [End-to-end](./Fluxos/end-to-end.md)
- [Sequência: clique (Tracker)](./Fluxos/sequencia-clique-tracker.md)
- [Sequência: conversão (Postback)](./Fluxos/sequencia-conversao-postback.md)
- [Sequência: status da conversão](./Fluxos/sequencia-status-conversao.md)

### Contratos
- [Evento: click_logged](./Contratos/click_logged.md)
- [Evento: postback_received](./Contratos/postback_received.md)

### Dados
- [Enums e índices](./Dados/Enums%20e%20Indices/enums-e-indices.md) — CONVERSION_STATUS (enum) e estratégia de indexação.
- [ERD (Rodrigo)](./Dados/Modelo%20(ERD)/erd-rodrigo.md) — entidades e relacionamentos resumida com detalhes a partir de track.
