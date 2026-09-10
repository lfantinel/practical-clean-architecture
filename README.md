# Practical Clean Architecture

Uma arquitetura orientada ao domínio, inspirada na Clean Architecture,
simplificada para favorecer clareza, manutenção e produtividade.

## Contexto e Objetivos

Este modelo é uma síntese pragmática de práticas observadas em arquiteturas consolidadas e refinadas ao longo da experiência no desenvolvimento de sistemas de diferentes domínios, tamanhos e níveis de complexidade.

Seu objetivo é combinar separação de responsabilidades, isolamento do domínio e flexibilidade de integração com uma estrutura prática, fácil de entender, implementar e manter. O modelo não pretende reproduzir integralmente uma arquitetura específica; ele aproveita princípios que demonstraram valor e simplifica cerimônias que não trazem benefício proporcional ao contexto da aplicação.

### Princípios Aproveitados

| Referência arquitetural | Pontos fortes incorporados |
| --- | --- |
| Arquitetura em camadas | Separação clara de responsabilidades, fluxo previsível e organização simples para o entendimento da equipe |
| Clean Architecture | Proteção do domínio contra frameworks e infraestrutura, direção de dependências para dentro e isolamento do `Core` |
| Hexagonal ou Ports and Adapters | Uso de fronteiras explícitas, contratos estáveis e ocultação das fontes externas pelo `Datastore` |
| DDD | Centralidade do domínio, Models sem dependência técnica, invariantes e regras de negócio concentradas no `Core` |
| Application Service | Orquestração de operações, conversão de modelos, controle transacional e coordenação entre UseCases e Datastore |
| Padrão Data Mapper | Separação entre modelos da API, domínio e persistência por meio de mappers explícitos |
| Arquiteturas orientadas a eventos | Entrada e saída por mensagens, confirmação após processamento, retry, dead-letter e propagação de contexto de observabilidade |
| Observabilidade moderna | Logs estruturados, métricas, tracing distribuído, correlação de operações e proteção de dados sensíveis |

### Simplificações Adotadas

Para manter o modelo prático, alguns elementos foram deliberadamente simplificados:

- Não há obrigação de criar Ports, Interfaces ou um UseCase para toda operação. Operações simples de conversão e acesso ao `Datastore` podem ser orquestradas diretamente pelo `Service`.
- O `Datastore` é apresentado como um conceito único para o restante da aplicação. A implementação pode utilizar banco, API externa ou mensageria internamente, sem expor a origem dos dados.
- O `UseCase` não acessa repositórios nem portas de infraestrutura. Ele recebe Models, aplica regras de negócio e devolve Models processados, mantendo a lógica pura, stateless e fácil de testar.
- Não são exigidos agregados, entidades ricas, eventos de domínio ou objetos de valor para todos os casos. Esses recursos devem ser usados quando reduzirem complexidade ou protegerem uma regra relevante.
- Não é imposta uma ferramenta específica de mapeamento, mensageria, logging, tracing ou persistência. A escolha fica com a implementação, desde que respeite os contratos e limites definidos.
- A organização por camadas e contexto de negócio é preferida a uma decomposição excessiva em módulos, submódulos e abstrações sem necessidade comprovada.

### Critério de Aplicação

As regras deste documento são um padrão de decisão, não uma exigência de complexidade. A solução deve adotar a menor estrutura capaz de preservar correção, segurança, clareza e manutenibilidade. Uma abstração adicional só deve ser criada quando houver uma responsabilidade real, uma fronteira que precise ser protegida ou uma variação que justifique o isolamento.

A aplicação é organizada em três pacotes raiz, que agrupam cinco camadas lógicas com limites explícitos:

| Pacote raiz | Camada | Responsabilidade principal |
| --- | --- | --- |
| `presentation` | `Presentation` | Receber entradas externas (HTTP, GraphQL, mensagens) e expor respostas |
| `business` | `Service` | Orquestrar a operação e controlar a unidade transacional |
| `business` | `Core` | Aplicar regras e invariantes de negócio |
| `data` | `Datastore` | Consumir as origens dos dados e ocultar sua implementação |
| `data` | `Data` | Implementar o acesso técnico às fontes e destinos externos |

O agrupamento em três pacotes raiz é apenas organizacional. As responsabilidades, a direção das dependências e o isolamento de cada camada permanecem os mesmos: dentro de `business`, `Core` não conhece `Service`; dentro de `data`, apenas o contrato do `Datastore` é visível para fora. Cada camada conhece somente os contratos permitidos, e modelos de uma camada não atravessam esses limites.

## Fluxo Principal

Para uma requisição externa:

```text
Cliente -> Presentation -> Service -> Core/UseCase
                                  -> Datastore -> Data/origem
```

O fluxo de execução pode seguir de fora para dentro e retornar com o resultado processado. A direção das dependências continua sendo de fora para dentro, conforme definido a seguir.

## Direção das Dependências

A direção única das dependências é de fora para dentro: módulos externos podem depender de módulos internos, mas módulos internos nunca podem depender de módulos externos. Essa regra se aplica às dependências de código, aos imports e à injeção de componentes; o fluxo de execução pode seguir o caminho inverso quando necessário.

