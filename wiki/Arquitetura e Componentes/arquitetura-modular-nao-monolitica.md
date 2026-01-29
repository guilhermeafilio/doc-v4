# Por que esta arquitetura não é um monolito problemático

Este documento articula como a combinação **estrutura de código modular** + **Single Image Multiple Roles (EKS)** + **monorepo** evita os problemas típicos de um monolito e permite defender a solução como uma **aplicação modular bem estruturada**, mesmo com um único repositório e uma única imagem Docker.

## 1. Monolito vs. aplicação modular

Um **monolito problemático** costuma ter:

- Lógica de negócio acoplada a frameworks, UI e infraestrutura.
- Front-end, back-end e jobs misturados sem fronteiras claras.
- Escalabilidade única (tudo sobe/desce junto).
- Deploy e evolução travados por um “bloco único” difícil de testar e alterar.

Uma **aplicação modular** (mesmo em um único deploy) tem:

- **Camadas e bounded contexts** bem definidos.
- **Contratos e interfaces** que isolam domínio de infraestrutura.
- **Runtime e escalabilidade** que podem ser diferenciados por função (Tracker, Worker, Manager).
- **Código organizado** para manutenção, testes e eventual extração de serviços.

A arquitetura descrita aqui e no [EKS Mapping](eks-mapping-roles.md) segue o segundo modelo.

---

## 2. Defesa pela estrutura de pastas (Clean/Hexagonal)

A estrutura de pastas (Domain, Application, Infrastructure, Interfaces) implementa **separação de responsabilidades** e **inversão de dependência**, o que afasta o “big ball of mud” típico do monolito mal estruturado.

### 2.1 Camadas claras

| Camada | Função | Efeito anti-monolito |
|--------|--------|----------------------|
| **Domain** | Entidades, regras de negócio, contratos (interfaces). Zero dependência de Laravel, DB ou HTTP. | Núcleo estável; mudanças em framework ou infra não quebram regras de negócio. |
| **Application** | Use cases, DTOs, orquestração. Depende só do Domain. | Casos de uso explícitos e testáveis sem levantar servidor ou banco. |
| **Infrastructure** | Eloquent, Redis, S3, filas, multi-tenancy. Implementa interfaces do Domain. | Troca de ORM, cache ou fila sem alterar Domain/Application. |
| **Interfaces / Http** | Controllers, entrada HTTP. Chamam use cases, não acessam DB diretamente. | Troca de API (REST, GraphQL, CLI) sem mexer no núcleo. |

Com isso, o **domínio e a aplicação não dependem** de detalhes de entrega (HTTP) nem de persistência (Eloquent, Redis). Isso é o oposto de um monolito acoplado.

### 2.2 Bounded contexts no Domain e na Application

Pastas como `Domain/Tenant`, `Domain/Projects`, `Application/Tenant/UseCases`, `Application/Projects/UseCases` funcionam como **contextos delimitados**:

- Responsabilidades agrupadas por capacidade de negócio (tenant, projetos).
- Contratos (`Repository`, `TenantAware`, etc.) na camada Shared.
- Redução de acoplamento entre contextos; evolução pode ser feita por “módulo”.

Isso facilita entendimento, refatoração e, no futuro, eventual extração de um contexto para outro serviço, **sem reescrever tudo**.

### 2.3 Abstrações e baixo acoplamento

- **Domain:** define interfaces (ex.: `TenantRepositoryInterface`, `ProjectRepositoryInterface`).
- **Infrastructure:** implementa (ex.: `EloquentTenantRepository`, `EloquentProjectRepository`).
- **Providers:** fazem o binding em um só lugar.

Efeito: troca de implementação (ex.: outro ORM ou outro storage) não propaga para Domain/Application. O monolito ruim quebra em vários pontos ao trocar tecnologia; aqui o impacto fica contido na Infrastructure.

### 2.4 Multi-tenancy e cross-cutting como módulos

Pastas dedicadas (`MultiTenancy`, `Storage`, `Cache`, `Mail`) tratam **cross-cutting** como módulos da Infrastructure, com traits, scopes e middlewares bem definidos. Isso evita que multi-tenancy e infraespalhem “if (tenant)” e detalhes técnicos por todo o código.

