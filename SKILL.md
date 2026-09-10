---
name: practical-clean-architecture
description: "Aplicar, revisar e refatorar aplicações seguindo a Practical Clean Architecture: Presentation, Service, Core, Datastore e Data; direção de dependências; isolamento do domínio; UseCases puros; mappers; transações; mensageria; tratamento de erros e observabilidade. Usar ao criar funcionalidades, revisar arquitetura, organizar pacotes Java ou detectar violações entre camadas."
argument-hint: "Descreva a funcionalidade, módulo ou código que deve seguir a arquitetura"
user-invocable: true
---

# Practical Clean Architecture

## Objetivo

Aplicar uma arquitetura orientada ao domínio, inspirada em Clean Architecture, Hexagonal, DDD e Application Service, com o mínimo de abstrações necessário para preservar correção, segurança, clareza e manutenibilidade.

Esta skill é um padrão de decisão. Não crie camadas, interfaces, Ports, agregados ou UseCases apenas por convenção. Crie uma abstração quando existir uma responsabilidade real, uma fronteira que precise ser protegida ou uma variação que justifique o isolamento.

## Quando Usar

Use esta skill para:

- criar ou alterar funcionalidades em aplicações organizadas por camadas;
- revisar imports, dependências, contratos e limites entre módulos;
- decidir se uma operação pertence ao `Service`, `Core/UseCase` ou `Datastore`;
- separar DTOs, Models, Entities, eventos e modelos de APIs externas;
- implementar entradas REST, GraphQL, mensagens, jobs ou integrações;
- revisar transações, erros, mappers e observabilidade.

## Procedimento

1. Identifique o contexto de negócio e a entrada da operação.
2. Leia a implementação existente, os contratos adjacentes e os testes relevantes antes de editar.
3. Determine se a operação é simples ou contém regra de negócio:
   - conversão + chamada ao `Datastore`: permaneça no `Service`;
   - invariantes, decisões, cálculos ou transições de domínio: crie ou use um `UseCase` puro;
   - acesso técnico a banco, API, fila ou outro sistema: encapsule no `Datastore` e na `Data`.
4. Verifique a direção das dependências e os tipos usados nas assinaturas públicas.
5. Faça a menor alteração que preserve os limites abaixo.
6. Adicione ou ajuste testes da regra alterada.
7. Execute a validação mais específica disponível antes de ampliar a alteração.
8. Revise o diff, imports, tratamento de erros, transação e exposição de dados sensíveis.

## Camadas e Dependências

A direção das dependências é de fora para dentro:

```text
Presentation -> Service -> Core
                      -> Datastore -> Data/origem externa
```

| Camada | Responsabilidade | Pode conhecer |
|---|---|---|
| `Presentation` | Entrada externa (HTTP, GraphQL, mensagens), contrato, resposta, validação de formato e conversão `DTO`/`Type`/`Event` ↔ `Model` | `Service`, `Core` (Models) |
| `Service` | Orquestração, transação e coordenação | `Core`, API pública do `Datastore` |
| `Core` | Models, invariantes, regras de negócio e UseCases puros | Somente biblioteca padrão e abstrações próprias do domínio |
| `Datastore` | Contrato único de dados, mapeamento e ocultação da origem | `Core` e componentes internos da `Data` |
| `Data` | Repositories, Entities, ApiClients, MessagePublishers, mappers Core↔Data e modelos tecnológicos | Infraestrutura externa e `Core` |

### Dependências proibidas

- `Core` não importa `Presentation`, `Service`, `Datastore`, `Data`, frameworks ou infraestrutura.
- `Service` não importa `DTO`/`Type`/`Event` da `Presentation`, `Entity`, modelo de API externa, mensagem de saída, `Repository` ou `MessagePublisher`.
- `Presentation` não acessa `Datastore`, `Data`, `UseCase` nem aplica regra de domínio; usa Models do `Core` apenas como dado trocado com o `Service`.
- `Data` não expõe seus componentes para `Presentation`, `Service` ou `Core`.
- `UseCase` não acessa `Datastore`, Repository, API, fila, sessão de banco ou qualquer infraestrutura.
- O `Datastore` não depende do `Service` nem da `Presentation`.

`MessageListener` é um adaptador de entrada e fica na `Presentation`, par do Controller: converte o payload em `Event`, mapeia para Model do Core em `presentation.mapper` e chama o `Service`, sem acessar `Datastore` ou `Data`. `MessagePublisher` é um adaptador de saída da `Data`, acionado internamente pelo `Datastore`.