O `Core` é o núcleo da aplicação. Ele não depende de `Presentation`, `Service`, `Datastore`, `Data`, frameworks ou infraestrutura.

O `Service` pode depender do `Core` e do contrato público do `Datastore` disponibilizado pela `Data`. Seu contrato público trabalha apenas com Models do Core, valores simples e resultados definidos pelo Core — nunca com `DTO`, `Type` ou `Event` da `Presentation`. A `Data` concentra Datastores, Repositories, Entities, ApiClients, MessagePublishers e os mappers Core↔Data (`data.mapper`), dependendo somente do `Core` para trabalhar com Models. Componentes da `Data` não podem depender do `Service` ou da `Presentation`.

A `Presentation` pode depender do `Service` e do `Core`. A dependência do `Core` se limita ao `Model`, necessário para o `presentation.mapper` converter o modelo de contrato (`DTO`/`Type`/`Event`) e para entregar e receber o `Model` do `Service`; não alcança `UseCase`, invariantes ou regras de domínio. O adaptador de entrada (Controller ou `MessageListener`) converte o modelo de contrato (`DTO`/`Type`/`Event`) em Model do Core pelo mapper de `presentation.mapper` antes de chamar o `Service`, e converte o Model de volta na resposta. A `Presentation` usa o Model apenas como dado que entrega ao `Service` e dele recebe: não chama `UseCase`, `Datastore` ou `Repository`, nem aplica regra de domínio. A publicação de mensagens de saída (`MessagePublisher`) é um adaptador de saída da `Data`, acionado internamente pelo `Datastore`.

Dependências proibidas:

```text
Core         -X-> Service, Datastore, Data, Presentation
Service      -X-> Presentation
Presentation -X-> Datastore, Data, UseCase
Data         -X-> Service, Presentation
Presentation -> Service, Core
Service      -> Core, Data
Data         -> Core
```

O `UseCase` segue a mesma regra do `Core`: recebe Models, processa a operação e devolve Models, sem acessar componentes ou modelos de camadas externas. Deve ser stateless: não pode manter estado mutável de execução ou estado compartilhado entre chamadas; o estado da operação deve ser recebido pelos parâmetros e devolvido no resultado.

## Estrutura de Pacotes

![Diagrama da Practical Clean Architecture: pacotes raiz presentation, business e data; core aninhado em business; os mappers ficam na fronteira, ligando presentation e data ao Model do Core; clientes e brokers externos nas laterais.](images/Practical%20clean%20architecture%20-%20diagram.jpg)

Uma sugestão de organização para uma aplicação Java é agrupar o código em três pacotes raiz — `presentation`, `business` e `data` — e, dentro de cada camada, por contexto de negócio. O nome `com.example.app` representa o pacote raiz da aplicação e deve ser substituído pelo namespace real do projeto.

```text
com.example.app
├── presentation
│   ├── rest
│   │   └── user
│   │       ├── UserController.java
│   │       └── UserDTO.java
│   ├── graphql
│   │   └── user
│   │       ├── UserController.java
│   │       └── UserType.java
│   ├── messaging
│   │   └── user
│   │       ├── UserMessageListener.java
│   │       └── UserEvent.java
│   └── mapper
│       ├── UserMapper.java
│       └── UserEventMapper.java
├── business
│   ├── service
│   │   └── user
│   │       └── UserService.java
│   └── core
│       └── user
│           ├── User.java
│           ├── UserUseCase.java
│           └── UserDomainException.java
└── data
    ├── datastore
    │   └── user
    │       └── UserDatastore.java
    ├── mapper
    │   └── user
    │       ├── UserEntityMapper.java
    │       ├── UserApiMapper.java
    │       └── UserMessageMapper.java
    ├── persistence
    │   └── user
    │       ├── UserRepository.java
    │       └── UserEntity.java
    ├── api
    │   └── google
    │       ├── GoogleApiClient.java
    │       └── GoogleUser.java
    └── messaging
        └── user
            ├── UserMessagePublisher.java
            └── UserMessage.java
```

### Regras de Organização

- `presentation.rest`, `presentation.graphql` e `presentation.messaging` contêm os adaptadores de entrada (Controllers e MessageListeners) e seus modelos de contrato (`DTO`, `Type`, `Event`). `presentation.mapper` contém os mappers da fronteira Presentation↔Core (`DTO`/`Type`/`Event` ↔ `Model`), usados pelos adaptadores de entrada — não pelo `Service`.
- `business.service` contém a orquestração das operações e o controle transacional. Trabalha apenas com Models do Core; a conversão de/para `DTO`/`Type`/`Event` fica nos adaptadores de entrada. O `Datastore` público é fornecido por `data.datastore`.
- `business.core` contém somente Models, UseCases e exceções de domínio, sem dependência de frameworks ou infraestrutura. `business.core` não pode importar `business.service`.
- `data.datastore` contém a implementação do contrato único de dados.
- `data.mapper` contém os mappers da fronteira Core↔Data (`Model` ↔ `Entity`/`API Model`/`Message`), usados pela implementação do `Datastore`.
- `data.persistence`, `data.api` e `data.messaging` contêm Repositories, Entities, ApiClients, MessagePublishers e modelos de mensagem de saída específicos das tecnologias externas.
- Os pacotes técnicos de `data` (`data.mapper`, `data.persistence`, `data.api`, `data.messaging`) não devem ser importados por `presentation` ou `business`; somente as classes públicas de `data.datastore` podem ser consumidas por `business.service`.
- Um contexto de negócio deve manter seus componentes próximos dentro de cada camada, incluindo os pacotes `mapper`, e evitar pacotes globais como `model`, `mapper` ou `exception` que misturem contextos.
- Pacotes internos podem usar visibilidade de pacote (`package-private`) para esconder detalhes de implementação. Tipos públicos devem ser limitados aos contratos necessários entre módulos.

