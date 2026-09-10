# Practical Clean Architecture: Architecture Test

Este documento contém os testes arquiteturais usados no projeto multi-module: dependências entre camadas e convenções de nomenclatura.

## Dependências

O módulo `architecture` reúne as classes dos demais módulos apenas no escopo de testes, junto com o ArchUnit. Nenhuma dessas dependências deve vazar para o escopo de compilação. Ajuste a versão do ArchUnit para a estável mais recente e os nomes dos módulos (`presentation`, `business`, `data`) para os do projeto — se `Service` e `Core` forem módulos separados, liste os dois.

### Maven (`architecture/pom.xml`)

```xml
<dependencies>
    <dependency>
        <groupId>com.tngtech.archunit</groupId>
        <artifactId>archunit-junit5</artifactId>
        <version>1.3.0</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>com.example.app</groupId>
        <artifactId>presentation</artifactId>
        <version>${project.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.example.app</groupId>
        <artifactId>business</artifactId>
        <version>${project.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.example.app</groupId>
        <artifactId>data</artifactId>
        <version>${project.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle (`architecture/build.gradle`)

```groovy
dependencies {
    testImplementation 'com.tngtech.archunit:archunit-junit5:1.3.0'

    testImplementation project(':presentation')
    testImplementation project(':business')
    testImplementation project(':data')
}
```

## Teste de Dependências Arquiteturais

O teste fica em `architecture/src/test`, pois esse módulo reúne no classpath de teste as classes dos demais módulos sem atribuir à `presentation` uma responsabilidade de validação global. Ele verifica os imports, que são o ponto de controle das dependências proibidas durante a codificação.

Os módulos físicos são uma decisão do projeto. `Service` e `Core` podem ficar em módulos separados ou em um único módulo `business` com os pacotes `business.service` e `business.core`; da mesma forma, `Datastore` fica sob `data.datastore`. Os padrões `..service..`, `..core..` e `..datastore..` casam com qualquer prefixo e continuam válidos nas duas organizações.

A mensageria é dividida por direção: o `MessageListener` de entrada fica em `presentation.messaging` (adaptador de entrada, par do Controller) e a publicação de saída fica em `data.messaging`. As regras abaixo tratam `presentation.messaging` como os demais adaptadores de entrada e `data.messaging` como componente técnico da `Data`.

Os mappers Core↔Data ficam em `data.mapper` — componente técnico da `Data`, tratado como `data.persistence`, `data.api` e `data.messaging`. Ele pode depender de `..core..` (para os Models), nunca de `..service..` ou `..presentation..`.

A conversão `DTO`/`Type`/`Event` ↔ `Model` acontece nos adaptadores de entrada, em `presentation.mapper`. Por isso os adaptadores de entrada podem depender de `..core..` (Models trocados com o `Service`), mas não de `..data..`; e o `..service..` não pode depender de `..presentation..` — o `Service` só recebe e devolve Models do Core.

O conjunto de regras é o mesmo do projeto single-module — a separação física em módulos impede parte das violações já na compilação, mas os limites internos (`presentation.mapper`, `data.datastore` versus componentes técnicos de `data`, sufixos, anotações, isolamento do `Core`) continuam dependendo do ArchUnit. As regras ficam em duas classes no pacote `architecture`: `LayerDependencyTest` e `NamingConventionTest`.

As regras de `LayerDependencyTest` estão agrupadas em cinco blocos:

1. Direção das dependências entre camadas, incluindo a proibição de a `Presentation` chamar `UseCase` diretamente e de `..datastore..`/`..data..` dependerem do `Service` ou da `Presentation`.
2. Isolamento do `Core` contra frameworks e infraestrutura: o `Core` só pode depender do próprio domínio e da biblioteca padrão.
3. Limite dos modelos por sufixo: `DTO` preso à `Presentation`; `Entity` e `Message` de saída presos à `Data`.
4. Transação restrita ao `Service`.
5. Ausência de ciclos entre os pacotes raiz.

As regras de sufixo, pacote e anotação de cada componente ficam em `NamingConventionTest` (ver adiante).

O import é feito com `ImportOption.DoNotIncludeTests` para o próprio pacote `architecture` não entrar na análise.

```java
package com.example.app.architecture;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.methods;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;
import static com.tngtech.archunit.library.dependencies.SlicesRuleDefinition.slices;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit5.ArchTest;
import com.tngtech.archunit.junit5.AnalyzeClasses;
import com.tngtech.archunit.lang.ArchRule;

