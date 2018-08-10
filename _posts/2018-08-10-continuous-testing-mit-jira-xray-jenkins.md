---
layout:         [post, post-xml]              
title:          "Continuous Testing mit JIRA, XRay und Jenkins"
date:           2018-08-10 09:17
modified_date: 
author:         mhoeber
categories:     [Methodik]
tags:           [JIRA, Testautomatisierug, DevOps, Jenkins]
---
In der agilen Softwareentwicklung werden häufig Jenkins für Build & Deployment sowie JIRA für die Entwicklungs- und Testaufgaben verwendet. Sofern ein Testmanagement-Werkzeug verwendet wird, ist es selten JIRA. Automatisierte Testfälle werden separat gepflegt und berichtet. Von einer geschlossenen Werkzeugkette für Continuous Testing kann nicht die Rede sein. Die Grundkonfiguration ist jedoch einfacher als gedacht...

# Testmanagement mit JIRA & XRay
Das [JIRA-Plugin XRay](https://www.xpand-addons.com/xray/) vom Anbieter xpand-addons erweitert JIRA um ein umfassende Testmanagement-Funktionen, beispielsweise durch eine Testfallbibliothek, Visualisierung der Anforderungsabdeckung und eine bi-direktionale Nachverfolgbarkeit.
![JIRA/Xray Sprint-Board](/assets/images/posts/jenkins-xray/jira-xray-sprint-board-small.jpg)
![JIRA/Xray Nachverfolgbarkeit](/assets/images/posts/jenkins-xray/jira-xray-traceability-small.jpg)
XRay integriert sehr gut in bestehende JIRA-Projekte und lässt sich einfach und umfassend an die Projektbedürfnisse anpassen. Unter anderem kann XRay ein eigenständiges Testprojekt verwenden, das mit dazugehörenden Entwicklungsprojekten verknüpft wird, oder bestehende Entwicklungsprojekte maßgeschneidert ergänzen.
Die neuen Entitäten, darunter Testfälle, Testpläne und Testausführungen, und JQL, die XRay mitbringt ermöglichen eine zielgerichtete und transparente Steterung der erforderlichen Testaktivitäten.

## generische Testfälle für die Testautomatisierung
Testfälle werden nach manuellen, cucumber und generisch unterschieden. Generisch wird für automatisierte Testfälle verwendet, die von einem CI-Server regelmäßig ausgeführt werden

![Xray Testfall](/assets/images/posts/jenkins-xray/xray-testcase.jpg)

#Verbindung zwischen Jenkins & XRay herstellen
## Jenkins Xray-Plugin einrichten
[Xray-Plugin für Jenkins](https://confluence.xpand-addons.com/display/XRAY/Integration+with+Jenkins)
![Jenkins Xray Konfiguration](/assets/images/posts/jenkins-xray/jenkins-plugin-config.jpg)

# Jenkins-Slave für Testausführung anlegen
## Node konfigurieren

![Jenkins Node](/assets/images/posts/jenkins-xray/jenkins-node.jpg)
## JNLP-Verbindung herstellen

![Jenkins JNLP-Slave](/assets/images/posts/jenkins-xray/jenkins-jnlp.jpg)

# Testausführung über Jenkins
## Anlage des Jenkins-Jobs
![Jenkins Job](/assets/images/posts/jenkins-xray/jenkins-job-config.jpg)

## Ausführen des Jenkins-Jobs
![Jenkins Testausfuehrung](/assets/images/posts/jenkins-xray/jenkins-job-run.jpg)

## Testergebnis in JIRA/Xray
![JIRA/XRay Testausfuehrung](/assets/images/posts/jenkins-xray/jira-testrun.jpg)
![JIRA/Xray Testfall](/assets/images/posts/jenkins-xray/jira-testcase.jpg)