O `MessageListener` fica em `presentation.messaging` e segue as mesmas regras de um Controller: enxerga apenas `business.service`, sem acesso a `business.core`, `data.datastore` ou aos pacotes técnicos de `data`. Não há exceção de entrada em `data`; `data.messaging` contém somente a publicação de mensagens de saída, consumida internamente pelo `Datastore`.

## Testes Arquiteturais

Tanto projetos single-module quanto multi-module devem possuir um teste arquitetural com ArchUnit. A estrutura de diretórios, sozinha, não impede imports proibidos. Com `Service` e `Core` sob `business` e `Datastore` sob `data`, o teste ArchUnit é o mecanismo que garante regras como `business.core` não depender de `business.service` e apenas `data.datastore` ser visível para fora de `data`.

O conjunto de regras vai além da direção das dependências entre camadas e cobre: a proibição de a `Presentation` chamar `UseCase` diretamente; o isolamento do `Core` contra frameworks e infraestrutura (só domínio e biblioteca padrão); o limite dos modelos por sufixo (`DTO` preso à `Presentation`; `Entity` e `Message` de saída presos à `Data`); sufixo, pacote e anotação de cada componente conforme a tabela de Nomenclatura e Beans; a transação restrita ao `Service`; e a ausência de ciclos entre os pacotes raiz. As regras ficam em duas classes: `LayerDependencyTest` (dependências, isolamento, modelos e transação) e `NamingConventionTest` (nomenclatura).

No projeto single-module, o teste deve ficar em `src/test/java/com/example/app/architecture`, separado das camadas de produção e executado junto com a suíte normal. O exemplo completo está em [Single-Module Architecture Test.md](Single-Module%20Architecture%20Test.md).

No projeto multi-module, o teste deve ficar no módulo técnico `architecture`, em `architecture/src/test`, com dependências dos módulos da aplicação apenas no escopo de testes. O exemplo completo está em [Multi-Module Architecture Test.md](Multi-Module%20Architecture%20Test.md).

## Entrada e Saída por Mensagens

### Entrada (MessageListener)

O `MessageListener` é um adaptador de entrada da `Presentation`, par do Controller. Fica em `presentation.messaging` e depende do `Service` e do `Core` (Models).

Ele é responsável somente por receber a mensagem, validar o envelope técnico mínimo, confirmar ou rejeitar o recebimento conforme a política da mensageria, converter o payload em um `Event`, mapeá-lo para Model do Core pelo `UserEventMapper` (`presentation.mapper`) e chamar um `Service`. Não deve conter regra de negócio, cálculo, composição de dados ou decisão de domínio.

O fluxo padrão é:

```text
MessageListener -> Service -> UseCase
                         -> Datastore
```

Mesmo quando a mensagem representa apenas uma operação de persistência, recuperação ou atualização sem regra de negócio, o listener chama um `Service` — fino nesses casos. Ele não acessa `Datastore`, `Repository`, `Entity` ou modelo de API externa, e não chama o `UseCase` diretamente.

A conversão de `Event` para Model do Core ocorre em `presentation.mapper`, pelo `UserEventMapper` — exatamente como um `UserDTO` é convertido pelo `UserMapper` no Controller. O listener passa o Model ao `Service`, que orquestra o `UseCase` e os Datastores necessários.

O listener não deve iniciar transações de negócio, traduzir erros em respostas HTTP ou capturar exceções para ocultar falhas. A confirmação da mensagem deve ocorrer somente após o processamento bem-sucedido; em caso de falha, a política de retry, reprocessamento, dead-letter ou descarte deve ser definida pelo adaptador de mensageria e pela operação.

### Saída (MessagePublisher)

A publicação de mensagens em filas ou tópicos é um adaptador de saída da `Data`, em `data.messaging`, na mesma categoria de `Repository` e `ApiClient`. O `MessagePublisher` é acionado pela implementação do `Datastore`, nunca pelo `Service` ou pelo `Core`.

Para os níveis acima, publicar uma mensagem é apenas mais uma operação do `Datastore`: eles não sabem que a operação envolve mensageria. O `Datastore` converte o Model do Core para o modelo de mensagem de saída (`Message`) pelo `UserMessageMapper` antes de publicar.

## Limites dos Modelos

Cada módulo possui seus próprios modelos. DTOs, Types e Events de entrada da `Presentation`, Entities, modelos de APIs externas e mensagens de saída da `Data` não podem atravessar os limites dos módulos. O `Model` do Core é o único modelo que trafega entre `Presentation`, `Service` e `Datastore`.

