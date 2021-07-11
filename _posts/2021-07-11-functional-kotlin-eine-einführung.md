---
layout: [post, post-xml] # Pflichtfeld. Nicht ändern!
title: 'Functional Kotlin - Eine Einführung' # Pflichtfeld. Bitte einen Titel für den Blog Post angeben.
date: 2021-07-11 13:00 # Pflichtfeld. Format "YYYY-MM-DD HH:MM". Muss für Veröffentlichung in der Vergangenheit liegen. (Für Preview egal)
modified_date: 2021-07-11 # Optional. Muss angegeben werden, wenn eine bestehende Datei geändert wird.
author: fabianvolkert # Pflichtfeld. Es muss in der "authors.yml" einen Eintrag mit diesem Namen geben.
categories: [Softwareentwicklung] # Pflichtfeld. Maximal eine der angegebenen Kategorien verwenden.
tags: [
    Kotlin,
    Funktionale Programmierung,
    Property Based Testing,
    Programmierparadigmen,
    Immutabilität,
    Higher-Order-Functions,
    Funktionen Höherer Ordnung,
    Threadsicherheit,
    Nullsicherheit,
    Extension Functions
  ] # Bitte auf Großschreibung achten.
---

# Functional Kotlin -- Eine Einführung
In diesem Blogeintrag widmen wir uns der Programmiersprache Kotlin.
Wir werfen einen kurzen Blick auf die Ursprünge der Sprache, wie sie aufgebaut ist und mit welchen Designprinzipien im Hinterkopf sie entworfen wurde. 
An Beispielen betrachten wir die Best Practices und stellen uns dabei die Frage: Was hat das mit funktionaler Programmierung zu tun?

# Kotlin