@AnalyzeClasses(packages = "com.example.app", importOptions = ImportOption.DoNotIncludeTests.class)
class LayerDependencyTest {

    // 1. Direção das dependências entre camadas

    @ArchTest
    static final ArchRule core_nao_depende_de_camadas_externas = noClasses()
        .that().resideInAnyPackage("..core..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..service..", "..datastore..", "..data..");

    @ArchTest
    static final ArchRule service_nao_depende_de_presentation_nem_de_componentes_tecnicos_de_data = noClasses()
        .that().resideInAnyPackage("..service..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..data.mapper..", "..data.persistence..", "..data.api..", "..data.messaging..");

    // Presentation pode depender de ..service.. e de ..core.. (Models trocados com o Service),
    // mas nunca do Datastore nem dos pacotes de dados. Cobre também presentation.mapper.
    @ArchTest
    static final ArchRule presentation_nao_depende_de_datastore_nem_de_data = noClasses()
        .that().resideInAnyPackage("..presentation..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..datastore..", "..data..");

    // Presentation troca Models com o Service, mas nunca chama o UseCase diretamente.
    @ArchTest
    static final ArchRule presentation_nao_chama_use_case = noClasses()
        .that().resideInAnyPackage("..presentation..")
        .should().dependOnClassesThat()
        .haveSimpleNameEndingWith("UseCase");

    @ArchTest
    static final ArchRule datastore_nao_depende_de_service_nem_de_presentation = noClasses()
        .that().resideInAnyPackage("..datastore..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..service..");

    @ArchTest
    static final ArchRule data_nao_depende_de_service_nem_de_presentation = noClasses()
        .that().resideInAnyPackage("..data..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..service..");

    @ArchTest
    static final ArchRule use_case_nao_depende_de_infraestrutura = noClasses()
        .that().haveSimpleNameEndingWith("UseCase")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..service..", "..datastore..", "..data..");

    // 2. Isolamento do Core contra frameworks e infraestrutura

    // Ajuste a allowlist se o domínio tiver um kernel compartilhado próprio.
    @ArchTest
    static final ArchRule core_so_depende_do_dominio_e_da_biblioteca_padrao = classes()
        .that().resideInAnyPackage("..business.core..")
        .should().onlyDependOnClassesThat()
        .resideInAnyPackage("..business.core..", "java..");

    // 3. Limite dos modelos por sufixo

    @ArchTest
    static final ArchRule dto_nao_sai_da_presentation = noClasses()
        .that().resideOutsideOfPackage("..presentation..")
        .should().dependOnClassesThat().haveSimpleNameEndingWith("DTO");

    @ArchTest
    static final ArchRule entity_nao_sai_da_data = noClasses()
        .that().resideOutsideOfPackage("..data..")
        .should().dependOnClassesThat().haveSimpleNameEndingWith("Entity");

    @ArchTest
    static final ArchRule message_de_saida_nao_sai_da_data = noClasses()
        .that().resideOutsideOfPackage("..data..")
        .should().dependOnClassesThat().haveSimpleNameEndingWith("Message");

    // 4. Transação restrita ao Service

    @ArchTest
    static final ArchRule apenas_classes_do_service_sao_transacionais = classes()
        .that().resideOutsideOfPackage("..business.service..")
        .should().notBeAnnotatedWith("org.springframework.transaction.annotation.Transactional")
        .andShould().notBeAnnotatedWith("jakarta.transaction.Transactional");