As conversões ocorrem nas fronteiras, com o `Model` do Core como pivô:

```text
DTO / Type / Event   (presentation)
    <-> presentation.mapper   (no adaptador de entrada)
Model   (core)   — trafega entre Presentation, Service e Datastore
    <-> data.mapper   (na implementação do Datastore)
Entity / API Model / Message   (data)
```

Um tipo de uma camada não pode ser utilizado em assinaturas de métodos, atributos, retornos, exceções ou contratos públicos de outra camada. O `DTO`/`Type`/`Event` não chega ao `Service`: o adaptador de entrada já converte para `Model` antes da chamada.

## Datastore como Abstração Única

O sistema deve possuir um conceito único de `Datastore` para os níveis acima. A origem do dado é transparente para `Presentation`, `Service` e `Core`: eles não devem saber se o dado veio de um banco, de uma API externa, de uma fila, de um tópico ou de outra fonte.

O `Datastore` é responsável por localizar, consumir e persistir o dado na origem adequada, além de converter os modelos externos para `Core Models` e os `Core Models` para os modelos externos. Seu contrato público deve aceitar e retornar somente `Core Models`, valores simples ou resultados definidos pelo Core.

```text
Presentation -> Service -> Core Model
                           -> Datastore
                               -> origem do dado
                           <- Core Model
```

A implementação do `Datastore` pode utilizar internamente Repositories, ApiClients, MessagePublishers, os mappers de `data.mapper` e seus respectivos modelos. A publicação de mensagens de saída é um detalhe interno do `Datastore`, como qualquer outra origem ou destino. Esses detalhes não podem aparecer nos contratos, nomes de métodos, atributos, exceções ou configurações consumidas pelos níveis acima. A organização interna pode ter componentes auxiliares por fonte, mas isso não cria adaptadores ou conceitos distintos visíveis para o restante do sistema.

## Validações

Cada tipo de validação deve permanecer na camada que possui o contexto necessário para aplicá-la:

| Tipo de validação | Camada | Exemplos |
| --- | --- | --- |
| Formato e contrato de entrada | Presentation | Campo obrigatório, formato de e-mail, tamanho máximo, pattern, paginação e enum aceito pela API |
| Invariantes e regras de negócio | Core | Estado permitido, transição de status, limite de operação, unicidade de uma regra de domínio e consistência entre atributos |
| Restrições específicas da persistência | Data | `NOT NULL`, tamanho da coluna, índice, chave estrangeira, constraint, tipo de coluna e limite imposto pelo banco |

A validação de formato na `Presentation` melhora o feedback para o cliente, mas não substitui as validações do `Core`. O Core deve proteger suas invariantes independentemente da origem da chamada, inclusive em operações disparadas por mensagens, jobs ou outros Services.

O `Service` pode coordenar a execução e traduzir erros entre camadas, mas não deve concentrar regras de negócio. O `Datastore` pode tratar falhas de conversão e traduzir erros técnicos de persistência, sem transformar restrições do banco em regras do Core.

As restrições da `Data` são necessárias para garantir a integridade física dos dados, mas não devem ser o único mecanismo de proteção de uma regra de negócio. Quando uma regra puder ser violada antes da persistência, ela deve ser validada no `Core`.

## Tratamento de Erros e Exceções

Cada camada deve criar e tratar somente os erros que pertencem ao seu contexto. Exceções de uma camada não devem expor detalhes de implementação para as camadas externas.

| Origem | Tipo de erro | Responsabilidade |
| --- | --- | --- |
| Presentation | Erro de contrato ou formato | Rejeitar a entrada e retornar o erro no formato padronizado (resposta da API ou ack/nack para mensagens) |
| Core | Erro de regra de negócio ou invariante | Interromper a operação usando uma exceção de domínio, sem conhecer HTTP, banco ou framework |
| Datastore | Erro de conversão ou integração | Traduzir falhas técnicas da Data para uma exceção de infraestrutura conhecida pelo Service |
| Data | Falha de banco, API externa ou mensageria | Registrar o detalhe técnico, preservar a causa e lançar uma exceção específica da fonte |

O fluxo de tratamento deve seguir a fronteira de cada módulo:

```text
Data exception
    -> Datastore infrastructure exception
Core domain exception
    -> Service error classification
Presentation exception handler
    -> API error response
```

Regras para o tratamento:

- O `Core` lança apenas exceções de domínio e nunca lança exceções HTTP, SQL, JPA, Kafka, RabbitMQ ou de APIs externas.
- O `Data` não retorna exceções de bibliotecas externas diretamente ao `Datastore`; deve preservá-las como causa de uma exceção específica da integração.
- O `Datastore` traduz falhas técnicas e de conversão, sem decidir o resultado HTTP e sem converter uma falha de infraestrutura em regra de negócio.
- O `Service` orquestra o fluxo, classifica erros conhecidos e permite que erros inesperados sejam tratados pelo mecanismo global da aplicação.
- A `Presentation` converte erros classificados em uma resposta padronizada, sem expor stack trace, SQL, credenciais, payloads sensíveis ou detalhes internos.
- Erros inesperados devem ser registrados com contexto e correlation ID, mas a resposta externa deve ser genérica.
- O tratamento não deve capturar `Exception` indiscriminadamente para continuar a execução ou mascarar falhas.

