---
layout: [post, post-xml]                                                                # Pflichtfeld. Nicht ändern!
title:  "Contract-Driven Development: Hands-On am Beispiel von Spring Cloud Contract"   # Pflichtfeld. Bitte einen Titel für den Blog Post angeben.
date:   2021-05-19 08:00                                                                # Pflichtfeld. Format "YYYY-MM-DD HH:MM". Muss für Veröffentlichung in der Vergangenheit liegen. (Für Preview egal)
modified_date:                                                                          # Optional. Muss angegeben werden, wenn eine bestehende Datei geändert wird.
author: tgraefenstein                                                                   # Pflichtfeld. Es muss in der "authors.yml" einen Eintrag mit diesem Namen geben.
categories: [Softwareentwicklung]                                                       # Pflichtfeld. Maximal eine der angegebenen Kategorien verwenden.
tags: [Spring, Cloud, Contract-Driven Development, Microservices, Java]                 # Bitte auf Großschreibung achten.
---
In modernen Softwaresystemen kommt man mittlerweile kaum noch daran vorbei, mit anderen Systemen zu kommunizieren. 
Je mehr Parteien beteiligt sind, desto mehr Schnittstellen müssen abgestimmt und angebunden werden.
Doch wie können wir die korrekte Integration von Schnittstellen zu jeder Zeit sicherstellen?
Dazu bietet sich Contract-Driven Development an, welches wir uns mittels Spring Cloud Contract näher anschauen wollen.

# Spring Cloud Contract
Führt eine Partei eine nicht kompatible Änderung einer angebotenen Schnittstelle durch, so führt dies zwangsweise zu Fehlern, im schlimmsten Falle sogar zum Ausfall des Gesamtsystems.
Um dem entgegenzuwirken, bietet es sich an, Schnittstellenverträge (Contracts) zwischen den beteiligten Systemen auszuhandeln.
Daraus ergibt sich die Möglichkeit für Nutzende (Consumer) und Anbietende (Producer) Contract-Tests zu entwickeln, um die korrekte Anbindung und Funktionalität von Schnittstellen sicherzustellen.
Spring Cloud Contract ist ein Spring Cloud Projekt, welches eine einfache und strukturierte Möglichkeit bietet, Contract-Tests umzusetzen.
Contracts lassen sich über eine domänenspezifische Sprache (DSL) in Form von Groovy oder YAML definieren.
Im Folgenden schauen wir uns einmal an, wie Contracts in auf Spring basierenden Services mittels Kommunikation über HTTP umgesetzt werden können.

## Ausgangsszenario
Ein Service benötigt in seinem Kontext die Information, dass ein bestimmtes Jahr ein Schaltjahr ist.
Dazu bietet ein anderer Service eine Schnittstelle an, welche das Jahr entgegennimmt und prüft, ob es ein Schaltjahr ist.
Wir bezeichnen den anfragenden Service daher als Consumer und den Service, der die Schnittstelle anbietet, als Producer. 
Der Consumer konsumiert also die Schnittstelle des Producers. 
Zu Demonstrationszwecken bietet der Consumer auch eine Schnittstelle an, die lediglich das Ergebnis der Anfrage an den Client zurückgibt.
Im Folgenden schauen wir uns eine mögliche Umsetzung von Contracts mittels Spring Cloud Contract an.

## Producer Setup
Der Producer ist in unserem Beispiel ein Spring Boot Microservice.

### GET-Schnittstelle 
Der Producer bietet die folgende GET-Schnittstelle an.
Diese gibt unter Angabe des Parameters `year` an, ob das angefragte Jahr ein Schaltjahr ist:

```java
@RestController
public class LeapYearController {

  @GetMapping(value = "/leap-year", consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
  public LeapYearResponse checkLeap(@RequestParam(value = "year") int year) {
    final LeapYearResponse response = new LeapYearResponse();
    final boolean isLeap = year % 4 == 0 && (year % 100 != 0 || year % 400 == 0);
    response.setLeap(isLeap);
    return response;
  }

}
```

Anhand dieser Schnittstelle können wir unterschiedliche Aspekte ermitteln, die für die Nutzung dieser Schnittstelle relevant sind.
Diese Aspekte werden wir im weiteren Verlauf zur Definition der Contracts verwenden:
1. Die Schnittstelle wird mit der HTTP-Methode `GET` angefragt.
2. Die Schnittstelle erwartet den Pflichtparameter `year` als Zahl.
3. Die Schnittstelle antwortet im JSON-Format. 
4. Das Response-JSON enthält das Feld `leap` vom Typ `boolean`.

