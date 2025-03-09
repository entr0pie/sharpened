---
tags:
  - arquitetura
  - the-hard-parts
date: 2025-03-09
---
## O que é arquitetura?

- As coisas difíceis de alterar mais tarde.
- O objetivo de qualquer arquiteto é encontrar o melhor conjunto de Trade-Offs.
- Arquitetura de Software são problemas únicos. Não tem no Stack Overflow!
## Papel dos Dados

- Os dados são o bem mais valioso de qualquer empresa.
- Os dados vivem além dos próprios sistemas.
- Classificamos os dados em dois tipos: **Operacionais e Analíticos**.
	- **Operacionais**: **pra empresa funcionar** corretamente!
	- **Analíticos**: **pra gerar mais lucro** a longo prazo.
## Registrando Decisões (ADRs)

- Os [ADRs (Architecture Decision Records)](https://medium.com/@jhonywalkeer/guia-completo-sobre-architecture-decision-records-adr-defini%C3%A7%C3%A3o-e-melhores-pr%C3%A1ticas-f63e66d33e6) servem para **documentar decisões de arquitetura**.
- Ótimo para rastrear decisões tomadas antes e formalizar elas.
- São estruturados assim:

```
ADR: uma frase resumo da decisão tomada.

Contexto: breve descrição dos problemas e soluções alternativas.

Decisão: qual decisão de arquitetura foi tomada para os problemas. Justificativa detalhada.

Consequências: quais consequências de aplicarmos essa decisão? Também lista os Trade-Offs da decisão tomada.
```
## Fitness Functions

- Mecanismos para **garantir que as decisões de arquitetura estão sendo aplicadas** nos serviços!
- **Pode ser qualquer mecanismo!** Testes nos projetos, Code Review, Auditorias ao longo do tempo, etc...
- **Deve medir características objetivas**:
	- "Se o sistema é rápido" ou "se o sistema aguenta muitos usuários" NÃO É OBJETIVO!
	- Implantabilidade, testabilidade, tempo de ciclo e elasticidade É OBJETIVO!
- São **classificadas em** duas: **Atômicas e Holísticas**
	- **Atômicas**: testa **uma única característica** da arquitetura.
	- **Holísticas**: validam uma **combinação de características**.

### Um exemplo em Spring Boot

- Em Java, podemos utilizar da biblioteca [ArchUnit](https://www.archunit.org) para testar a estrutura arquitetural do nosso projeto.
- Imagine uma aplicação web em camadas com a seguinte estrutura:

![[Pasted image 20250309165502.png]]
- Podemos garantir de que o app seguirá nessa estrutura escrevendo uma *fitness function* com o ArchUnit.
- Primeiro, criamos algumas classes de exemplo do nosso app:

```java
package thundera.thehardparts.arch_unit_example.controller;

@RestController  
@RequestMapping(path = "/hello")  
public class HelloWorldController {  
  
    private final HelloWorldService service;  
  
    public HelloWorldController(HelloWorldService service) {  
        this.service = service;  
    }  
  
    @GetMapping()  
    public ResponseEntity<String> sayHello() {  
        return ResponseEntity.ok(this.service.sayHello());  
    }  
}
```

```java
package thundera.thehardparts.arch_unit_example.service;

@Service  
public class HelloWorldService {
    private final HelloWorldRepository repository;  
  
    public HelloWorldService(HelloWorldRepository repository) {  
        this.repository = repository;  
    }  
  
    public String sayHello() {  
        return this.repository.sayHello();  
    }  
}
```

```java
package thundera.thehardparts.arch_unit_example.repository;  
  
@Component  
public class HelloWorldRepository {  
    public String sayHello() {  
        return "Hello, world!";  
    }  
}
```

- Para testarmos a estrutura do nosso projeto, escrevemos a *fitness function*:

```java
@Test  
public void shouldImportCorrectPackagesByDomain() {  
    JavaClasses jc = new ClassFileImporter().importPackages("thundera.thehardparts.arch_unit_example");  

	// Primeira forma
    ArchRule controllersShouldDependOnlyOnServices = classes()  
            .that()  
            .resideInAPackage("..controller..")  
            .should()  
            .onlyDependOnClassesThat()  
            .resideInAnyPackage("..service..", "java..", "javax..", "org.springframework..");  
  
    controllersShouldDependOnlyOnServices.check(jc);  

	// Segunda forma
    ArchRule controllersShouldNotDependOnRepositories = noClasses()  
            .that()  
            .resideInAPackage("..controller..")  
            .should()  
            .dependOnClassesThat()  
            .resideInAPackage("..repository..");  
  
    controllersShouldNotDependOnRepositories.check(jc);  
}
```

- Dessa forma, conseguimos garantir que somente os pacotes de determinados domínios podem importar outras classes.
- Outra forma que podemos utilizar é definindo a arquitetura da nossa aplicação:

```java
@Test
public void shouldImportCorrectPackagesByDomainAgain() {  
    JavaClasses jc = new ClassFileImporter().importPackages("thundera.thehardparts.arch_unit_example");  
  
    LayeredArchitecture arch = layeredArchitecture()  
            .consideringOnlyDependenciesInLayers()  
            .layer("Controllers").definedBy("..controller..")  
            .layer("Services").definedBy("..service..")  
            .layer("Repositories").definedBy("..repository")  
            .whereLayer("Controllers").mayNotBeAccessedByAnyLayer()  
            .whereLayer("Services").mayOnlyBeAccessedByLayers("Controllers")  
            .whereLayer("Repositories").mayOnlyBeAccessedByLayers("Services");  
  
    arch.check(jc);  
}
```