Exemplos de categorias de resposta da API:

| Categoria | Resposta sugerida |
| --- | --- |
| Entrada inválida | `400 Bad Request` |
| Não autenticado | `401 Unauthorized` |
| Sem permissão | `403 Forbidden` |
| Recurso inexistente | `404 Not Found` |
| Conflito de regra ou estado | `409 Conflict` |
| Falha inesperada | `500 Internal Server Error` |
| Dependência externa indisponível | `502 Bad Gateway` ou `503 Service Unavailable` |

## Logs e Observabilidade

A aplicação deve ser observável sem acoplar o `Core` a frameworks, bibliotecas de logging ou plataformas de monitoramento. Logs, métricas e traces são responsabilidades da infraestrutura e devem acompanhar o fluxo da operação entre as camadas.

### Logs

Os logs devem ser estruturados, preferencialmente em JSON, e conter contexto suficiente para investigação sem expor dados sensíveis:

- `timestamp` em UTC;
- `level`;
- `service` ou aplicação;
- `environment`;
- `operation` ou caso de uso;
- `correlationId` para acompanhar uma requisição ou mensagem;
- `traceId` e `spanId` quando houver tracing distribuído;
- resultado da operação;
- duração em milissegundos;
- código ou categoria do erro, quando houver.

Não registrar tokens, senhas, chaves, dados pessoais desnecessários, payloads completos, cabeçalhos de autenticação ou informações financeiras. Quando necessário para diagnóstico, os dados devem ser mascarados, resumidos ou identificados por um hash não reversível.

Os níveis devem seguir uma convenção consistente:

| Nível | Uso |
| --- | --- |
| `ERROR` | Falha que interrompe a operação ou exige intervenção |
| `WARN` | Situação anormal tratada, retry, fallback ou degradação |
| `INFO` | Início e conclusão de operações relevantes, sem excesso de volume |
| `DEBUG` | Detalhes para diagnóstico, desabilitados por padrão em produção |

Cada falha deve ser registrada uma única vez no ponto em que houver contexto suficiente para tratá-la. Camadas intermediárias podem adicionar contexto e preservar a causa, mas não devem gerar o mesmo log repetidamente.

### Métricas

Devem ser coletadas, no mínimo:

- quantidade de requisições e mensagens processadas;
- taxa de sucesso e erro por operação;
- latência por operação, incluindo percentis `p50`, `p95` e `p99`;
- quantidade e duração de retries;
- tamanho de filas e mensagens em dead-letter;
- disponibilidade e latência das dependências usadas pelo Datastore;
- quantidade de erros de domínio, contrato e infraestrutura;
- uso de recursos da aplicação, como CPU, memória e conexões.

Métricas não devem conter dados pessoais ou valores de alta cardinalidade, como IDs individuais, no lugar de tags controladas. Os nomes e dimensões devem ser estáveis para permitir alertas e comparação histórica.

### Tracing

O contexto de trace deve ser propagado entre requisições HTTP, mensagens e chamadas a dependências externas. Cada entrada deve criar ou continuar um trace, e cada operação relevante deve criar um span, incluindo Services, Datastore e integrações externas.

O tracing deve permitir identificar a duração e a falha de cada etapa sem revelar o payload. A origem do dado continua transparente para o `Service` e o `Core`; detalhes da dependência podem aparecer apenas nos spans e logs técnicos do `Datastore` e da `Data`.

### Responsabilidades de Observabilidade por Camada

- `Presentation`: registra correlação da entrada, resultado (HTTP ou ack/nack), status e duração; não registra payload sensível.
- `Service`: registra a operação e seu resultado, incluindo sucesso, erro de domínio e duração da orquestração.
- `Core`: não depende de logging, métricas, tracing ou SDKs de observabilidade. Exceções de domínio carregam somente informações necessárias para classificação.
- `Datastore`: registra falhas de conversão, latência da operação e categoria técnica da origem, sem expor essa origem nos contratos do sistema.
- `Data`: registra detalhes técnicos da integração, retries, timeout e falhas da dependência, preservando segredos e dados sensíveis.

Alertas devem ser baseados em sintomas e impacto, como aumento de erros, latência, indisponibilidade ou acúmulo de mensagens, e não apenas em mensagens específicas de log.

## Transações

A transação deve abranger a unidade de negócio completa, e não apenas uma chamada isolada à origem do dado. Como o `UseCase` é puro e não possui vínculo com `Datastore` ou infraestrutura, ele não deve declarar nem controlar transações. O `UseCase` define a unidade lógica da operação; o `Service` implementa essa unidade no contexto transacional.

Regras para transações:

