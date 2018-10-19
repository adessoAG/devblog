---
layout:         [post, post-xml]              
title:          "Einführung in Kubernetes"
date:           2018-10-05 10:28
author:         t-buss
categories:     [Softwareentwicklung]
tags:           [cloud, kubernetes]
---
Die Container-Orchestrierungs-Lösung Kubernetes ist das wohlmöglich am stärksten gewachsene Open Source Projekt der letzten Jahre.
Container-Orchestrierung, das bedeutet das Management von hunderten lose gekoppelten Anwendungs-Container, die zusammen miteinander interagieren müssen.
Alle großen Cloud-Anbieter wie Google, Amazon, Microsoft und weitere bieten heutzutage Kubernetes-Instanzen an und unzählige Firmen lagern ihre Anwendungen auf Kubernetes-gestützten Clustern aus.
Grund genug, sich einmal näher mit Kubernetes und Konzepten dahinter zu beschäftigen.

# Einführung in Kubernetes
In diesem Blogpost geht es um die grundlegenden Konzepte, mit denen Kubernetes arbeitet.
Wir betrachten ein kleines Beispiel, indem wir eine triviale SpringBoot-Anwendung in einem lokalen Kubernetes-Cluster, Minikube, ausführen.
Zunächst klären wir aber einmal die Begrifflichkeiten.

# Cluster, Nodes und Pods
Kuberentes ist eine verteilte Anwendung, wird also auf mehreren physikalischen (oder virtuellen) Rechnern ausgeführt, die man als "Nodes" bezeichnet und zusammen den Kubernetes Cluster bilden.
Mindestens ein Node nimmt dabei die Rolle des Masters ein, der den Cluster managed und die Befehle des Benutzers entgegen nimmt.

Auf den Nodes laufen sogenannte "Pods".
Sie sind die kleinste Ressource in Kubernetes.
Ein Pod führt einen oder mehrere Container aus, die zusammen als Einheit gestartet werden müssen.
Die Wörtchen "oder mehrere" können dabei leicht zu Verwirrung führen.
In einer Microservices-Anwendung korrespondiert ein Pod zu einem Microservice, also meistens auch einem Container.
Es gibt verschiedene Fälle, in denen man gleich mehrere Container in einem Pod starten möchte.
Dies soll jedoch nicht Inhalt dieses Blogposts sein.
Da die Nodes miteinander ein virtuelles Netzwerk bilden und verbunden sind, kann es uns grundsätzlich egal sein, auf welchem Node ein Pod läuft.
Kubernetes verteilt automatisch die Pods auf solche Nodes, die im Moment weniger Last als die anderen haben.

Pods sind kurzlebig.
Sie werden erstellt, bekommen eine interne IP im virtuellen Netzwerk und führen ihre Container aus.
Wenn ein Pod beendet wird oder abstürzt, werden lokale Daten und Speicher gelöscht.
Die IP des Pods kann nun von beliebigen anderen Pods, die der Cluster startet, verwendet werden.
Es stellt sich also die Frage, wie man eine Anwendung erreichen kann, wenn die interne IP nicht als fest angenommen werden kann.
Zudem ist es für Anwendungen mit hoher Last nicht möglich, alle Anfragen von nur einer Container-Instanz abwickeln zu lassen.
Zur Skalierung müssen mehrere identische Pods gestartet werden, unter denen sich die Last aufteilt.
Diese Probleme nennt man "Service Discovery" und "Load Balancing" und sind in vielen verteilten Anwendungen präsent.

# Zielsetzung
Konkretisieren wir unser Ziel noch einmal mit den neuen Begriffen, die wir gerade kennen gelernt haben.
Das Ziel in diesem Blogpost soll es sein, einen REST-Service in einem Kubernetes Cluster bereitzustellen.
Clients im selben Cluster können eine URI aufrufen und erhalten die erwartete Antwort.
Wir programmieren eine einfache Anwendung, der den Wert einer Konfigurationsvariablen ausgibt.
Diese packen wir in einige Pods, die in unserem Cluster laufen.
Es soll sichergestellt werden, dass bei einem Absturz eines Pods automatisch ein neuer Pod gestartet wird.
Zudem soll gewährleistet werden, dass die Last auf alle aktiven Pods verteilt wird, sodass der Ausfall eines Pods von außen quasi nicht zu erkennen ist.

