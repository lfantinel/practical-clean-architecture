# Practical Clean Architecture: Single-Module Architecture Test

Este documento descreve os testes arquiteturais para uma aplicação Java single-module: dependências entre camadas e convenções de nomenclatura.

## Estrutura

O teste deve ficar no source set de testes do próprio módulo, em um pacote técnico separado das camadas da aplicação:

```text
src/
└── test/
    └── java/com/example/app/architecture/
        ├── LayerDependencyTest.java
        └── NamingConventionTest.java
```

`LayerDependencyTest` cobre a direção das dependências, o isolamento do `Core`, o limite dos modelos e as transações. `NamingConventionTest` cobre sufixo, pacote e anotação de cada componente. As duas classes ficam no mesmo pacote `architecture` e rodam na mesma suíte.

A dependência do ArchUnit deve ser exclusiva do escopo de testes. O pacote `architecture` serve para validar a aplicação e não deve ser utilizado pelo código de produção.

## Dependências

O teste usa o ArchUnit com a integração JUnit 5, apenas no escopo de testes. Ajuste a versão para a estável mais recente.

### Maven

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.3.0</version>
    <scope>test</scope>
</dependency>
```

### Gradle (Groovy DSL)

```groovy
testImplementation 'com.tngtech.archunit:archunit-junit5:1.3.0'
```

### Gradle (Kotlin DSL)

```kotlin
testImplementation("com.tngtech.archunit:archunit-junit5:1.3.0")
```

## Teste de Dependências Arquiteturais

O teste analisa o pacote raiz da aplicação e verifica os imports proibidos entre as camadas. Como todas as classes estão no mesmo módulo, a proteção depende do ArchUnit e da revisão dos pacotes; não há separação física de classpath entre `Presentation`, `Service`, `Core`, `Datastore` e `Data`, nem entre `Service` e `Core` dentro de `business` ou entre `Datastore` e os componentes técnicos dentro de `data`.

Os padrões `..core..`, `..service..` e `..datastore..` continuam válidos com a estrutura de três pacotes raiz, pois casam com qualquer prefixo (`..business.core..`, `..business.service..`, `..data.datastore..`).

A `Presentation` (incluindo `presentation.mapper` e o `MessageListener`) pode depender de `..core..` — os adaptadores de entrada convertem `DTO`/`Type`/`Event` ↔ `Model` e trocam Models com o `Service`. Não pode depender de `..datastore..` nem `..data..`. Na direção oposta, `..service..` não pode depender de `..presentation..`: o `DTO`/`Type`/`Event` nunca chega ao `Service`. A publicação de saída fica em `data.messaging` e os mappers Core↔Data em `data.mapper`; ambos são componentes técnicos de `data`, cobertos por `..data..` e pela lista de pacotes técnicos.

As regras de `LayerDependencyTest` estão agrupadas em cinco blocos:

1. Direção das dependências entre camadas, incluindo a proibição de a `Presentation` chamar `UseCase` diretamente.
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
    // mas nunca do Datastore nem dos pacotes de dados.
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

As duas classes devem ser executadas junto com a suíte normal no CI. Elas complementam os testes unitários do `Core` e os testes de integração das fronteiras, verificando que a organização de pacotes não introduziu dependências, nomes, anotações ou vazamentos de modelo proibidos. Se alguma regra `should` puder não casar nenhuma classe em um projeto novo, habilite `archunit.properties` com `failOnEmptyShould=false`.