    @ArchTest
    static final ArchRule apenas_metodos_do_service_sao_transacionais = methods()
        .that().areDeclaredInClassesThat().resideOutsideOfPackage("..business.service..")
        .should().notBeAnnotatedWith("org.springframework.transaction.annotation.Transactional")
        .andShould().notBeAnnotatedWith("jakarta.transaction.Transactional");

    // 5. Ausência de ciclos entre os pacotes raiz

    @ArchTest
    static final ArchRule pacotes_raiz_livres_de_ciclos = slices()
        .matching("com.example.app.(*)..")
        .should().beFreeOfCycles();
}
```

## Teste de Convenções de Nomenclatura

Verifica a tabela de Nomenclatura e Beans: cada componente identificado pelo sufixo do nome deve residir no pacote esperado e, quando houver, carregar a anotação Spring correspondente. Além do sufixo `Controller`, toda classe anotada com `@RestController` ou `@Controller` deve estar em `..presentation..`. As regras de anotação assumem componentes concretos, sem divisão interface/implementação; ajuste-as se o projeto usar esse padrão.

```java
package com.example.app.architecture;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit5.ArchTest;
import com.tngtech.archunit.junit5.AnalyzeClasses;
import com.tngtech.archunit.lang.ArchRule;

@AnalyzeClasses(packages = "com.example.app", importOptions = ImportOption.DoNotIncludeTests.class)
class NamingConventionTest {

    @ArchTest
    static final ArchRule controllers_ficam_nos_adaptadores_de_entrada = classes()
        .that().haveSimpleNameEndingWith("Controller")
        .should().resideInAnyPackage("..presentation.rest..", "..presentation.graphql..");

    @ArchTest
    static final ArchRule controllers_anotados_ficam_em_presentation = classes()
        .that().areAnnotatedWith("org.springframework.web.bind.annotation.RestController")
        .or().areAnnotatedWith("org.springframework.stereotype.Controller")
        .should().resideInAPackage("..presentation..");

    @ArchTest
    static final ArchRule message_listeners_ficam_em_presentation_messaging = classes()
        .that().haveSimpleNameEndingWith("MessageListener")
        .should().resideInAPackage("..presentation.messaging..");

    @ArchTest
    static final ArchRule services_ficam_em_business_service = classes()
        .that().haveSimpleNameEndingWith("Service")
        .should().resideInAPackage("..business.service..")
        .andShould().beAnnotatedWith("org.springframework.stereotype.Service");

    @ArchTest
    static final ArchRule use_cases_ficam_em_business_core = classes()
        .that().haveSimpleNameEndingWith("UseCase")
        .should().resideInAPackage("..business.core..");

    @ArchTest
    static final ArchRule datastores_ficam_em_data_datastore = classes()
        .that().haveSimpleNameEndingWith("Datastore")
        .should().resideInAPackage("..data.datastore..");

    @ArchTest
    static final ArchRule repositories_ficam_em_data_persistence = classes()
        .that().haveSimpleNameEndingWith("Repository")
        .should().resideInAPackage("..data.persistence..")
        .andShould().beAnnotatedWith("org.springframework.stereotype.Repository");

    @ArchTest
    static final ArchRule entities_ficam_em_data_persistence = classes()
        .that().haveSimpleNameEndingWith("Entity")
        .should().resideInAPackage("..data.persistence..");

    @ArchTest
    static final ArchRule api_clients_ficam_em_data_api = classes()
        .that().haveSimpleNameEndingWith("ApiClient")
        .should().resideInAPackage("..data.api..");

    @ArchTest
    static final ArchRule message_publishers_ficam_em_data_messaging = classes()
        .that().haveSimpleNameEndingWith("MessagePublisher")
        .should().resideInAPackage("..data.messaging..");
}
```

As duas classes devem permanecer em `architecture/src/test` e ser executadas no CI junto com a suíte normal. As dependências entre os módulos impedem parte das violações durante a compilação; o ArchUnit complementa essa proteção verificando imports, pacotes, nomes, anotações e vazamentos de modelo proibidos no conjunto da aplicação. Se alguma regra `should` puder não casar nenhuma classe, habilite `archunit.properties` com `failOnEmptyShould=false`.