Vor zehn Jahren (2011) stellte JetBrains erstmals die Open-Source Programmiersprache Kotlin auf dem [JVM Language Summit](https://blog.jetbrains.com/kotlin/2011/07/hello-world-2/) vor -- "Eine Sprache einfach genug für den gewöhnlichen Entwickler und produktiv genug für moderne Anforderungen an Projekte". 
Die Sprache setzt auf der Java Virtual Machine (JVM) auf und erschien 2016 in der ersten Release-Version. 
2017 wurde sie von Google offiziell zur Entwicklung von Android-Apps unterstützt und ist seit 2019 Googles bevorzugte Sprache für diese Plattform.

Im [TIOBE-Index](https://www.tiobe.com/tiobe-index/)[^1] rangiert Kotlin aktuell (Juli 2021) auf Platz 38 der beliebtesten Programmiersprachen. 
Betrachtet man ausschließlich die JVM-spezifischen Sprachen steht Kotlin dort auf Platz 4 (übertrumpft von #36 Scala, #15 Groovy und #2 Java). 

Im [PYPL-Index](https://pypl.github.io/PYPL.html)[^2] belegt Kotlin den 11. Platz.

[^1]:TIOBE zählt die gefundenen Ergebnisse zu Programmiersprachen aus verschiedenen Suchmaschinen.
[^2]:PYPL nutzt Google Trends für Tutorial-Suchanfragen.

Kotlin, eine statisch typisierte Programmiersprache, ist voll interoperabel zu Java-Programmen und -Bibliotheken und kann ohne aufwändige Integration in bereits bestehende Projekte eingepflegt werden.

## Grundlagen
Jeder Einstieg in eine Programmiersprache fängt gleich an. 
Man muss sich an die Syntax gewöhnen. 
Folgendes Beispiel zeigt ein kleines in Kotlin geschriebenes Programm, welches zuerst das Ergebnis einer Instanzmethode und dann das einer statischen Methode ausgibt:
```kotlin
fun main(){
    val mainClassInstance = MainClass("instanceString")
    println(mainClassInstance.instanceMethod())
    println(MainClass.staticMethod(3))
}

class MainClass(private var member: String){
    companion object{
        fun staticMethod(parameter:Int):Int{
            return parameter*2
        }
    }
    
    public fun instanceMethod():String{
        return "A"
    }
}
```

Das [`companion object`](https://kotlinlang.org/docs/object-declarations.html#companion-objects) ist, wie der Name es andeutet, ein Begleiterobjekt zu dieser Klasse und verhält sich ähnlich der statischen Initialisierung in Java (auch wenn das Begleiterobjekt noch einiges mehr kann, auf das ich hier nicht eingehen werde).
Anders als in Java werden die Rückgabewerte von Methoden am Ende des Methodenkopfs plaziert und auch für Wertdefinitionen und Parameter wird der jeweilige Typ durch einen ":" getrennt auf die rechte Seite gestellt. 
Kotlin unterstützt [Typinferenz](https://kotlinlang.org/spec/type-inference.html), weswegen die Typdefinitionen in den meisten Fällen auch weggelassen werden können.
Was -- anders als bei Java -- hier auch auffällt, ist, dass ich die *Properties* der `MainClass` direkt hinter den Klassennamen in "( )" definieren kann und sie nicht im Codeblock schreiben muss (, aber auch das könnte ich). 
Kotlin generiert für die Variablen Getter- und Setter-Methoden und für die Values nur Getter-Methoden.
Auf den Unterschied komme ich im Abschnitt [Immutabilität](#Immutabilitt_80) zu sprechen.
Wie das Beispiel oben auch zeigt, habe ich einen primären Konstruktor für die `MainClass`geschrieben, der sich direkt im Header befindet. 
Die in den Klammern des Konstruktors angegebenen Properties (hier *member*) entsprechen direkt einer Deklaration dieser als Teil der Klasse.
Auch die Semikolons können wir in den meisten Fällen weglassen.

## Nullsicherheit
Eine Ärgerlichkeit, mit der wir uns als Entwicklerinnen und Entwickler im Alltag häufig auseinandersetzen müssen ist das Behandeln von Nullpointer-Exceptions; 
Also das Fehlen von Daten an Stellen, an denen das Programm welche erwartet hat. 
([My billion-dollar mistake](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/))
Kotlin behandelt dieses Problem aus meiner Sicht pragmatisch ([Null-Safety](https://kotlinlang.org/docs/null-safety.html)), indem, wenn nicht anders angegeben, Werte einfach nicht null sein dürfen.
Betrachten wir folgendes kleines Beispiel eines Produkts, für das ein Preis mit Steuer berechnet werden soll.
```kotlin
fun main() {
    val product = Product(4.99)
    val vat: Double = null
    product.getConsumerPrice(vat)
}

class Product(private val price: Double) {
    fun getConsumerPrice(vat: Double): Double {
        val tempValue = helperFunction(vat)
        return tempValue * this.price
    }
    ...
}
```
Was tendenziell in Java und vielen anderen Sprachen funktioniert wird hier vom Compiler mit einer Fehlermeldung quittiert, da der Wert `val vat: Double` als Double mit Wert definiert und null dort nicht erlaubt ist.
So sind wir als Entwickler immer informiert, wann Daten potentiell undefinierte Zustände annehmen würden:
`"Kotlin: Null can not be a value of a non-null type Double"`
Manchmal lässt es sich allerdings auch nicht vermeiden, oder ist erwünscht, dass ein null Wert übernommen wird;
Zum Beispiel an Schnittstellen, wenn kein Default-Wert Sinn ergibt.
Dazu kann die Entwicklerin oder der Entwickler Werte mit einem `?` markieren, dass sie null (*nullable*) sein dürfen: 
`val vat: Double? = null`
Hier kommt die Arbeit zum Vorschein, die uns der Compiler durch diese kleine Änderung abnimmt: `product.getConsumerPrice(vat)` wird mit dem `Double?`aufgerufen, aber `getConsumerPrice(vat: Double)` erwartet einen Wert, der nicht null ist.
Auch das erkennt der Compiler und gibt `Type mismatch: inferred type is Double? but Double was expected`zurück.
So sind wir gezwungen uns um diesen Fall zu kümmern und entweder vorher sicherzustellen, dass `vat` nicht null sein kann, oder einen null-Wert als Eingabeparameter zu erlauben, wodurch sich die Fehlermeldung auf nachfolgende Aufrufe von `vat` verbreitet.

Ein anderes Beispiel zeigt, wie wir uns beim Programmieren mit Nullwerten in Kotlin viel Boilerplate-Code sparen können:
```java
class Head {
    public Node next;
}
class Node {
    public Node next;
    String value = null;
}
...
head.next.next.value;
```
Die Suche von `value` in dem Beispiel kann, wenn einer der Zwischenaufrufe null ist, zu einer `NullpointerException` führen.
Um dieses Problem zu umgehen, müssen wir zwischen den Aufrufen null-Checks einführen. 
Um Platz zu sparen schreibe ich die Prüfungen direkt als ternäre Operationen:
```java
Head head = new Head();
Node nodeA = head.next != null? head.next :null;
Node nodeB = nodeA.next != null? nodeA.next :null;
String value = nodeB != null? nodeB.value: null;
```
In Kotlin kann bei potentiellen Nullwerten `?` eingesetzt werden, um diese zu erlauben:
```java
class Head(val next: Node?)
class Node(val next: Node?, val value: String)
...
val head = Head(null) // Bei der Initialisierung muss ich den Wert für 'next' direkt angeben und kann ihn nicht unbestimmt lassen
val string = head.next?.next?.value
```
Dieser Aufruf führt zu keiner `NullpointerException`, sondern weist `string` null zu, da bereits der Aufruf von `head.next?` null zurückgibt. 
Der Wert ist dabei implizit vom Typ `String?`, wodurch auch alle folgenden Aufrufe vom Compiler wieder geprüft werden.

Alternativ kann der Elvis-Opeator `?:` genutzt werden, um in solchen Fällen direkt einen Standardwert zuzuweisen, sodass statt `String?` der Typ `String` inferiert wird.
```Kotlin
val string = head.next?.next?.value?:"default"
```

So kann sichergestellt werden, dass null-Werte innerhalb der Anwendung angemessen behandelt werden können.

## Immutabilität
In den beiden Beispielen der vorherigenen Sektionen habe ich das `val`- und das `var`-Schlüsselwort zur Definition von Werten genutzt. 
`val` wird genutzt um einen zur Laufzeit unveränderlichen Wert zu definieren (anders noch als `const`, welches für unveränderliche, zur Kompilierzeit bekannte, Werte steht). 
Es ist vergleichbar mit `final` aus Java.
Auf der anderen Seite steht das `var`-Schlüsselwort mit dem herkömmliche Variablen beschrieben werden können.
Immutabilität hilft Entwicklerinnen und Entwicklern Nebeneffekte im Code auf ein Minimum zu reduzieren und schafft so Sicherheit vor allem für Parallelität.
Betrachten wir folgendes Beispiel in Java:
```Java
class SideEffect {
    public int member = 0;

    public int someCalculation(int input) {
        int aux = member + 2;
        int result = member + aux + input;
        member++;
        return result;
    }
}
```
`someCalculation` nutzt die Variable `member` für einige Berechnungen. 
In einer synchronen Umgebung ist dies problemlos möglich. 
Soll die Methode allerdings parallel ausgeführt werden, kann es zu inkonsistentem Verhalten kommen, da `member` zu verschiedenen Zeitpunkten innerhalb der Ausführung der Methode unterschiedliche Werte annehmen kann.
Besser ist hier eine Lösung, die `someCalculation` weitestgehend unabhängig vom aktuellen Wert von `member`macht. 
Denkbar ist:
```Java
public int someCalculation(int input, int memberVal) {
    int aux = memberVal + 2;
    int result = memberVal + aux + input;
    member++;
    return result;
}
```
Durch das Verlagern des für die Berechnung genutzten Wertes ist sichergestellt, dass die Methode, selbst wenn sich `member` zur Laufzeit ändert, innerhalb ihres Ausführungskontextes einen konsistenten Zustand einhält.
An dieser Stelle bediene ich mich zusätzlich an einigen Punkten, auf die man zum Thema Immutabilität im Internet immer wieder trifft:
- Threadsicherheit (Durch Zugriff auf Werte, die sich nicht ändern)
- Keine versteckten Nebeneffekte (Es gibt kein Risiko, dass Methoden unbemerkt Werte an anderen Stellen ändern)
- Sicherheit vor Nullwerten (Wenn ein Wert einmal überprüft wurde, behält er seine Gültigkeit)
- Leichteres Chaching (Wenn ein Wert einmal geladen wurde und sich Rahmenbedingungen ändern, ist sichergestellt, dass dieser Wert nach wie vor gültig ist und nicht neu geladen werden muss)
- Bessere Kapselung von Methoden und Klassen (Es ist sichergestellt, dass Methoden und Klassen, die untereinander kommunizieren, sich nicht gegenseitig verändern)
- Einfacher zu Testen (Durch feste Werte und fehlende Nebeneffekte sind die Punkte, die es bei Fehlern zu überprüfen gilt weniger und einfacher)
- Leichtere Lesbarkeit und Wartbarkeit (Geht einher mit der leichteren Testbarkeit)
- Vorhersagbarkeit (Wenn Werte konkret sind, können zuverlässige Annahmen getroffen werden)

Ich denke diese kurze Übersicht ergibt Sinn und spricht für sich selbst, dass der Aufwand für Immutabilität im Verältnis zu den Vorteilen in den meisten Fällen gering ausfallen dürfte.

# Funktionale Programmierung
An dieser Stelle macht es Sinn, die funktionalen Programmierung ins Spiel zu bringen. 
Was ist funktionale Programmierung und wie kann sie uns bei unserer Arbeit helfen?
Die funktionale Programmierung ist ein Ansatz der Programmierung, die Verarbeitung von Daten nicht anweisungsgetrieben (imperativ, wie z.B. in Java) zu konzipieren, sondern aus einer mathematischen Perspektive heraus -- funktional -- zu betrachten.
Also statt das wir ein Problem aus der Perspektive betrachten, jeden Schritt einzeln durchzugehen, arbeiten wir mit einer Menge von Daten, auf die Operationen angewandt werden und die mitunter eine neue Menge von Daten erzeugt.
Die funktionale Programmierung ist etwas, was in vielen großen Programmiersprachen immer mehr Einzug hält, auch weil die Rechenleistung heutiger Computer so hoch ist, dass die schlechtere Performance, die durch diesen Ansatz erreicht wird, nicht mehr ins Gewicht fällt.
Theoretische Grundlage der funktionalen Programmierung ist das Lambda-Kalkül, welches in den 30er Jahren von Church und Kleene zur Beschreibung von Funktionen eingeführt wurde.
Ein einfacher Lambda-Ausdruck sieht dabei wie folgt aus und beschreibt hier f(x)=x+2:
```
λx.x+2
```
Lambda-Ausdrücke kennt man als Entwicklerin oder Entwickler im Java-Kontext hauptsächlich in Form von Lambda-Ausdrücken (ab Java 8 in 2014). 
Collections müssen dazu erst in einen *stream* konvertiert, transformiert und dann dann wieder zurück konvertiert werden:
```Java
strings
  .stream()
  .filter(s -> s.length() == 5)
  .collect(Collectors.toList());
```
Lambda-Ausdrücke gibt es auch in Kotlin und werden dort viel häufiger verwendet. 
Gibt es nur einen Wert im Lambda-Ausdruck der gebunden werden muss, kann der implizite Name *it* benutzt werden, statt dem Laufwert einen konkreten Namen geben zu müssen.
```Kotlin
strings.filter { it.length == 5 }
```
Auch wenn beide Beispiele hier nur einfache sind, empfinde ich persönlich die Kotlin-seitigen Lösungen häufig intuitiver und kürzer als das bei Java der Fall ist. 
Allein der Wegfall der Konvertierungen reduziert den Boilerplate-Code und erleichtert damit die Wartung der Software.

## map, reduce, filter, ...
Die `filter`-Methode haben wir gerade eben kennengelernt. 
Wie der Name beschreibt, kann sie genutzt werden, um Elemente aus einer Menge an Daten herauszufiltern.
Die zwei wichtigen anderen Methoden, die häufig eingesetzt werden sind die `map`- und die `reduce`-Methode.
`Map` iteriert über jedes Element einer Menge von Daten und wendet eine Funktion auf dieses an. 
Heraus kommt dabei eine neue Menge von Daten, die möglicherweise geändert wurden. 
(Ich sage möglicherweise, weil die identische Abbildung f(x)=x existiert)
```Kotlin
productList.map { product -> product.getConsumerPrice(0.19) }
```
Obiges Beispiel zeigt, wie eine Liste von Produkten in eine Liste von Preisen konvertiert wird, in dem von jedem Produkt-Element der Konsumentenpreis geholt wird.
Die `reduce`-Methode verhält sich ähnlich zur `map`-Methode, mit dem Unterschied, dass das Ergebnis ein einzelnes Element ist. 
Auch hier wird auf jedes Element der Menge eine Funktion angewandt.
Das folgende Beispiel zeigt, wie aus unserer Preisliste eine Summe über alle Preise gebildet wird. 
`sum` definiert dabei das Akkumulator-Element im ersten Parameter.
Im zweiten Parameter wird die Funktion mit `price` als Laufwert aus der Liste vorgegeben.
```Kotlin
priceList.reduce { sum, price -> sum+price }
```
Statt uns mit der Iteration beschäftigen zu müssen, erlaubt diese Heransgehensweise uns das eigentliche Problem behandeln zu können.
`Map`, `filter` und `reduce` sind Beispiele für sogenannte [Funktionen höherer Ordnung](https://kotlinlang.org/docs/lambdas.html), denn sie nehmen nicht nur einfache Werte als Parameter entgegen, sondern erwarten Funktionen, die sie während ihrer Ausführung ausrufen können.
Ihre Flexibilität im Kern, während sie einen klaren Rahmen für die Verarbeitung von Daten in einer bestimmten Art und Weise schaffen, machen sie zu mächtigen Werkzeugen.

## Extension Functions

```Kotlin
inline fun <S, T : S> Iterable<T>.reduce(
    operation: (acc: S, T) -> S
): S
```
*Definition der reduce Extensionfunktion mit Generics*

Zwei Dinge, die an dem obigen Beispiel auffallen sind der Einsatz von Generics zur Verallgemeinerung der Anwendbarkeit der Funktion;
Und das es sich hierbei um eine sogenannte [*Extension-Function*](https://kotlinlang.org/docs/extensions.html) handelt, die -- in diesem Fall -- Iterable um eine Methode erweitert.
Extension-Functions können genutzt werden um Klassen zu erweitern, ohne neue Klassen oder Interfaces definieren zu müssen, die von der Grundklasse erben.
(Mehr Informationen zu [inline-Funktionen](https://kotlinlang.org/docs/inline-functions.html))

Hier ein Beispiel aus einem Projekt in dem ich gearbeitet habe. 
Dort haben wir Extension-Functions häufig genutzt um vor allem die Lesbarkeit unseres Codes zu erhöhen.
```Kotlin
data class Partner(
        val name: String,
        ...
)

fun Partner.toDto(): PartnerDto = PartnerDto(
        name,
        ...
)
...
partner01.toDto()
```
Natürlich lässt sich die Konvertierung in ein DTO auch klassisch lösen: 
```Kotlin
fun createPartnerDto(partner:Partner):PartnerDto {
        return PartnerDto(
                partner.name,
                ...
        )
}

createPartnerDto(partner01)
```
Allerdings erhöht die erste Variante die Lesbarkeit des Codes, wenn es um [Methodenverkettung](https://en.wikipedia.org/wiki/Method_chaining) geht (wie oben zu sehen, gibt es dieses Konzept auch in Java). 

Klassische Funktionen:
```Java
prepareSend(enrichWithData(createFromPartnerDto(partner01), data), destination)
```
Extension-Functions:
```Kotlin
partner01.toDTO().enrichWithData(data).prepareSend(destination)
```
Ein anderes Beispiel zeigt, dass wir auch Klassen erweitern können, die wir nicht selber geschrieben haben.
(Das Schlüsselwort `suspend` kann ignoriert werden. 
Bei Interesse empfehle ich die Einführung in [Coroutines](https://kotlinlang.org/docs/coroutines-basics.html))
Hier haben wir String um eine domänenspezifische Funktion erweitert, um zu diesem eine zugehörige Klasse zu finden:
```Kotlin
private suspend fun String.getCategoryByCategoryId(): Category?
...
val category = item.category?.getCategoryByCategoryId()
```


# Funktionales Testen mit Property Based Testing

![Available automated test technics](/assets/images/posts/functional-kotlin-eine-einführung/available-automated-test-technics.png)

[Bildquelle](https://medium.com/criteo-engineering/introduction-to-property-based-testing-f5236229d237)

Nehmen wir das, was wir bis jetzt betrachtet haben, ergeben sich daraus auch neue Möglichkeiten Softwarequalität sicherzustellen. 
Die Standardvorgehensweise für das Schreiben von Tests auf unterster Ebene sind die Unit-Tests. 
Kleine Blöcke, die die Funktionalität einzelner, isolierter Code-Ausschnitte überprüfen sollen, indem wir feste Vorgabewerte definieren und an die jeweilige Funktion übergeben.
An dieser Stelle tritt das Property based testing aus der funktionalen Programmierung auf den Plan.
Dahinter verbirgt sich die Idee, statt einige feste Werte auf bestimmte Ergebnisse zu überprüfen (und damit Fehlerräume an den Stellen zu lassen, die man nicht testet), gemeinsame Eigenschaften in Gruppen von Eigabeparametern zu finden, die anschließend randomisiert überprüft werden können. 
Schauen wir uns für ein besseres Verständnis ein einfaches Beispiel an.
Die folgende Funktion konkateniert zwei Strings miteinander (Man beachte, dass die geschweiften Klammern weggelassen werden können, wenn es sich bei der Funktion um eine einzelne Operation handelt):
```kotlin
public fun concatenate(string1: String, string2: String): String = string1 + string2
```
In klassischer Herangehensweise würde man beim Testen neben den Grenzfällen (leerer String, Nullstring), einen "normalen" Methodenaufruf testen.
Eine andere Art und Weise an den Test heranszugehen, ist sich zu überlegen, welche Eigenschaft die Ergebnisse des Methodenaufrufs gemein haben.
Eigenschaften lassen sich dabei nach folgender Form beschreiben:\
*Für Werte ... gilt, wenn ... zutrifft, dass ... wahr/falsch ist*\
In diesem Fall können wir also sagen:\
*Für alle Strings string1 und string2 gilt, dass die Konkatenation von string1 und string2 mit string1 anfängt und mit string2 endet*\
In [Kotest](https://kotest.io/) könnte der Test dann so aussehen:
```kotlin
class StringConcatTest: StringSpec({
   "Alle konkatenierten Strings starten mit string1 und enden mit string2" {
      forAll<String, String> { string1, string2 -> {
         val concat = concatenate(string1, string2)
         return concat.startsWith(string1) && concat.endsWith(string2)
         }
      }
   }
})
```
Kotest würde mit seinem Standardgenerator diesen Test für 1000 Werte durchspielen und die Ergebnisse prüfen.
Selbstverständlich können auch eigene Generatoren geschrieben werden, die an die eigenen Bedürfnisse angepasst sind.
In diesem Beispiel mag der Test trivial sein, da auch die Fachlichkeit sehr simpel ist. 
Je komplexer allerdings die Methode, desto schwieriger kann es sein herauszufinden, welche Menge von Daten eigentlich welche Eigenschaften erfüllen soll.
Ein netter Nebeneffekt: 
Tests so zu schreiben zwingt uns damit nochmal auf eine andere Art und Weise über die Korrektheit einer Methode nachzudenken.

# Abschluss
Für mich persönlich, mit Java-Erfahrung seit 2012, war mein erster praktischer Kontakt mit Kotlin in 2019 wie eine kleine Offenbarung.
Möglicherweise ist es der Gewöhnungseffekt, dass Kotlin für mich in vielen Punkten durchdachter als Java scheint.
Sicherlich wird dabei auch das noch recht junge Alter der Sprache und die Erfahrungen, die in sie hineingeflossen sind, eine Rolle spielen, das dem wirklich so ist.
Das funktionale Paradigma und Kotlin in seiner Ausprägung, ich habe es schon vorher geschrieben, kommen mir häufig leichtgängiger vor und es bereitet mir viel Freude so zu programmieren.
Wir sollten uns dabei aber auch immer bewusst sein, dass dies zu einem gewissen Preis geschieht.
Der Preis, den wir hier zahlen, sind Effizienzeinbußen (Die in vielen Projekten allerdings vernachlässigbar sein werden), gegenüber zum Beispiel einer optimierten Programmierlösung in C, sowie der Aufwand, den es für jeden und jede von uns mit sich bringt, sich an diese *neue* Art zu denken zu gewöhnen.
Es hat einen Grund, dass viele Sprachen sind heutzutage an diesen Aspekten orientieren und diese bei den Entwicklerinnen und Entwicklern populär sind und ich hoffe, dass ich euch einen kleinen Einblick in Kotlin und die funktionale Welt geben konnte.

Wenn ihr an dieser Stelle noch neugierig seid und euch weiter mit der Sprache beschäftigen möchtet, dann empfehle ich ausdrücklich die [Kotlin Koans](https://kotlinlang.org/docs/koans.html).
Eine sehr gute offizielle Tutorial-Reihe die sich unter anderem mit den Inhalten beschäftigt, die wir hier nur oberflächlich betrachten konnten und auch noch viel weiter in die Details der Sprache eintaucht.

Im Allgemeinen empfehle ich auch den [Kotlin Playground](https://play.kotlinlang.org/) zum schnellen und unkomplizierten herumprobieren und programmieren im Webbrowser eurer Wahl, wenn ihr Kotlin nicht lokal ausführen wollt.

Ein anderer spannender Blogeintrag zum Thema Kotlin bei adesso, wenn ihr direkt weiterlesen wollt, was sonst noch so in der Kotlin Welt passiert:
[Kotlin Multiplattform Mobile](https://www.adesso.de/de/news/blog/vorteile-von-kotlin-fuer-die-businesslogik-von-android-und-ios-apps.jsp)
oder direkt an der Quelle:
[The Kotlin Blog](https://blog.jetbrains.com/kotlin/)


Quellen:  
- [Kotlin-Wikipedia](https://de.wikipedia.org/wiki/Kotlin_(Programmiersprache)) 
- [Funktionale Programmierung-Wikipedia](https://de.wikipedia.org/wiki/Funktionale_Programmierung)
- [Lambda Kalkül-Wikipedia](https://de.wikipedia.org/wiki/Lambda-Kalk%C3%BCl)
- [Kotlin Dokumentation](https://kotlinlang.org/docs/properties.html)
- [Learning Journal](https://www.learningjournal.guru/article/scala/functional-programming/immutability-in-functional-programming/)
- [Baeldung](https://www.baeldung.com/kotlin/const-var-and-val-keywords)
- [Philipp Hauer's Blog](https://phauer.com/2017/idiomatic-kotlin-best-practices/#functional-programming)
- [Medium](https://medium.com/criteo-engineering/introduction-to-property-based-testing-f5236229d237)
- [Kotest](https://kotest.io/docs/proptest/property-test-functions.html)
- [Leading Agile](https://www.leadingagile.com/2018/03/immutability-in-java/)
- [Hackernoon](https://hackernoon.com/5-benefits-of-immutable-objects-worth-considering-for-your-next-project-f98e7e85b6ac)
- [Stackoverflow](https://stackoverflow.com/questions/34385243/why-is-immutability-so-important-or-needed-in-javascript)