- O `Service` deve ser o ponto padrão de abertura, confirmação e rollback da transação, conforme a unidade de negócio da operação.
- Uma operação de negócio que altera múltiplos dados deve ser executada dentro de uma única transação sempre que as fontes de dados suportarem a mesma transação.
- O `Service` deve chamar o `UseCase` e o Datastore dentro do mesmo limite transacional quando a consistência da operação exigir isso.
- O `UseCase` não deve usar `@Transactional`, `EntityManager`, sessão de banco ou qualquer API de infraestrutura.
- Operações somente de leitura não precisam de transação de escrita; podem usar uma transação de leitura quando houver necessidade de consistência, isolamento ou controle de sessão.
- O `Datastore` deve participar da transação iniciada pelo `Service` e não deve abrir uma nova transação para cada operação interna.
- Se uma operação envolver fontes que não compartilham a mesma transação, o Service deve usar uma estratégia explícita, como compensação, retry controlado, outbox ou saga, conforme o requisito de consistência.
- O rollback deve ocorrer para falhas de negócio e técnicas que invalidem a operação. Exceções capturadas apenas para registro não devem impedir o rollback.
- A transação não deve incluir chamadas externas demoradas quando isso puder manter locks desnecessariamente; nesses casos, separar a operação ou usar mensageria com processamento assíncrono.

## Mappers

Toda conversão entre modelos de módulos diferentes deve ser centralizada em um componente de mapeamento dedicado — um mapper nomeado (`UserMapper`, `UserEntityMapper`) ou o `Mapper<From, To>` genérico injetado. Controllers, MessageListeners, Services, UseCases, Datastores, Repositories, ApiClients e MessagePublishers não devem copiar atributos à mão nem aplicar regra de conversão em seus métodos de negócio ou de acesso a dados; podem chamar `to`/`from` de um mapper injetado.

Os mappers ficam na fronteira que controlam e são os únicos componentes autorizados a conhecer os dois modelos envolvidos na conversão:

| Mapper | Conversão | Localização |
| --- | --- | --- |
| `UserMapper` | `UserDTO` ou `UserType` <-> `User` | `presentation.mapper` |
| `UserEventMapper` | `UserEvent` <-> `User` | `presentation.mapper` |
| `UserEntityMapper` | `User` <-> `UserEntity` | `data.mapper` |
| `UserApiMapper` | `User` <-> `GoogleUser` | `data.mapper` |
| `UserMessageMapper` | `User` <-> `UserMessage` | `data.mapper` |

A fronteira Presentation↔Core fica em `presentation.mapper`; a fronteira Core↔Data fica em `data.mapper`. O `Model` do Core é o pivô entre as duas: nenhum mapper conhece `DTO` e `Entity` ao mesmo tempo. Os mappers de `presentation.mapper` são chamados pelos adaptadores de entrada (Controller, `MessageListener`) antes e depois da chamada ao `Service`; o `Service` nunca os utiliza. Os mappers de `data.mapper` são usados pela implementação do `Datastore` e não saem da `Data`.

Um mapper pode receber e devolver modelos de módulos adjacentes somente durante a conversão. Esses tipos não podem ser armazenados, retornados ou expostos pelos contratos públicos de módulos além da fronteira. O restante da aplicação utiliza apenas o modelo pertencente ao seu próprio módulo.

### Mapper Genérico

Quando dois modelos possuem os mesmos nomes de atributos, tipos compatíveis e a mesma estrutura, pode ser utilizado um mapper genérico `Mapper<From, To>` para reduzir código repetitivo. Ele guarda o par de tipos da fronteira, delega a cópia dos atributos a um `ObjectMapper` interno e resolve `From` e `To` pelo ponto de injeção. O exemplo completo de implementação — a classe, a fábrica `prototype` e o uso no adaptador de entrada — está em [Generic Mapper.md](Generic%20Mapper.md).

Quando a conversão é totalmente automática, o adaptador de entrada injeta o `Mapper<From, To>` tipado direto e chama `to`/`from`, sem uma classe de mapper escrita à mão. Assim que a fronteira precisar de qualquer regra — campo com nome ou tipo diferente, transformação, `null` condicional, objeto aninhado não automático, campo sensível —, a conversão é promovida para um mapper nomeado em `presentation.mapper` ou `data.mapper` (`UserMapper`, `UserEntityMapper`), com métodos explícitos. O mapper nomeado pode continuar delegando ao `Mapper<From, To>` a parte trivial e tratar à parte só os campos com regra.

O `Mapper<From, To>` deve ser utilizado somente nas fronteiras autorizadas (`presentation.mapper`, `data.mapper`, adaptadores de entrada e implementação do `Datastore`) e não pode ser colocado no `Core`, pois sua implementação depende de reflection, bibliotecas de mapeamento ou frameworks. O bean é `prototype`: cada ponto de injeção carrega um par de tipos distinto e recebe a própria instância. Os genéricos precisam ser concretos no campo ou parâmetro; um `Mapper` sem tipos declarados não é resolvível e a fábrica falha de forma explícita. Cada fronteira configura os tipos permitidos, por exemplo `Mapper<UserDTO, User>` ou `Mapper<User, UserEntity>`.

O mapper genérico não deve ser utilizado quando houver:

- nomes ou tipos de atributos diferentes;
- transformação de valores, normalização ou cálculo;
- regras condicionais ou tratamento especial de `null`;
- conversão de objetos aninhados, coleções ou polimorfismo que não seja automática;
- campos sensíveis que não possam ser copiados por convenção;
- incompatibilidade de versões ou semântica entre os modelos.