## Limites dos Modelos

Cada camada possui modelos próprios. Nunca atravesse uma fronteira com o modelo de outra camada:

```text
Presentation: DTO, Type ou Event
        -> mapper da fronteira
Core: Model
        -> mapper da fronteira
Data: Entity, modelo de API ou Message
```

Regras:

- DTOs REST usam o sufixo `DTO`; tipos GraphQL usam `Type`; mensagens de entrada usam `Event`.
- Models do Core não têm sufixo técnico e não dependem de frameworks.
- Entities, modelos de API e mensagens de saída (`Message`) são exclusivos da `Data`.
- Um tipo externo não pode aparecer em atributo, parâmetro, retorno, exceção ou contrato público de outra camada.
- Mapeamentos entre modelos devem ser explícitos e ficar na fronteira que conhece os dois tipos: `presentation.mapper` para `DTO`/`Type`/`Event` ↔ `Model`; `data.mapper` para `Model` ↔ `Entity`/`API Model`/`Message`. O `Model` do Core é o pivô — nenhum mapper conhece `DTO` e `Entity` ao mesmo tempo.
- Use mapper genérico apenas quando nomes, tipos e semântica forem compatíveis. Para normalização, cálculo, `null` especial, objetos aninhados, campos sensíveis ou incompatibilidade semântica, use mapper específico.

## Nomenclatura e Beans

Sufixo, pacote e anotação Spring esperados por tipo de componente. Os nomes usam o contexto de negócio como prefixo (ex.: `User`, `Order`).

| Componente | Sufixo / nome | Pacote | Anotação |
|---|---|---|---|
| Controller REST | `Controller` | `presentation.rest.<contexto>` | `@RestController` |
| Controller GraphQL | `Controller` | `presentation.graphql.<contexto>` | `@Controller` |
| MessageListener | `MessageListener` | `presentation.messaging.<contexto>` | classe `@Component`; `@KafkaListener`/`@RabbitListener`/`@JmsListener`/`@SqsListener` no método |
| DTO REST | `DTO` | `presentation.rest.<contexto>` | — |
| Type GraphQL | `Type` | `presentation.graphql.<contexto>` | — |
| Event de entrada | `Event` | `presentation.messaging.<contexto>` | — |
| Mapper da Presentation | `Mapper`, `EventMapper` | `presentation.mapper.<contexto>` | `@Component` |
| Service | `Service` | `business.service.<contexto>` | `@Service` |
| Model do Core | sem sufixo | `business.core.<contexto>` | — |
| UseCase | `UseCase` | `business.core.<contexto>` | — (POJO; sem `@Transactional` nem anotação de infraestrutura) |
| Exceção de domínio | `Exception` (ex.: `UserDomainException`) | `business.core.<contexto>` | — |
| Datastore (contrato + implementação) | `Datastore` | `data.datastore.<contexto>` | `@Component` |
| Repository | `Repository` | `data.persistence.<contexto>` | `@Repository` |
| Entity | `Entity` | `data.persistence.<contexto>` | — |
| ApiClient | `ApiClient`, prefixo do provedor (ex.: `GoogleApiClient`) | `data.api.<provedor>` | `@Component` |
| Modelo de API externa | prefixo do provedor (ex.: `GoogleUser`) | `data.api.<provedor>` | — |
| MessagePublisher | `MessagePublisher` | `data.messaging.<contexto>` | `@Component` |
| Message de saída | `Message` (ex.: `UserMessage`) | `data.messaging.<contexto>` | — |
| Mapper da Data | `EntityMapper`, `ApiMapper`, `MessageMapper` | `data.mapper.<contexto>` | `@Component` |

- O sufixo `UseCase` é o marcador usado, em revisão ou verificação automatizada, para confirmar que ele não depende de infraestrutura.
- `DTO`, `Type`, `Event`, `Entity`, modelo de API externa e `Message` não aparecem fora do pacote que os define.

## Service e UseCase

O `Service` prepara dados, chama o `UseCase` quando necessário, coordena `Datastore`s e controla a transação. Recebe e devolve apenas Models do Core (ou resultados do Core); a conversão para `DTO`/`Type`/`Event` fica no adaptador de entrada.

O `UseCase` recebe Models do Core, aplica regras e devolve Models processados. Ele deve ser determinístico, stateless e testável sem Spring, banco, HTTP, mensageria ou SDK externo. Não mantenha estado mutável de execução nem estado compartilhado entre chamadas em atributos do `UseCase`; o estado da operação deve entrar pelos parâmetros e sair pelo resultado.

