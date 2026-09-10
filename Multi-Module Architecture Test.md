# Practical Clean Architecture: Architecture Test

Este documento contém o teste de dependências arquiteturais usado no projeto multi-module.

## Teste de Dependências Arquiteturais

O teste fica em `architecture/src/test`, pois esse módulo reúne no classpath de teste as classes dos demais módulos sem atribuir à `presentation` uma responsabilidade de validação global. Ele verifica os imports, que são o ponto de controle das dependências proibidas durante a codificação.

Os módulos físicos são uma decisão do projeto. `Service` e `Core` podem ficar em módulos separados ou em um único módulo `business` com os pacotes `business.service` e `business.core`; da mesma forma, `Datastore` fica sob `data.datastore`. Os padrões `..service..`, `..core..` e `..datastore..` casam com qualquer prefixo e continuam válidos nas duas organizações.

A mensageria é dividida por direção: o `MessageListener` de entrada fica em `presentation.messaging` (adaptador de entrada, par do Controller) e a publicação de saída fica em `data.messaging`. As regras abaixo tratam `presentation.messaging` como os demais adaptadores de entrada e `data.messaging` como componente técnico da `Data`.

Os mappers Core↔Data ficam em `data.mapper` — componente técnico da `Data`, tratado como `data.persistence`, `data.api` e `data.messaging`. Ele pode depender de `..core..` (para os Models), nunca de `..service..` ou `..presentation..`.

A conversão `DTO`/`Type`/`Event` ↔ `Model` acontece nos adaptadores de entrada, em `presentation.mapper`. Por isso os adaptadores de entrada podem depender de `..core..` (Models trocados com o `Service`), mas não de `..data..`; e o `..service..` não pode depender de `..presentation..` — o `Service` só recebe e devolve Models do Core.

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
        .resideInAnyPackage("..presentation..", "..service..", "..data..");

    // Adaptadores de entrada trocam Models do Core com o Service (conversão em presentation.mapper),
    // por isso podem depender de ..core.., mas nunca de ..data...
    @ArchTest
    static final ArchRule entry_adapters_do_not_depend_on_data = noClasses()
        .that().resideInAnyPackage("..presentation.rest..", "..presentation.graphql..", "..presentation.messaging..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..data..");

    @ArchTest
    static final ArchRule service_does_not_depend_on_presentation = noClasses()
        .that().resideInAnyPackage("..service..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..");

    @ArchTest
    static final ArchRule service_does_not_depend_on_data_technical_components = noClasses()
        .that().resideInAnyPackage("..service..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..data.mapper..", "..data.persistence..", "..data.api..", "..data.messaging..");

    @ArchTest
    static final ArchRule data_does_not_depend_on_presentation = noClasses()
        .that().resideInAnyPackage("..data..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..");

    @ArchTest
    static final ArchRule data_technical_components_do_not_depend_on_outer_layers = noClasses()
        .that().resideInAnyPackage("..data.mapper..", "..data.persistence..", "..data.api..", "..data.messaging..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..presentation..", "..service..");
}
```

O teste deve permanecer em `architecture/src/test` e ser executado no CI junto com a suíte normal. As dependências entre os módulos impedem parte das violações durante a compilação; o ArchUnit complementa essa proteção verificando imports, pacotes e tipos proibidos no conjunto da aplicação.