Nesses casos, deve ser utilizado um mapper específico, como `UserMapper` ou `UserEntityMapper`, com conversões explícitas. O mapper genérico é uma conveniência de implementação; ele não altera os limites dos módulos nem permite que modelos atravessem suas fronteiras.

## Responsabilidades por Camada

- **Presentation**: Responsável por receber entradas externas — requisições REST, operações GraphQL e mensagens de filas ou tópicos —, converter o modelo de contrato em Model do Core (via `presentation.mapper`) antes de chamar o Service e converter o Model de volta na resposta.
  - **Modelo de dados**: Deve ter o sufixo "DTO" para REST, "Type" para GraphQL ou "Event" para mensagens de entrada (ex: UserDTO, UserType ou UserEvent).
  - **Validações**: Responsável por validar formato, presença, tamanho, pattern, paginação, envelope técnico de mensagens e demais regras do contrato de entrada. Não deve validar invariantes de negócio.
  - **Erros**: Deve possuir um tratador global de exceções para converter erros classificados em respostas padronizadas (resposta da API ou ack/nack para mensagens). Não deve expor detalhes técnicos ou exceções de frameworks.
  - **Bean Type**: @RestController, @Controller, @KafkaListener, @RabbitListener, @JmsListener, @SqsListener, etc.
  - **Visibilidade**: Depende do Service e do Core (Models). Não acessa Datastore, Data, UseCase, nem aplica regra de domínio. DTOs, Types e Events não podem ser utilizados pelo Service, Core, Datastore ou Data.
  - **Listener de Mensagens (MessageListener)**: Adaptador de entrada para filas ou tópicos, par do Controller. Deve ser nomeado com o sufixo "MessageListener" (ex: UserMessageListener).
    - **Modelo de dados (Event)**: Contrato de entrada da Presentation, com o sufixo "Event" (ex: UserEvent). É convertido para Model do Core pelo `UserEventMapper` (`presentation.mapper`) antes da chamada ao Service, como um DTO.
    - **Fluxo**: Valida o envelope técnico, converte o payload em um Event, mapeia para Model do Core pelo `UserEventMapper` e chama um Service, inclusive para operações simples de persistência. Confirma a mensagem apenas após o processamento bem-sucedido; falhas seguem retry, dead-letter ou descarte do adaptador.
    - **Restrições**: Não pode conter regra de negócio, chamar UseCase diretamente, acessar Datastore, Repository, Entity ou modelos de APIs externas, nem iniciar transação de negócio.
    - **Bean Type**: @KafkaListener, @RabbitListener, @JmsListener, @SqsListener, etc.

- **Service**: É uma camada intermediária entre "presentation" e "core", responsável pela orquestração da operação e pelo controle transacional. Recebe e devolve apenas Models do Core, valores simples ou resultados definidos pelo Core; a conversão de/para `DTO`/`Type`/`Event` é feita pelos adaptadores de entrada, não pelo Service. O Service conhece apenas o contrato único do Datastore, nunca a origem dos dados ou os componentes internos utilizados para acessá-la.
O Service pode consumir diretamente o Datastore quando a operação consiste apenas em persistência, recuperação ou atualização de dados, sem aplicação de regras de negócio.
Quando houver regras de negócio, validações, cálculos ou transformações de domínio, o Service passa o Model como parâmetro ao UseCase apropriado. O UseCase processa o Model e devolve o Model processado, sem acessar Datastore, Repository, APIs externas ou qualquer outra infraestrutura.
O Service é responsável por preparar os dados necessários antes da chamada ao UseCase, usando o Datastore quando necessário, e por persistir ou recuperar os dados depois da execução do UseCase. Ao final, devolve o Model do Core (ou um resultado definido pelo Core) ao adaptador de entrada, que faz a conversão para `DTO`/`Type`/`Event`.
Se a operação puder ser implementada apenas com uma chamada ao Datastore, não crie um UseCase. Se for necessário aplicar regra de negócio, crie um UseCase puro e mantenha a orquestração de dados no Service.
  - **Bean Type**: @Service.
  - **Transações**: Responsável por definir o limite transacional da unidade de negócio e coordenar abertura, commit e rollback. A anotação transacional, quando utilizada, deve ficar no Service ou em um componente externo de infraestrutura que envolva o Service.
  - **Visibilidade**: O Service utiliza Models do Core e o contrato do Datastore. Não pode utilizar `DTO`, `Type` ou `Event` da Presentation, nem Entities, modelos de APIs externas, mensagens de saída ou componentes internos da Data.
  - **Erros**: Deve traduzir ou propagar erros conhecidos entre as fronteiras sem expor detalhes da Data. Não deve capturar erros inesperados para mascarar a falha.