---

## 3. Defesa pelo runtime (EKS: Single Image, Multiple Roles)

O [EKS Mapping](eks-mapping-roles.md) usa **uma única imagem**, mas **três papéis de execução** (Tracker, Worker, Manager), cada um com comando, recursos e healthchecks próprios. Isso quebra a ideia de “um único processo gigante” do monolito clássico.

### 3.1 Isolamento de falhas e gargalos

- **Tracker (Octane):** foco em latência e Redis; não escreve em RDS.
- **Worker (Horizon):** consome fila, escreve em RDS; escala com backlog da fila.
- **Manager (FPM):** CRUD, relatórios, uso de memória; escala por memória.

Se o Manager travar em um relatório pesado, Tracker e Worker continuam operando. Falhas e picos de carga ficam **isolados por papel**, não por “um monolito único”.

### 3.2 Escalabilidade independente

- HPA por **RPS/CPU** (Tracker), **tamanho da fila** (Worker), **memória** (Manager).
- Cada papel pode ter réplicas e limites de recurso diferentes.

Ou seja: você escala por **função**, não “o sistema inteiro de uma vez”, o que é característico de desenho modular, não de monolito rígido.

### 3.3 Mesma base de código, comportamentos distintos

Single Image garante **consistência de modelo e contratos** (ERD, Jobs, DTOs); o **comportamento** muda pelo **comando** (`octane:start`, `horizon`, `php-fpm`). Isso evita duplicação e drift entre “microserviços” mantidos separadamente, mantendo benefícios de modularidade no runtime.

---

## 4. Defesa pelo monorepo modular

O [Monorepo Modular](monorepo-modular.md) reforça que:

- **Single Source of Truth:** ERD e migrations em um só lugar; não há “vários monolitos” com modelos diferentes.
- **Contratos compartilhados:** Jobs SQS e eventos são as mesmas classes para Tracker e Worker; não há acoplamento por rede ou cópia de código.
- **CI/CD único:** uma pipeline gera a imagem que alimenta todos os papéis; qualidade e versão são consistentes.

O risco do monorepo é virar “monolito confuso”. Por isso a organização em **DDD/Clean** (Domain, Application, Infrastructure, bounded contexts) e a separação por **Controller/Jobs** (Tracker vs Manager vs Jobs) são essenciais: o repositório é único, mas o **código é modular**.

---

## 5. Resumo: por que não é “monolito problemático”

| Aspecto | Monolito típico | Esta arquitetura |
|---------|------------------|------------------|
| **Acoplamento** | Domínio grudado em framework e DB | Domain/Application sem dependência de HTTP ou ORM |
| **Escalabilidade** | Um processo, um tipo de carga | Três papéis (Tracker, Worker, Manager) com HPA e recursos distintos |
| **Falhas** | Um gargalo derruba tudo | Isolamento por papel; healthchecks e probes por runtime |
| **Evolução** | Mudança em um lugar quebra vários | Camadas e bounded contexts; mudanças contidas por módulo |
| **Organização** | Código espalhado, difícil de localizar | Estrutura de pastas clara (Domain, Application, Infrastructure, Interfaces) |
| **Contratos** | Chamadas diretas a DB/framework | Interfaces e use cases; infraestrutura intercambiável |

---

## 6. Conclusão

A arquitetura pode ser caracterizada como:

- **Modular no código:** Clean/Hexagonal, bounded contexts, contratos e camadas bem definidos.
- **Modular no runtime:** Single Image, Multiple Roles no EKS, com isolamento de falhas e escalabilidade por função.
- **Monorepo disciplinado:** uma única fonte de verdade e uma única pipeline, com organização que evita o “monolito confuso”.

Assim, **não se trata de um monolito no sentido problemático**: é uma **aplicação modular** que hoje vive em um repositório e uma imagem, mas com fronteiras claras que permitem manutenção, testes e, se no futuro for necessário, extração de partes em serviços independentes sem reescrever o núcleo.
