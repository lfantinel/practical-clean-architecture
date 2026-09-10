# Practical Clean Architecture: Mapper Genérico

Este documento contém o exemplo completo de implementação do `Mapper<From, To>` descrito na seção "Mappers" do [README](README.md). As regras de uso (fronteiras autorizadas, escopo `prototype`, promoção para mapper nomeado e casos em que o mapper genérico não se aplica) ficam no README; aqui está apenas a implementação de referência.

O `Mapper<From, To>` serve às conversões em que os dois modelos possuem os mesmos nomes de atributos, tipos compatíveis e a mesma estrutura. Ele guarda o par de tipos da fronteira e delega a cópia dos atributos a um `ObjectMapper` próprio, interno à classe.

## Classe `Mapper<From, To>`

```java
public class Mapper<From, To> {

    private static final ObjectMapper OBJECT_MAPPER = JsonMapper.builder()
            .addModule(new Jdk8Module())
            .addModule(new JavaTimeModule())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            .build();

    private final Class<From> fromType;
    private final Class<To> toType;

    public Mapper(Class<From> fromType, Class<To> toType) {
        this.fromType = fromType;
        this.toType = toType;
    }

    public To to(From source) {
        return source == null ? null : OBJECT_MAPPER.convertValue(source, toType);
    }

    public From from(To source) {
        return source == null ? null : OBJECT_MAPPER.convertValue(source, fromType);
    }
}
```

O `ObjectMapper` fica dentro da classe, não como bean. Um `@Bean ObjectMapper` no contexto — com qualquer nome — desativa o `ObjectMapper` autoconfigurado do Spring Boot (`@ConditionalOnMissingBean`) e faz a configuração do mapper vazar para a (de)serialização HTTP.

## Fábrica: resolução dos tipos pelo ponto de injeção

A fábrica resolve os tipos `From` e `To` a partir do ponto de injeção, sem exigir uma subclasse por fronteira. Ela lê os genéricos declarados no campo ou parâmetro e entrega um `Mapper` já tipado:

```java
@Configuration
public class GenericMapperConfiguration {

    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public Mapper<?, ?> mapper(InjectionPoint injectionPoint) {
        ResolvableType type = injectionPoint.getResolvableType();
        Class<?> fromType = type.getGeneric(0).resolve();
        Class<?> toType = type.getGeneric(1).resolve();

        if (fromType == null || toType == null) {
            throw new IllegalStateException(
                    "Mapper exige os tipos From e To concretos no ponto de injeção: " + injectionPoint);
        }
        return new Mapper<>(fromType, toType);
    }
}
```

## Uso no adaptador de entrada

Quando a conversão é totalmente automática, o adaptador de entrada injeta o `Mapper<From, To>` tipado direto e chama `to`/`from`, sem uma classe de mapper escrita à mão:

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
        User model = mapper.to(dto);
        model = service.save(model);
        return mapper.from(model);
    }
}
```

Assim que a fronteira precisar de qualquer regra — campo com nome ou tipo diferente, transformação, `null` condicional, objeto aninhado não automático, campo sensível —, a conversão é promovida para um mapper nomeado em `presentation.mapper` ou `data.mapper` (`UserMapper`, `UserEntityMapper`), com métodos explícitos. O mapper nomeado pode continuar delegando ao `Mapper<From, To>` a parte trivial e tratar à parte só os campos com regra. O adaptador passa a injetar o mapper nomeado no lugar do genérico; a assinatura dos métodos de negócio não muda.