- **Core**: O Core representa a lógica de negócios da aplicação e deve ser independente de frameworks, infraestrutura e Datastore. Contém as regras de negócio e validações necessárias para garantir a integridade dos dados e o correto funcionamento do sistema.
  - **Modelo de dados (Model)**: Não tem sufixo (ex: User).
  - **Validações**: Responsável por validar invariantes, regras de negócio e consistência do domínio. Essas validações devem ser independentes da forma de entrada e da tecnologia de persistência.
  - **Transações**: Não abre, confirma, reverte ou configura transações. O UseCase expressa a unidade lógica da operação por meio de sua regra, mas a transação é controlada pelo Service.
  - **Erros**: Deve lançar exceções de domínio para violações de regras e invariantes. Não pode depender de códigos HTTP, exceções de banco ou frameworks de infraestrutura.
  - **UseCase**: Representa uma operação de negócio específica, encapsulando a lógica necessária para processar um ou mais Models. Recebe um Model por parâmetro e devolve o Model processado. Deve ser stateless e não manter estado mutável entre chamadas; o estado da operação deve estar nos Models recebidos e retornados. Não acessa Datastore, Repository, APIs externas, filas ou qualquer outra infraestrutura. Deve ser nomeado com o sufixo "UseCase" (ex: UserUseCase). Em situações simples, pode agrupar operações intimamente relacionadas desde que mantenha uma única responsabilidade.
  - **Visibilidade**: O Core pode ser utilizado pela Presentation (Models), pelo Service e pelo Datastore. Essa visibilidade é unidirecional: o Core não pode importar ou depender dessas camadas. Models do Core não podem conter DTOs, Types, Events, Entities, modelos de APIs externas ou mensagens de saída.

- **Datastore**: É uma camada intermediária entre "Service" e "Data", responsável por consumir a fonte de dados adequada e utilizar os mappers de `data.mapper` para converter os Models do Core para os modelos da Data e vice-versa. Pode consumir internamente Repositories, ApiClients e componentes de mensageria, sem expor qual origem foi utilizada. Recebe um Model do Service, usa o mapper apropriado para convertê-lo ao modelo externo, acessa a origem e converte o resultado de volta para um Model antes de retorná-lo ao Service. O Datastore não é utilizado diretamente pelo UseCase.
  - **Bean Type**: @Component.
  - **Transações**: Participa da transação iniciada pelo Service e não deve abrir, confirmar ou reverter uma transação independente para cada operação interna.
  - **Visibilidade**: O contrato do Datastore deve ser visível para o Service e pode utilizar Models do Core internamente. Entities, modelos de APIs externas, mensagens de saída e componentes da Data não podem sair do Datastore em direção ao Service ou ao Core.
  - **Erros**: Deve traduzir erros de conversão, banco, APIs externas e mensageria para exceções de infraestrutura, preservando a causa original. Não deve definir respostas HTTP.

- **Data**: É a camada responsável pelo acesso a dados, seja ele um banco de dados relacional, NoSQL, uma API externa ou qualquer outra fonte de dados. Contém componentes especializados no acesso às diferentes fontes de dados ou sistemas externos.
  - **Visibilidade**: Repositories, ApiClients, MessagePublishers e os mappers de `data.mapper` só devem ser visíveis para o Datastore. Nenhum componente da Data deve ser visível diretamente para a Presentation, o Service ou o Core. O adaptador de entrada de mensagens (`MessageListener`) não pertence à Data; fica na Presentation.
  - **Validações**: Responsável por restrições específicas do armazenamento, como constraints, tipos, tamanhos, chaves e índices. Essas restrições não substituem as regras de negócio do Core.
  - **Erros**: Deve lançar ou encapsular exceções específicas da fonte de dados, preservando a causa técnica. Não deve expor essas exceções diretamente à Presentation, ao Service ou ao Core.
  - **Repositório (Repository)**: Responsável por realizar operações de persistência em bancos de dados relacionais ou NoSQL. Deve ser nomeado com o sufixo "Repository" (ex: UserRepository).
    - **Modelo de dados (Entity)**: Deve ter o sufixo "Entity" (ex: UserEntity).
    - **Bean Type**: @Repository.
  - **Cliente de API (ApiClient)**: Responsável por realizar operações de consumo de APIs externas. Deve ser nomeado com o sufixo "ApiClient" (ex: UserApiClient).
    - **Modelo de dados**: Deve ser exclusivo da Data e ter um prefixo que identifique a API (ex: GoogleUser). Não pode ser retornado diretamente ao Datastore, Service ou Presentation.
    - **Bean Type**: @Component.
  - **Publicador de Mensagens (MessagePublisher)**: Responsável por publicar mensagens em filas ou tópicos. Deve ser nomeado com o sufixo "MessagePublisher" (ex: UserMessagePublisher).
    - **Modelo de dados (Message)**: Deve ser exclusivo da Data e ter o sufixo "Message" (ex: UserMessage). Não pode ser retornado ao Datastore, ao Service ou à Presentation.
    - **Fluxo**: É acionado pela implementação do Datastore. Recebe um Model do Core convertido pelo `UserMessageMapper` e publica a mensagem. Service e Core não sabem que a operação envolve mensageria.
    - **Bean Type**: @Component.
  - **Mappers da Data (`data.mapper`)**: Convertem `Model` do Core ↔ `Entity`, modelo de API externa ou `Message`. Ficam em `data.mapper`, organizados por contexto (ex: `data.mapper.user`), são usados pela implementação do `Datastore` e não podem sair da `Data`. Podem depender do `Core` (para os Models) e dos pacotes técnicos da `Data`; nunca de `Service` ou `Presentation`.
    - **Nomes**: `UserEntityMapper`, `UserApiMapper`, `UserMessageMapper`.
