# Estratégia de Código: Monorepo Modular

Opta-se pela estratégia de **Monorepo** para garantir a integridade do modelo de dados (ERD) e evitar a duplicação de lógica de negócio.

## Vantagens para a solução

1. **Single Source of Truth:** O ERD (Migrations e Models) é definido uma única vez.
2. **Consistência de contratos:** Os Jobs de fila (SQS) compartilham as mesmas classes de eventos entre quem envia (Tracker) e quem recebe (Worker).
3. **Eficiência de CI/CD:** Um único pipeline gera a imagem que serve a toda a infraestrutura AWS.

## Organização do Código

Para evitar que o “Monolito” se torne confuso, o projeto segue o padrão de **DDD** (e estrutura tipo Clean/Hexagonal: Domain, Application, Infrastructure, Interfaces).

A estrutura de pastas é organizada da seguinte forma:

```
monorepo-root/
├── domain/          # Lógica de negócio principal
├── application/     # Casos de uso e orquestração
├── infrastructure/  # Implementações concretas (DB, API, etc.)
├── interfaces/      # Adapters e comunicação externa
└── ...
```

Cada pasta representa uma camada do DDD, com responsabilidades bem definidas.

- **domain**: Contém as entidades, value objects, e regras de negócio.
- **application**: Define os casos de uso da aplicação, utilizando a lógica do domínio.
- **infrastructure**: Implementa as abstrações definidas nas camadas superiores, como acesso ao banco de dados e comunicação com serviços externos.
- **interfaces**: Expõe a aplicação através de APIs, interfaces de usuário, ou outros meios.
