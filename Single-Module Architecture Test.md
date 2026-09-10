# Practical Clean Architecture: Single-Module Architecture Test

Este documento descreve o teste de dependências arquiteturais para uma aplicação Java single-module.

## Estrutura

O teste deve ficar no source set de testes do próprio módulo, em um pacote técnico separado das camadas da aplicação:

```text
src/
└── test/
    └── java/com/example/app/architecture/
        └── LayerDependencyTest.java
```

A dependência do ArchUnit deve ser exclusiva do escopo de testes. O pacote `architecture` serve para validar a aplicação e não deve ser utilizado pelo código de produção.

## Teste de Dependências Arquiteturais

O teste analisa o pacote raiz da aplicação e verifica os imports proibidos entre as camadas. Como todas as classes estão no mesmo módulo, a proteção depende do ArchUnit e da revisão dos pacotes; não há separação física de classpath entre `Presentation`, `Service`, `Core`, `Datastore` e `Data`, nem entre `Service` e `Core` dentro de `business` ou entre `Datastore` e os componentes técnicos dentro de `data`.

Os padrões `..core..`, `..service..` e `..datastore..` continuam válidos com a estrutura de três pacotes raiz, pois casam com qualquer prefixo (`..business.core..`, `..business.service..`, `..data.datastore..`).

A `Presentation` (incluindo `presentation.mapper` e o `MessageListener`) pode depender de `..core..` — os adaptadores de entrada convertem `DTO`/`Type`/`Event` ↔ `Model` e trocam Models com o `Service`. Não pode depender de `..datastore..` nem `..data..`. Na direção oposta, `..service..` não pode depender de `..presentation..`: o `DTO`/`Type`/`Event` nunca chega ao `Service`. A publicação de saída fica em `data.messaging` e os mappers Core↔Data em `data.mapper`; ambos são componentes técnicos de `data`, cobertos por `..data..` e pela lista de pacotes técnicos.

```java
package com.example.app.architecture;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

import com.tngtech.archunit.junit5.ArchTest;
import com.tngtech.archunit.junit5.AnalyzeClasses;
import com.tngtech.archunit.lang.ArchRule;

@AnalyzeClasses(packages = "com.example.app")
class LayerDependencyTest {
    @ArchTest
    static final ArchRule core_does_not_depend_on_external_layers = noClasses()
        .that().resideInAnyPackage("..core..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..service..", "..datastore..", "..data..");

    @ArchTest
    static final ArchRule service_does_not_depend_on_presentation_or_data_technical_components = noClasses()
        .that().resideInAnyPackage("..service..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..data.mapper..", "..data.persistence..", "..data.api..", "..data.messaging..");

    // Presentation pode depender de ..service.. e de ..core.. (Models trocados com o Service),
    // mas nunca do Datastore nem dos pacotes de dados.
    @ArchTest
    static final ArchRule presentation_does_not_depend_on_data = noClasses()
        .that().resideInAnyPackage("..presentation..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..datastore..", "..data..");

    @ArchTest
    static final ArchRule datastore_does_not_depend_on_outer_application_layers = noClasses()
        .that().resideInAnyPackage("..datastore..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..service..");

    @ArchTest
    static final ArchRule data_does_not_depend_on_outer_application_layers = noClasses()
        .that().resideInAnyPackage("..data..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..service..");

    @ArchTest
    static final ArchRule use_cases_do_not_depend_on_infrastructure = noClasses()
        .that().haveSimpleNameEndingWith("UseCase")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..service..", "..datastore..", "..data..");
}
```

O teste deve ser executado junto com a suíte normal no CI. Ele complementa os testes unitários do `Core` e os testes de integração das fronteiras, verificando que a organização de pacotes não introduziu dependências proibidas.
