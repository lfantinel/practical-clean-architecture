# Practical Clean Architecture: Mapper Genérico

Este documento contém o exemplo completo de implementação do `Mapper<Source, Model>` descrito na seção "Mappers" do [README](README.md). As regras de uso (fronteiras autorizadas, escopo `prototype`, promoção para mapper nomeado e casos em que o mapper genérico não se aplica) ficam no README; aqui está apenas a implementação de referência.

O `Mapper<Source, Model>` serve às conversões em que a origem e o `Model` possuem os mesmos nomes de atributos, tipos compatíveis e a mesma estrutura. Ele guarda o par de tipos da fronteira e delega a cópia dos atributos a um `ObjectMapper` próprio, interno à classe.

## Classe `Mapper<Source, Model>`

```java
public class Mapper<Source, Model> {

    private static final ObjectMapper OBJECT_MAPPER = JsonMapper.builder()
            .addModule(new Jdk8Module())
            .addModule(new JavaTimeModule())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            .build();

    private final Class<Source> sourceType;
    private final Class<Model> modelType;

    public Mapper(Class<Source> sourceType, Class<Model> modelType) {
        this.sourceType = sourceType;
        this.modelType = modelType;
    }

    public Model toModel(Source source) {
        return source == null ? null : OBJECT_MAPPER.convertValue(source, modelType);
    }

    public Source fromModel(Model model) {
        return model == null ? null : OBJECT_MAPPER.convertValue(model, sourceType);
    }

    public List<Model> toModels(Collection<Source> sources) {
        return toModels(sources, null);
    }

    public List<Model> toModels(Collection<Source> sources, Consumer<Model> consumer) {
        if (sources == null) return null;
        List<Model> models = new ArrayList<>(sources.size());
        for (Source source : sources) {
            Model model = toModel(source);
            if (consumer != null) consumer.accept(model);
            models.add(model);
        }
        return models;
    }

    public List<Source> fromModels(Collection<Model> models) {
        return fromModels(models, null);
    }

    public List<Source> fromModels(Collection<Model> models, Consumer<Source> consumer) {
        if (models == null) return null;
        List<Source> sources = new ArrayList<>(models.size());
        for (Model model : models) {
            Source source = fromModel(model);
            if (consumer != null) consumer.accept(source);
            sources.add(source);
        }
        return sources;
    }
}
```

Os métodos de coleção retornam `null` quando a coleção de entrada é `null` e uma lista nova quando ela existe, inclusive quando está vazia. O `Consumer` é opcional: quando informado, é executado para cada item depois da conversão e antes de o item ser adicionado ao resultado. As sobrecargas sem `Consumer` executam somente a conversão automática.

O `ObjectMapper` fica dentro da classe, não como bean. Um `@Bean ObjectMapper` no contexto — com qualquer nome — desativa o `ObjectMapper` autoconfigurado do Spring Boot (`@ConditionalOnMissingBean`) e faz a configuração do mapper vazar para a (de)serialização HTTP.

## Fábrica: resolução dos tipos pelo ponto de injeção

A fábrica resolve os tipos `Source` e `Model` a partir do ponto de injeção, sem exigir uma subclasse por fronteira. Ela lê os genéricos declarados no campo ou parâmetro e entrega um `Mapper` já tipado:

```java
@Configuration
public class GenericMapperConfiguration {

    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public Mapper<?, ?> mapper(InjectionPoint injectionPoint) {
        ResolvableType type = injectionPoint.getField() != null
                ? ResolvableType.forField(injectionPoint.getField())
                : ResolvableType.forMethodParameter(injectionPoint.getMethodParameter());
        Class<?> sourceType = type.getGeneric(0).resolve();
        Class<?> modelType = type.getGeneric(1).resolve();

        if (sourceType == null || modelType == null) {
            throw new IllegalStateException(
                    "Mapper exige os tipos Source e Model concretos no ponto de injeção: " + injectionPoint);
        }
        return new Mapper<>(sourceType, modelType);
    }
}
```

## Uso no adaptador de entrada

Quando a conversão é totalmente automática, o adaptador de entrada injeta o `Mapper<Source, Model>` tipado direto e chama `toModel`/`fromModel`, sem uma classe de mapper escrita à mão:

```java
@RestController
public class UserController {

    private final Mapper<UserDTO, User> mapper;
    private final UserService service;

    public UserController(Mapper<UserDTO, User> mapper, UserService service) {
        this.mapper = mapper;
        this.service = service;
    }

    @PostMapping("/users")
    public UserDTO create(@RequestBody UserDTO dto) {
        User model = mapper.toModel(dto);
        model = service.save(model);
        return mapper.fromModel(model);
    }
}
```

Assim que a fronteira precisar de qualquer regra — campo com nome ou tipo diferente, transformação que não caiba no `Consumer`, `null` condicional, objeto aninhado não automático, campo sensível, ou acoplamento escondido que a cópia automática mascararia —, a conversão é promovida para um mapper nomeado. O mapper nomeado **estende** `Mapper<Source, Model>` (não o compõe como campo); o construtor chama `super(Source.class, Model.class)` com os tipos concretos da fronteira, sem depender da fábrica `prototype` — só quem injeta o genérico *sem* subclasse passa pela fábrica.

**Sobrescrita parcial** — parte dos campos é automática, um precisa de transformação. Sobrescreve `toModel`/`fromModel`, chamando `super` para a parte trivial:

```java
public class UserMapper extends Mapper<UserDTO, User> {

    public UserMapper() {
        super(UserDTO.class, User.class);
    }

    @Override
    public User toModel(UserDTO dto) {
        User user = super.toModel(dto);
        return user.comEmailNormalizado(dto.email().trim().toLowerCase());
    }
}
```

**Sobrescrita total** — a conversão automática não é possível ou não deve ser usada (ex.: dependeria de uma anotação pensada para outra fronteira). Sobrescreve `toModel`/`fromModel` por completo, sem chamar `super`:

```java
public class UserEntityMapper extends Mapper<UserEntity, User> {

    public UserEntityMapper() {
        super(UserEntity.class, User.class);
    }

    @Override
    public User toModel(UserEntity entity) {
        return new User(entity.getId(), entity.getName(), entity.getEmail());
    }

    @Override
    public UserEntity fromModel(User user) {
        return UserEntity.builder()
                .id(user.id())
                .name(user.name())
                .email(user.email())
                .build();
    }
}
```

Em qualquer um dos dois casos, `toModels`/`fromModels` (com `Consumer` opcional) continuam disponíveis sem reescrever nada — são herdados de `Mapper<Source, Model>` e chamam `this.toModel`/`this.fromModel`, que resolvem para a versão sobrescrita por despacho polimórfico. Por isso os métodos sobrescritos precisam se chamar exatamente `toModel`/`fromModel`: um nome próprio (`paraModel`, `converter`) não é override, e `toModels`/`fromModels` continuariam chamando a conversão automática da base.