Não crie um `UseCase` quando a operação for somente conversão e uma chamada ao `Datastore`. Crie um quando houver regra de negócio, invariante, cálculo, decisão ou transformação de domínio.

O `Service` não deve concentrar regras de negócio. Também não deve conhecer a origem concreta dos dados.

## Datastore e Data

O `Datastore` é a única abstração de dados visível para `Service` e níveis superiores. Seus contratos aceitam e retornam apenas Models do Core, valores simples ou resultados definidos pelo Core.

Internamente, o `Datastore` pode usar Repositories, ApiClients, componentes de mensageria e os mappers de `data.mapper`. Ele deve converter falhas técnicas e modelos externos antes de devolvê-los ao `Service`.

A `Data` implementa os detalhes tecnológicos:

- `Repository` acessa persistência e trabalha com `Entity`.
- `ApiClient` acessa APIs externas e trabalha com modelos exclusivos da integração.
- `MessagePublisher` publica mensagens de saída e trabalha com `Message`, acionado pelo `Datastore`.
- `data.mapper` converte `Model` do Core ↔ `Entity`/`API Model`/`Message`; usado pelo `Datastore`, depende do `Core` e dos pacotes técnicos da `Data`, nunca de `Service` ou `Presentation`.
- Restrições de banco pertencem à `Data`, mas não substituem invariantes do `Core`.

## Entrada e Saída por Mensagens

**Entrada.** O `MessageListener` é um adaptador de entrada da `Presentation`, par do Controller, em `presentation.messaging`. Enxerga apenas o `Service`. Deve somente validar o envelope técnico mínimo, converter o payload em um `Event` (contrato da `Presentation`) e chamar um `Service` — inclusive para operações simples. Não contém regra de negócio, cálculo, composição, transação de negócio, chamada direta a `UseCase`, nem acesso a `Datastore`, `Repository`, `Entity` ou modelo de API. A conversão `Event` -> Model ocorre na fronteira do `Service`.

Fluxo padrão:

```text
MessageListener -> Service -> UseCase
                         -> Datastore
```

Confirme a mensagem apenas após o processamento bem-sucedido. Falhas devem seguir a política de retry, reprocessamento, dead-letter ou descarte do adaptador.

**Saída.** A publicação em filas ou tópicos é feita pelo `MessagePublisher` em `data.messaging`, na mesma categoria de `Repository` e `ApiClient`. É acionado pela implementação do `Datastore`; `Service` e `Core` não sabem que a operação envolve mensageria. O `Datastore` converte o Model para `Message` antes de publicar.

## Validações

- `Presentation`: presença, formato, tamanho, pattern, paginação e enum do contrato externo.
- `Core`: invariantes, estados permitidos, transições, limites, unicidade de negócio e consistência entre atributos.
- `Data`: `NOT NULL`, tipos, tamanhos, chaves, índices e constraints físicas.

A validação da `Presentation` melhora o feedback, mas nunca substitui a validação do `Core`. O Core deve proteger suas regras independentemente da origem da chamada.

## Erros

As exceções devem ser traduzidas na fronteira correta e preservar a causa original:

```text
Data exception -> Datastore infrastructure exception
Core domain exception -> classificação do Service
Presentation handler -> resposta da API
```

- O `Core` lança somente exceções de domínio, nunca HTTP, SQL, JPA, Kafka ou exceções de API externa.
- A `Data` encapsula exceções da biblioteca ou fornecedor e não as expõe diretamente.
- O `Datastore` traduz falhas de integração e conversão, sem decidir status HTTP.
- O `Service` classifica erros conhecidos e não captura exceções inesperadas para mascará-las.
- A `Presentation` retorna respostas padronizadas sem stack trace, SQL, credenciais ou payload sensível.
- Erros inesperados devem ter contexto e correlation ID no log, mas resposta externa genérica.

Categorias usuais: entrada inválida `400`, não autenticado `401`, sem permissão `403`, inexistente `404`, conflito de regra `409`, dependência indisponível `502/503` e falha inesperada `500`.

## Transações

A unidade transacional pertence ao `Service` e deve abranger a unidade completa de negócio. O `UseCase` é puro e nunca usa `@Transactional`, `EntityManager`, sessão de banco ou API de infraestrutura.