# Die Tools
Schauen wir uns die Tools an, mit denen wir unser Beispiel durchführen werden.

## Minikube
Allen voran brauchen wir einen Cluster, auf dem wir unsere Beispielanwendung laufen lassen.
Für die lokale Entwicklung eignet sich [Minikube](https://kubernetes.io/docs/setup/minikube/) sehr gut.
Es stellt einen Cluster mit nur einem Node in einer virtuellen Maschine bereit.
Dieser unterscheidet sich von einem "echten" Kubernetes Cluster nur darin, dass auf dem einen Node sowohl die Pods als auch die Master-Prozesse zur Verwaltung des Clusters laufen.
Normalerweise sind die Master-Prozesse auf designierten Nodes.

*Achtung: Obwohl V-Sphere offiziell von Minikube als Virtualisierungs-Lösung unterstütz wird, hatte ich einige Probleme, es damit zu starten.
Mit VirtualBox habe ich wesentlich bessere Erfahrungen gemacht und möchte es daher jedem ans Herz legen.*

Nachdem ein lokaler Cluster nach den Anweisungen auf der Minikube-Website installiert und gestartet wurde, können wir uns schon ein wenig in unserem Cluster umsehen.
Dazu dient das Kommandozeilentool `kubectl`, dass bei der Installation von Minikube mit installiert wird.
Mit `kubectl get pods` können wir uns beispielsweise alle Pods anzeigen lassen, die gerade laufen.
Wer kein Freund von Kommandozeilentools ist, kann sich mit dem Kubernetes Dashboard weiterhelfen.
Dazu gibt man das Kommando `minikube dashboard` ein, woraufhin sich der Browser öffnet und das Dashboard anzeigt.
Hier lassen sich Informationen zu allen Kubernetes Ressourcen anzeigen, die aktuell auf dem Cluster ausgeführt werden.
Wir kennen bereits Nodes und Pods.
Auf einige anderen Arten von Ressourcen wird später noch eingegangen.

## Kotlin und Spring
Als Programmiersprache für unsere Beispielanwendung werden wir Kotlin verwenden.
Kotlin ist eine moderne Programmiersprache für die JVM und alle coolen Kinder benutzen sie, also machen wir das auch!
Spring ist ein sehr beliebtes Framework für Webanwendungen auf Basis der JVM und eignet sich perfekt für unsere Zwecke:
Wir benötigen einen einfachen REST-Endpunkt und müssen eine Umgebungsvariable auslesen.
Beides lässt sich mit Spring relativ leicht bewerkstelligen.

## Docker
Kubernetes verwaltet Container und die beliebteste Software für Container ist Docker.
Docker an sich ist bereits ein riesiges Themengebiet, weshalb wir an dieser Stelle nicht weiter darauf eingehen können.
Soviel sei gesagt: Wir erstellen ein Docker-Image (eine Art Container-Blaupause) für unsere Spring-Anwendung und laden sie in eine Registry hoch (ich verwende GitLabs Docker Registry).
Der Cluster wird bei der Erstellung von Pods dieses Image runterladen und für die Container verwenden.

# Let's Go!
Genug der Theorie, gehen wir ans Werk.

## Die Spring Anwendung
Die Anwendung ist denkbar simpel, sodass wir uns nicht lange mit Erklärungen aufhalten.
Die interessante Stelle im Quellcode ist die Folgende:

```kotlin
@RestController
class EnvironmentVariableController {

    @Value("\${SOME_ENV_VAR}")
    lateinit var envVar: String

    @GetMapping("/getenv")
    fun getEnvironmentVariable() = this.envVar
}
```
Das Repository mit dem gesamten Quellcode ist [hier](https://gitlab.com/tbuss/sample-sck) zu finden.
Normalerweise würde an dieser Stelle jetzt die Erstellung eines Dockerfiles kommen.
Da Gitlab allerdings schlau ist und erkennt, dass es sich um ein Gradle-Projekt handelt, kann es das Schreiben des Dockerfiles für uns übernehmen.
Die Anwendung wird automatisch gebaut und ein Docker-Image erstellt und in die Registry geladen.
Den Link zum aktuellen Image findet man [hier](https://gitlab.com/tbuss/sample-sck/container_registry).

## Pods manuell starten und prüfen
Wir können jetzt einen oder mehrere Pods in unserem Cluster manuell erstellen.
Für alle Kubernetes-Ressourcen benutzen wir deklarative YAML-Dateien.
Dies hat den Vorteil, dass wir unser Setup als Dateien abspeichern und in ein eigenes Git-Repository speichern können.
Kubernetes ließt den gewünschten Status aus den Dateien aus und kümmert sich für uns dafür, dass dieser Status aufrecht erhalten wird.
Einfach ausgedrückt: "Was will ich haben?" anstatt "Was muss passieren?".
Die YAML-Datei für einen einfachen Pod sieht folgendermaßen aus:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-sck
spec:
  containers:
  - name: sample-sck
    image: registry.gitlab.com/tbuss/sample-sck/master:778763dd78540773aff9bc21fc3967e6ca3a0cbc
    ports:
    - containerPort: 5000
    env:
      - name: SOME_ENV_VAR
        value: Foo
```
Die YAML-Dateien in Kubernetes starten immer mit Meta-Informationen über die API, die benutzt wird und die Art von Ressource, die erstellt werden soll.
Auch ein Name wird angegeben.
Danach folgt die Spezifizierung des Pods, wo wir nicht nur das Image und den Namen angeben, sondern auch den Port der Anwendung (den müssen wir vorher wissen!) und die Umgebungsvariable, die wir nachher ausgegeben haben möchten.

Um den spezifizierten Pod zu erstellen, muss man die YAML-Datei unter `sample-sck.yaml` abspeichern und den Befehl

> `kubectl create -f sample-sck.yaml` 

ausführen.
Alternativ kann man auch im Dashboard rechts oben auf "Create" klicken und die Datei dort hochladen.
Mit `kubectl get pods` oder dem Dashboard kann man sehen, dass der Pod ausgeführt wird.

Der Pod läuft also, aber wie lässt sich erkennen, dass alles wie erwartet funktioniert?
Die IP des Pods ist schließlich eine interne IP des Clusters, worauf man von außen keinen Zugriff hat.
Dazu kann man den Befehl

> `kubectl port-forward pod/sample-sck 8080:5000` 

verwenden.
Dadurch werden Requests an `localhost:8080` weitergeleitet an den Port 5000 des angegebenen Pods.
Wenn man also http://localhost:8080/getenv im Browser öffnet, sollte das Wort "Foo" angezeigt werden, den Wert der Umgebungsvariable, die wir in der Definition des Pods angebenen haben.
Abbildung 1 zeigt den einfachen Aufbau:

![Clients wenden ich direkt an Pod](/assets/images/posts/intro-zu-kubernetes/k8s-0.png "Abbildung 1")

## Services
Wir können noch einige Pods auf diese Weise erstellen, wobei wir jedes mal den Namen des Pods ändern müssen, was sehr umständlich ist (eine Lösung dazu gibt es später).
Die Pods haben immer noch unterschiedliche IPs.
Daher kann ein Client unserer Anwendung nicht zu einem zentralen Punkt im Cluster navigieren und dort die Anwendung aufrufen.
Wir benötigen also einen Mechanismus zur Service Discovery.
Dafür gibt es in Kubernetes sogenannte *Services*.
Ein Service ist nichts anderes als ein Fixpunkt im Cluster, der die Anfragen an die damit verknüpften Pods weiterleitet.
Dabei achtet ein Service auf alle Pods, die ein bestimmtes *Label* haben, und leitet die Requests an einen dieser Pods weiter.
Labels sind ein mächtiges Werkzeug in Kubernetes.
Diese Key-Value-Paare können an alle Arten von Ressourcen angehängt werden und bieten eine flexible Möglichhkeit zur Gruppierung von Ressourcen, inklusive Pods.
Wenn wir die Pod-Definition von oben um Labels erweitern, können wir alle Pods als Gruppe identifizieren:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-sck1
  labels:
    app: sample-sck
    version: v1
spec:
  containers:
  - name: sample-sck
    image: registry.gitlab.com/tbuss/sample-sck/master:778763dd78540773aff9bc21fc3967e6ca3a0cbc
    ports:
    - containerPort: 5000
    env:
      - name: SOME_ENV_VAR
        value: Foo
```
Mit diesem Wissen lässt sich leicht ein Service definieren:
```yaml
apiVersion: v1
kind: Service
metadata:
  name:  sample-sck-service
spec:
  selector:
    app:  sample-sck
  type: NodePort
  ports:
  - port:  8080
    targetPort:  5000
```
Die Selector-Direktive beschreibt die Labels, die die Pods haben müssen, um von diesem Service erfasst zu werden.
Wir geben bewusst nur das `app`-Label an, damit unser Service zu allen Versionen der Anwendung weiterleitet (dazu später mehr).
Der Typ `NodePort` zeigt an, das Kubernetes für diesen Service auf jedem Node (bei Minikube nur das eine) einen Port öffnen soll, über den man den Service ansprechen kann.
In einem "richtigen" Kubernetes-Cluster hätten wir auch noch andere Möglichkeiten, den Service öffentlich zugänglich zu machen.
Port und targetPort zeigen an, das der Service auf Port 8080 läuft und auf die Ports 5000 der Pods weiterleitet.
Diese Grafik zeigt den momentanen Aufbau:

![Service leitet an Pods weiter](/assets/images/posts/intro-zu-kubernetes/k8s-1.png)

Speichern wir die Datei unter `sample-sck-service.yaml` ab und erstellen den Service mit:

> `kubectl create -f sample-sck-service.yaml`

Wir können die Funktion des Services leider nicht auf die selbe Weise testen, wie die Funktion eines Pods.
Es lässt sich zwar ein Port-Forwarding auf einen Service einrichten, jedoch wird dabei implizit ein einzelner Pod ausgewählt, an den "geforwarded" wird.
Sollte dieser Pod ausfallen, wird nicht automatisch an einen anderen Pod weitergeleitet und der Vorteil unseres Services ist dahin (ja, ich habe lange gebraucht, um das rauszufinden).
Glücklicherweise können wir über Minikube schnell an die URL kommen, über den wir den Service erreichen:

> `$ minikube service sample-sck --url`<br>
> `http://192.168.99.100:31862`

Unter dieser Adresse sollte jetzt wieder das Wort "Foo" zu sehen sein.

Beenden wir einmal alle Pods, die wir gerade gestartet haben:
> `kubectl get pods -l app=sample-sck -o json | kubectl delete -f -`

Nun definieren wir zwei Pods, jeweils mit den Werten `Foo` und `Bar ` für `SOME_ENV_VAR`, in den zwei unterschiedlichen Dateien: `sample-sck1.yaml` und `sample-sck2.yaml`.
Anschließend starten wir die Pods:

> `kubectl create -f sample-sck1.yaml -f sample-sck2.yaml`

Wenn wir nun ein paar mal die URL des Service aufrufen, wird manchmal der eine, manchmal der andere Wert angezeigt.
Wir können auch beobachten, was passiert, wenn ein Pod entfernt wird.
Der Service leitet die Anfragen an den verbleibenden Service weiter, ohne, dass zwischenzeitlich ein Ausfall zu vermerken ist.
Unser Service funktioniert also.

## Deployments
Bisher haben wir Pods immer nur manuell erstellt.
Das dies auf Dauer zu mühselig wird, kann man sich denken.
Wir können auf diese Weise nicht automatisch Pods starten und müssen ständig den Namen ändern.
Um diese Probleme zu lösen gibt es *Deployments*.
Mit Deployments geben wir einerseits eine "Schablone" für unsere Pods an (wie bei der manuellen Definition von Pods) und andererseits die gewünschte Anzahl der Pods.

Hier ist die Definition eines Deployments:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sample-sck-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: sample-sck
        version: v1
    spec:
      containers:
      - name: sample-sck
        image: registry.gitlab.com/tbuss/sample-sck/master:778763dd78540773aff9bc21fc3967e6ca3a0cbc
        ports:
          - containerPort: 5000
```
Wir beschreiben die Anzahl mit `replicas`; hier sind es zwei.
Der Rest der Definition sollte uns schon bekannt vorkommen.

Speichern wir diese Datei unter `sample-sck-deployment.yaml` ab und erstellen das Deployment:

> `kubectl create -f sample-sck-deployment.yaml`

Im Dashboard unter "Pods" kann man sehen, dass die gewünschten Pods automatisch erstellt wurden.
Der Name der jeweiligen Pods ergibt sich aus dem Namen, der im Deployment im Template angegeben wurde, einem Hash für das Deployment und einem Hash für den Container selbst.

# Update ausführen

Wenn wir im Dashboard auf "Workload" gehen, dann sehen wir die Ressourcen, die durch das Deployment erstellt wurden.
Darunter sind nicht nur das Deployment, sondern auch die Pods und ein sogenanntes *ReplicaSet*.
ReplicaSets werden intern von Deployments genutzt, um die gewünschte Anzahl der Pods zu einem Deployment sicherzustellen.
Dieses Konzept ist von Bedeutung, wenn es um das Updaten von einem Deployment geht.

## Szenario
Nehmen wir an, wir haben ein neue Version unserer Anwendung entwickelt.
Das dazugehörige Docker-Image haben wir bereits in eine Registry hochgeladen.
Nun wollen wir die Pods in unserem Cluster aktualisieren, und zwar ohne zwischenzeitlich Offline zu sein.

## Update durch Kommandozeile
Wir haben zwei Möglichkeiten, dieses Szenario durchzuführen:
Wir können das Image direkt mit einem Befehl von der Kommandozeile setzen oder die YAML-Datei ändern und die Änderungen anschließend anwenden.
Sehen wir uns zunächst das Updaten über die Kommandozeile an.
Kubectl bietet den Befehl `kubectl set` an, um Änderungen an bestehenden Ressourcen anzuwenden, auch Images.
Bevor wir den Befehl eingeben, können wir mit
> `watch kubectl get replicasets`

beobachten, wie das Update durchgeführt wird (auf Windows gibt es das Programm `watch` leider nicht, dann einfach nur oft hintereinander `kubectl get replicasets` eingeben).
Es sollte nur ein ReplicaSet für unser Deployment angezeigt werden.

Nun führen wir in einem anderen Terminal den Befehl aus:
> `kubectl set image deployment sample-sck-deployment sample-sck=registry.gitlab.com/tbuss/sample-sck/master:2af15466e456f7112b8b1b557d75be4dbab78df3`

Wir geben dabei die Aktion und das Deployment an und spezifizieren für den Container `sample-sck` das neue Image.

Jetzt können wir im ersten Terminal das Update-Verfahren beobachten:
Ein zweites ReplicaSet für das Deployment wird erstellt.
Die Spalten DESIRED, CURRENT und READY geben die Anzahl der Pods an, die von diesem ReplicaSet verwaltet werden.
Nach und nach werden neue Pods durch das zweite ReplicaSet gestartet.
Parallel dazu werden Pods aus dem alten ReplicaSet heruntergefahren.
Die Geschwindigkeit und Anzahl der Pods innerhalb dieses Vorgangs kann durch Parameter innerhalb der Deployment-Konfigurationsdatei angepasst werden, aber wir begnügen uns in diesem Falle mit den default-Werten.
Irgendwann sind alle Pods des alten ReplicaSets gelöscht und unser Update war erfolgreich.

Mit `kubectl rollout undo deployment sample-sck-deployment` kann man das Update wieder rückgängig machen.
Sehr praktisch, wenn man bemerkt, dass ein Fehler vorliegt und man auf einen alten Stand zurückkehren möchte.
Führen wir den Befehl einmal aus, damit wird im Folgenden auch das Update per Konfigurationsdatei durchführen können.

## Update durch Datei
Die zweite Möglichkeit, das Szenario durchzuführen, ist über die YAML-Konfigurationsdateien.
Dazu bearbeiten wir die Datei `sample-sck-deployment.yaml` und tragen das neue Image im `template` ein:
```yaml
    ...

      containers:
      - name: sample-sck
        image: registry.gitlab.com/tbuss/sample-sck/master:2af15466e456f7112b8b1b557d75be4dbab78df3

    ...
```
Wieder können wir mit `watch kubectl get replicasets` den Fortschritt des Updates verfolgen, während es ausgeführt wird.
Geben wir jetzt in einem anderen Terminal den Befehl zum Update:
> `kubectl apply -f sample-sck-deployment.yaml`

Genau wie bei dem Update per Kommandozeile wird ein zweites ReplicaSet erstellt und übernimmt nach und nach die Last des Ursprünglichen.
Auch hier hat also das Update geklappt.

```yaml
Notizen:
  Mehr Tutorial-Artig (z.B. Git Repo erstellen, Ordnerstruktur, Dateinamen etc.)
  Dateien anhand der Best-Practices ausrichten https://kubernetes.io/docs/concepts/configuration/overview/

########## TEIL 2 (?) ##########
ConfigMaps:
	- Umgebungsvariablen in Pods injizieren
```