### Basisklasse für Tests
Um nun Contracts zu definieren, müssen wir zunächst das Setup dafür aufbauen. 
Dazu definieren wir eine Basisklasse für Tests, um den Spring Kontext zu laden.
Die aus den Contracts automatisch generierten Tests werden von dieser Klasse erben.

```java
@SpringBootTest
public class BaseTestClass {

  @Autowired
  private WebApplicationContext webApplicationContext;

  @BeforeEach
  public void setup() {
    RestAssuredMockMvc.webAppContextSetup(webApplicationContext);
  }

}
```

### Gradle Setup 
Im Producer-Service benötigen wir das Plugin [spring-cloud-contract-gradle-plugin](https://plugins.gradle.org/plugin/org.springframework.cloud.contract).
Das Plugin wird sich um die Generierung der Tests und Stubs kümmern:

```yaml
buildscript {
  dependencies {
    classpath 'org.springframework.cloud:spring-cloud-contract-gradle-plugin:3.0.2'
  }
}

plugins {
  id "org.springframework.cloud.contract" version "3.0.2"
}
```

Anschließend müssen wir das Plugin konfigurieren, indem das zu verwendende Testframework und die zuvor definierte Testbasisklasse angegeben wird:

```yaml
contracts {
  testFramework = TestFramework.JUNIT5
  baseClassForTests = 'com.adesso.contract.producer.BaseTestClass'
}
```

Die generierten Stubs müssen allen potenziellen Consumer-Services zur Verfügung gestellt werden.
In realen Umgebungen würde man die Stubs üblicherweise in ein Artifactory o. ä. hochladen.
Es gibt auch die Möglichkeit, sie in Form von Docker-Containern auszuliefern (siehe auch [Spring Cloud Contract - Docker Project](https://cloud.spring.io/spring-cloud-contract/reference/html/docker-project.html)).
Wir verwenden jedoch für die Generierung der Stubs der Einfachheit halber das lokale Maven-Repository:

```yaml
plugins {
  id 'maven-publish'
}

ext {
  contractsDir = file("mappings")
  stubsOutputDirRoot = file("${project.buildDir}/production/${project.name}-stubs/")
}

publishing {
  publications {
    stubs(MavenPublication) {
      artifactId "${project.name}"
      artifact verifierStubsJar
    }
  }
}
```

### Producer Contract
Da nun das Setup im Producer-Service steht, können wir den Contract definieren.
Der Contract sollte alle relevanten Aspekte enthalten, die zur Kommunikation relevant sind wie z.B. HTTP-Methode oder verpflichtende Parameter.
All jene Aspekte, die jedoch irrelevant sind, sollten auch nicht Teil des Contracts sein (z.B. nicht ausgewertete Header), da diese zu unerwarteten Vertragsbrüchen führen können.
Man sollte immer im Hinterkopf behalten, dass ein Contract nicht das Schema, sondern eher den Anwendungsfall abbilden soll.
Die relevanten Aspekte hatten wir ja bereits bei Betrachtung der GET-Schnittstelle ermittelt.
Daher definieren wir nun folgenden Contract:

```yaml
description: Given a year it returns the leap year information for that year
name: shouldReturnLeapYearInfo
request:
  method: GET
  url: /leap-year
  headers:
    Content-Type: application/json
  queryParameters:
    year: 2020
  matchers:
    queryParameters:
      - key: year
        type: matching
        value: "(\\d)+"
response:
  status: 200
  body:
    leap: true
  matchers:
    body:
      - path: $.['leap']
        type: by_type
        value: any_boolean
  headers:
    Content-Type: application/json
```

Mit diesem Contract wird die folgende Vereinbarung geschlossen:

*Für eine **GET-Anfrage** unter der URL`/leap-year` mit dem Parameter `year`, welcher eine Ganzzahl sein muss, besteht die **Antwort** aus dem HTTP-Status 200 und einem JSON, welches das Attribut `leap` mit einem booleschen Wert enthalten muss.*

Zu Beachten ist hierbei, dass wir mit den expliziten Angaben (z.B. `year: 2020` oder `leap: true`) konkrete Beispiele und mit `matchers` die erlaubten Datenformate von Request- oder Response-Parametern angeben können. 

Falls es für die Integration der beteiligten Parteien notwendig ist, können wir auch Fehlerfälle in Form von Contracts abbilden.
Hier haben wir beispielsweise einen Contract, der aussagt, dass beim Fehlen des Query-Parameters `year` mit einem Fehler geantwortet wird:
```yaml
description: Not given a year it returns a bad request error
name: shouldReturnBadRequestWhenRequestParamMissing
request:
  method: GET
  url: /leap-year
  headers:
    Content-Type: application/json
response:
  status: 400
```

### Generierung der Tests und Stubs
Da nun alle Contracts geschrieben sind, können wir anhand der Contracts die Tests generieren und ausführen lassen:
```console
gradle clean build publishToMavenLocal
```

Spring Cloud erstellt nun aus allen existierenden Contracts Tests und führt diese anschließend aus.
Hier sehen wir die automatisch generierten Tests aus den zwei zuvor definierten Contracts:
```java
public class LeapYearControllerTest extends BaseTestClass {

  @Test
  public void validate_shouldReturnBadRequestWhenRequestParamMissing() throws Exception {
    // given:
    MockMvcRequestSpecification request = given()
      .header("Content-Type", "application/json");

    // when:
    ResponseOptions response = given().spec(request)
        .get("/leap-year");

    // then:
    assertThat(response.statusCode()).isEqualTo(400);
  }

  @Test
  public void validate_shouldReturnLeapYearInfo() throws Exception {
    // given:
    MockMvcRequestSpecification request = given()
        .header("Content-Type", "application/json");

    // when:
    ResponseOptions response = given().spec(request)
        .queryParam("year","2020")
        .get("/leap-year");

    // then:
    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.header("Content-Type")).isEqualTo("application/json");

    // and:
    DocumentContext parsedJson = JsonPath.parse(response.getBody().asString());

    // and:
    assertThat((Object) parsedJson.read("$.['leap']")).isInstanceOf(java.lang.Boolean.class);
  }

}
```

Im Producer-Service haben wir nun sichergestellt, dass die angebotene Schnittstelle gemäß dem Contract funktioniert.
Ganz wichtig zu verstehen ist, dass der Test nicht sicherstellt, dass die Schnittstelle fachlich korrekt arbeitet, also dass 2020 tatsächlich ein Schaltjahr ist.
Es geht stattdessen darum, dass das Antwortformat sichergestellt ist.

Außerdem werden Stubs in Form einer Jar-Datei generiert.
Die Stubs können nun in Consumer-Services verwendet werden, in welchen wiederum gegen den Contract implementiert werden kann.

## Consumer Setup
Schauen wir uns daher die Perspektive im Consumer-Service an.
Der Consumer-Service soll nun die Schnittstelle des Producer-Service konsumieren und die generierten Stubs in Tests integrieren.

### Consumer-Schnittstelle
Wir implementieren nun zu Demonstrationszwecken eine GET-Schnittstelle, welche die Schnittstelle des Producer-Service konsumiert: 
```java
@RestController
public class YearCheckerController {

  private final Logger logger = LoggerFactory.getLogger(YearCheckerController.class);
  private WebClient webClient;

  @PostConstruct
  private void initWebClient() {
    webClient = WebClient.builder()
        .baseUrl("http://localhost:7000")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .build();
  }

  @GetMapping(value = "/check-leap-year", consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
  public ResponseEntity<LeapYearResponse> checkLeap(@RequestParam(value = "year", required = false) Integer year) {
    try {
      final LeapYearResponse leapYearResponse = webClient.get()
          .uri(uriBuilder -> {
            uriBuilder = uriBuilder.path("/leap-year");
            if (year != null) {
              uriBuilder = uriBuilder.queryParam("year", year);
            }
            return uriBuilder.build();
          })
          .retrieve()
          .bodyToMono(LeapYearResponse.class)
          .block();
      return ResponseEntity.ok(leapYearResponse);
    } catch (WebClientResponseException e) {
      logger.warn("Could not receive response while checking leap year", e);
      return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
    }
  }

}
```

Die Schnittstelle nimmt einen optionalen `year` Parameter entgegen und fragt damit den Producer-Service an.
Bei einer erfolgreichen Antwort wird diese an den anfragenden Client durchgestellt oder bei Fehlern mit einem `500 Internal Server Error` geantwortet.

### Gradle Setup
Im Gradle Setup des Consume-Service ergänzen wir das lokale Maven-Repository als Quelle:

```yaml
buildscript {
  repositories {
    mavenCentral()
    mavenLocal()
  }
}
repositories {
  mavenCentral()
  mavenLocal()
}
```

Außerdem benötigen wir noch die Abhängigkeiten zum [spring-cloud-contract-stub-runner](https://cloud.spring.io/spring-cloud-contract/2.1.x/multi/multi__spring_cloud_contract_stub_runner.html) und zu den Stubs des Producer-Service:

```yaml
dependencies {
  testImplementation 'org.springframework.cloud:spring-cloud-contract-stub-runner'
  testImplementation("com.adesso.contract:producer:+:stubs") {
    transitive = false
  }
}
```

### Test Setup
Nun wollen wir sicherstellen, dass der Producer-Service korrekt angebunden wurde.
Üblicherweise würde man nun für den Producer-Service händisch Mocks anlegen.
Stattdessen verwenden wir für die Mocks aber die generierten Stubs des Producer-Service.
Dazu bietet Spring Cloud Contract die Möglichkeit von sogenannten [Contract Stub Runnern](https://cloud.spring.io/spring-cloud-contract/2.1.x/multi/multi__spring_cloud_contract_stub_runner.html).
Diese vereinfachen die Integration von extern generierten Stubs in die Testumgebung.
Dies erfolgt, indem die benötigten Stubs automatisch zur Testlaufzeit heruntergeladen und ausgeführt werden.
Bei der Quelle der Stubs hat man unterschiedliche Auswahlmöglichkeiten:
* `CLASSPATH`: Die Stubs werden vom Java-Klassenpfad geladen.
* `LOCAL`: Die Stubs werden von dem lokalen Maven-Repository (.m2 Verzeichnis) geladen.
* `REMOTE`: Die Stubs werden von einer externen Quelle geladen (z.B. Artifactory)

Der Einfachheit halber verwenden wir das lokale Maven-Repository.
Die Testklasse sieht daher wie folgt aus:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
@AutoConfigureJsonTesters
@AutoConfigureStubRunner(
    stubsMode = StubRunnerProperties.StubsMode.LOCAL,
    ids = "com.adesso.contract:producer:+:stubs:7000")
class YearCheckerControllerTest {

  @Autowired
  private MockMvc mockMvc;

  @Test
  void given_WhenPassLeapYear_ThenReturnTrue() throws Exception {
    mockMvc.perform(MockMvcRequestBuilders.get("/check-leap-year")
        .queryParam("year", "2020")
        .contentType(MediaType.APPLICATION_JSON))
        .andExpect(status().isOk())
        .andExpect(content().string(containsString("true")));
  }

  @Test
  void given_WhenYearParameterMissing_ThenReturnInternalServerError() throws Exception {
    mockMvc.perform(MockMvcRequestBuilders.get("/check-leap-year")
        .contentType(MediaType.APPLICATION_JSON))
        .andExpect(status().isInternalServerError());
  }
}
```

Nun können wir die Consumer-Tests ausführen:
```console
gradle clean test
```

Wie man sieht, benötigen wir kein händisches Mocking des Producer-Service.
Im Build-Prozess werden automatisch die Producer-Stubs integriert.
Dadurch fällt nun jede Änderung aufseiten des Producer-Service bei Ausführung der Consumer-Tests auf.
Dies ist natürlich nur der Fall, sofern der Contract gebrochen wurde.
Das hier beschriebene Setup lässt sich als *Producer-Driven Contract Testing* beschreiben.
Denn die Contracts sind ausgehend vom Producer-Service definiert und Stubs angeboten worden.
Ebenso könnte man das Verfahren umkehren, sodass der Consumer-Service seine Anforderungen in Form von Contracts an den Producer-Service mitteilt.
Dies nennt man *Consumer-Driven Contract Testing*.
Die technische Umsetzung dazu wäre analog zu dem hier gezeigten.

## Fazit
Wir haben nach ein wenig Konfigurationsaufwand gesehen, dass das Contract-Driven Development mit Spring Cloud Contract recht unkompliziert möglich ist.
Contracts bieten den Vorteil im laufenden Entwicklungsprozess Breaking-Changes früher (oder überhaupt) erkennen zu können.
Spring Cloud Contract bietet die technische Möglichkeit sowohl den Ansatz des Consumer-Driven als auch des Producer-Driven Contract Testings umzusetzen.
Für welchen der beiden Wege man sich letztlich entscheidet, sollte im jeweiligen Projektkontext mit den beteiligten Parteien abgestimmt werden.

Das vollständige Beispielprojekt steht in [Github](https://github.com/g-stone7/spring-cloud-contract-showroom) zur Verfügung.