O `Datastore` participa da transação iniciada pelo `Service` e não abre uma transação independente por chamada. Para fontes que não compartilham transação, use uma estratégia explícita, como compensação, retry controlado, outbox ou saga.

Não mantenha chamadas externas demoradas dentro de transações que possam prolongar locks. O rollback deve ocorrer para falhas que invalidem a operação.

## Observabilidade

Não acople o `Core` a logging, métricas, tracing ou SDKs de observabilidade. O contexto de correlação deve atravessar HTTP, mensagens e integrações externas.

Logs estruturados devem conter, quando aplicável, timestamp UTC, nível, serviço, ambiente, operação, `correlationId`, `traceId`, resultado, duração e categoria do erro. Nunca registre tokens, senhas, chaves, cabeçalhos de autenticação, payloads completos ou dados pessoais desnecessários.

Colete métricas de volume, sucesso, erro, latência `p50/p95/p99`, retries, dead-letter e dependências. Use dimensões controladas, sem IDs individuais ou alta cardinalidade.

Registre cada falha uma vez, no ponto com contexto suficiente. Camadas intermediárias podem adicionar contexto e preservar a causa sem duplicar o mesmo erro.

## Organização Java Sugerida

Esta skill assume um único módulo. As camadas e os contextos de negócio são organizados como pacotes dentro desse módulo; a separação física em módulos está fora do escopo. Preserve os limites arquiteturais mesmo sem separação de classpath entre as camadas.

O código é agrupado em três pacotes raiz. `presentation` reúne os adaptadores de entrada (`rest`, `graphql`, `messaging`); `business` reúne `Service` e `Core`; `data` reúne `Datastore` e os adaptadores de saída. O agrupamento é apenas organizacional: as responsabilidades e a direção das dependências de cada camada permanecem as mesmas.

```text
com.example.app
├── presentation
│   ├── rest/user
│   ├── graphql/user
│   ├── messaging/user      (MessageListener + Event)
│   └── mapper/user
├── business
│   ├── service/user
│   └── core/user
└── data
    ├── datastore/user     (contrato + implementação)
    ├── mapper/user         (Model <-> Entity / API Model / Message)
    ├── persistence/user
    ├── api/provider
    └── messaging/user      (MessagePublisher + Message)
```

Dentro de `business`, `Core` não conhece `Service`. Dentro de `data`, apenas o contrato do `Datastore` em `data.datastore` é visível para `business.service`; `data.mapper`, `data.persistence`, `data.api` e `data.messaging` permanecem internos. O `MessageListener` (entrada) fica em `presentation.messaging`, depende do `Service` e do `Core` (Models); `data.messaging` contém apenas a publicação de saída. Cada fronteira tem seu pacote `mapper` (`presentation.mapper` e `data.mapper`), com o `Model` do Core como pivô. A `Presentation` pode depender de `business.core` (Models trocados com o `Service`); o `business` nunca depende da `Presentation`, e o `Service` nunca recebe `DTO`/`Type`/`Event`.

Organize por contexto de negócio dentro de cada camada. Evite pacotes globais como `model`, `mapper` ou `exception` quando eles misturarem contextos. Use visibilidade `package-private` para esconder detalhes e exponha somente contratos necessários.

## Checklist de Revisão

Antes de concluir uma alteração, confirme:

- [ ] A regra de negócio está no `Core`, não no Controller, listener, Service ou Data.
- [ ] O `UseCase`, quando existe, não conhece infraestrutura.
- [ ] O `UseCase` é stateless e não mantém estado mutável entre chamadas.
- [ ] O `Service` conhece somente o contrato do `Datastore`.
- [ ] Nenhum DTO, Entity, Event ou modelo de API atravessa sua fronteira.
- [ ] Mappers estão explícitos e localizados na fronteira correta.
- [ ] A transação está no Service e cobre a unidade de negócio necessária.
- [ ] Exceções foram traduzidas sem expor detalhes técnicos ou dados sensíveis.
- [ ] Mensagens só são confirmadas após sucesso e seguem retry/dead-letter em falhas.
- [ ] Logs, métricas e traces não vazam segredos nem dados desnecessários.
- [ ] A menor estrutura suficiente foi escolhida, sem abstrações cerimoniais.
- [ ] Sufixos, pacotes e anotações Spring seguem a tabela de Nomenclatura e Beans.
- [ ] Testes unitários cobrem regras do Core e testes de integração cobrem fronteiras relevantes.
- [ ] Imports e dependências proibidos foram verificados.
