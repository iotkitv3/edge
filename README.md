Internet of Things – Machine Learning Fast Data Pipeline mit Docker/Kubernetes
==============================================================================

> [⇧ **Home**](https://github.com/iotkitv3/intro)

![](https://raw.githubusercontent.com/iotkitv3/intro/main/images/edge-ml.png)

- - - 

Bei dieser Fast Data Pipeline werden
* Sensordaten (Temperatur) von einen IoT Gerät erhoben und
* weitergerreicht (Publish) an einem [Raspberry Pi](https://www.raspberrypi.org/) (Edge) mit MQTT Broker
* der Low-Code Service [Node-RED](https://nodered.org/) holt diese Daten (Subscribe) 
    * wandelt diese nach HTTP (REST) für die [Prozess Workflow Engine](https://camunda.com/) (BPMN) und
    * reicht sie an ein Hochverfügbares Messaging System ([Kafka](https://kafka.apache.org/)) weiter für Verarbeitungen wie Maschine Learning.

Dabei kommen folgende Technologien/Produkte zum Einsatz:
* **Iot**: [IoTKitV3](https://github.com/iotkitv3) 
* **Edge**: [Raspberry Pi](https://www.raspberrypi.org/) mit MQTT Broker [mosquitto](https://projects.eclipse.org/projects/iot.mosquitto) betrieben in einem Container.
* **Cloud**: [lernMAAS](https://github.com/mc-b/lernmaas) oder eine lokale Umgebung z.B. die vom [ModTec Kurs](https://github.com/mc-b/modtec)
    * [Kubernetes](https://kubernetes.io/de/) als Container Umgebung
    * [Node-RED](https://nodered.org/) (Triage und Protokollwandler)
    * Hochverfügbarkeits Messaging mittels [Kafka](https://kafka.apache.org/)
    * die [Camunda Prozess (BPMN) Engine](https://camunda.com/) für Geschäftsprozesse Workflows
    * [Juypter Notebook](https://jupyter.org/) für die Verarbeitung der Daten mittels Machine Learning.
    
***    
Installation und Konfiguration
------------------------------

### Internet of Things (IoTKitV3)


* [MQTTPublish](https://github.com/iotkitv3/mqtt) Beispiel in mbed Studio importieren und ca. auf Zeile 21 den `hostname` mit der IP-Adresse auswechseln wo der Mosquitto Server läuft. 
* Programm Compilieren und auf Board laden.

Wenn alles eingerichtet ist, können die Sensordaten via Jupyter Notebooks verarbeitet werden.

Wird ein Magnet an den Hall Sensor gehalten, wird ein BPMN Prozess gestartet.

Je nach Board können auch weitere Daten wie
* das Drücken des Buttons
* das Drehen des Encoders
* das lesen einer NFC Karte 
ausgewertet werden.

Für Details siehe das [MQTTPublish](https://github.com/iotkitv3/mqtt/blob/main/main.cpp) Beispielprogramm.

***
### Edge (Raspberry Pi)

#### Raspberry Pi 2 oder 3 als Edge aufsetzen

Der Raspberry Pi (2, 3) eignet sich als Edge. 

Dazu ist der Raspbery Pi (2, 3) wie folgt aufzusetzen:

* Raspbian Lite wie auf [raspberrypi.org](https://www.raspberrypi.org/) beschrieben, downloaden und auf SD Karte speichern.
* Auf der SD Karte eine Datei «ssh» ohne Endung erstellen.
* Auf Windows Git/Bash oder Putty o.ä. installieren. Mac und Linux brauchen nur Git.
* Raspberry Pi via Kabel mit dem Router LERNKUBE verbinden und via ssh Verbinden
* [mosquitto](https://projects.eclipse.org/projects/iot.mosquitto) und [Node-RED](https://nodered.org/) manuell Installieren, wie auf den entsprechenden Produktseiten beschrieben.
   
#### Raspberry Pi 4 als Edge aufsetzen

Der Raspberry Pi 4 hat mehr Leistung, deshalb lohnt es sich eine [Kubernetes](https://kubernetes.io/de/) Umgebung einzurichten.

Dazu verwenden wir [Ubuntu Server für ARM 64bit](https://ubuntu.com/download/raspberry-pi). Eine ausführliche Anleitung gibt es [hier](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#3-wifi-or-ethernet).

Anschliessend erfolgt die Installation von der Ubuntu Kubernetes Distribution [microk8s](https://microk8s.io/). Eine ausführliche Anleitung gibt es [hier](https://ubuntu.com/tutorials/how-to-kubernetes-cluster-on-raspberry-pi#1-overview).

Nach der Installation der benötigten Grundinfrastruktur können wir loslegen und die eigentliche Software als Container starten.

Wir verwenden wie im [Workflow Beispiel](../workflow) [Node-RED](https://nodered.org/) und Mosquitto und bauen das Beispiel dann Stück für Stück aus.
    
    # IoT Umgebung 
    kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/iot/mosquitto-arm64.yaml
    kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/iot/nodered-arm64.yaml
    
Der Mosquitto Broker ist auf Port 31883 (nur MQTT-Protokoll, kein UI) und Node-Red auf Port 32600 verfügbar.    

***
### Cloud Umgebung

**Variante a) lernMAAS mit ModTec Umgebung**

In einer [lernMAAS](https://github.com/mc-b/lernmaas) Umgebung ist eine VM mit Namen `modtec-XX` (XX = Hostanteil für VPN) zu erstellen und auf dem lokalen Notebook kann die 
Umgebung vom [ModTec Kurs](https://github.com/mc-b/modtec) verwendet werden. 

Bei beiden Umgebungen sind alle benötigten Services wie Camunda, Node-RED, Kafka etc. bereits gestartet.

Evtl. muss der BPMN Prozess noch veröffentlicht werden.

**Variante b) Cloud und Cloud-init**

Erstellen Sie einen Account für die [AWS](https://aws.amazon.com/de/free/) oder [Azure](https://azure.microsoft.com/de-de/overview/what-is-azure/iaas/) Cloud.

Erstellen Sie eine neue Virtuelle Maschine mit min. 2 CPUs, 4 GB RAM. Als Betriebsystem nehmen Sie Ubuntu ab der Version 18.04.

Im Feld `custom data` o.ä. füllen Sie folgende [Cloud-init](https://cloudinit.readthedocs.io/en/latest/) Konfiguration ein:

    #cloud-config
    users:
      - name: ubuntu
        sudo: ALL=(ALL) NOPASSWD:ALL
        groups: users, admin
        home: /home/ubuntu
        shell: /bin/bash
        lock_passwd: false
        plain_text_passwd: 'password'        
    # login ssh and console with password
    ssh_pwauth: true
    disable_root: false    
    packages:
      - unzip
    runcmd:
      - sudo snap install microk8s --classic
      - sudo usermod -a -G microk8s ubuntu
      - sudo microk8s enable dns  
      - sudo mkdir -p /home/ubuntu/.kube
      - sudo microk8s config >/home/ubuntu/.kube/config
      - sudo chown -f -R ubuntu /home/ubuntu/.kube
      - sudo snap install kubectl --classic
      - sudo mkdir /data
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/lernkube/master/data/DataVolume.yaml
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/addons/dashboard-skip-login-no-ingress.yaml
      - (cd /tmp; git clone https://github.com/mc-b/mlg; cd mlg; microk8s kubectl apply -f jupyter/jupyter-mlg.yaml; cp -rpv data/* /data/ )
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/iot/mosquitto.yaml
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/kafka/zookeeper.yaml
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/kafka/kafka.yaml
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/iot.kafka/master/iot-kafka-alert.yaml
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/iot.kafka/master/iot-kafka-consumer.yaml
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/iot.kafka/master/iot-kafka-pipe.yaml
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/iot/nodered-kafka.yaml
      - sudo microk8s kubectl apply -f https://raw.githubusercontent.com/mc-b/misegr/master/bpmn/camunda.yaml
      - sudo apt-get update  
      - sudo apt-get install -y mosquitto-clients curl git unzip
      - sudo chmod -R o=u,g=u /data  
      - sudo chmod 777 /data 

Nach erfolgreicher Installation sind folgende Services verfügbar:
* [Camunda BPMN Workflow Engine](https://camunda.com/) auf http://<ip vm>:30880/camunda/
* [mosquitto](https://projects.eclipse.org/projects/iot.mosquitto) auf Port 31883 (nur MQTT Protokoll ohne UI)
* [Node-RED](https://nodered.org/) auf http://<ip vm>:32600
* [Jupyter mit Machine Learning Notebooks](https://jupyter.org/) auf http://<ip vm>:32088
* [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) auf https://<ip vm>

**ACHTUNG**: das ist eine Lernumgebung, werden die Ports gegen das gesamte Internet geöffnet ist die VM ungeschützt im Internet. Ports nur für eigene IP-Adresse (Router) öffnen.

**Veröffentlichung des BPMN Prozesses**

Die Pipeline braucht einen vorbereitenden [BPMN Prozess](https://github.com/mc-b/bpmn-tutorial#schritt-2---prozess-ver%C3%B6ffentlichen-und-ausf%C3%BChren). Diesen können Sie über das Juypter Notebook http://<ip vm>:32088/notebooks/Microservices-BPMN-microk8s.ipynb veröffentlichen.

#### Node-RED (MQTT - nach HTTP/REST)

![](https://raw.githubusercontent.com/iotkitv3/intro/main/images/NodeREDREST.png)

- - -

Die MQTT Messages vom IoTKit Board sollen vom MQTT Protokoll in das HTTP/REST Protokoll umgewandelt werden.

Um dies zu Demonstrieren erstellen wir eine Input MQTT Node welche das Topic `iotkit/alert` empfängt, eine Node welche eine JSON Struktur erzeugt und eine Node welche die JSON Daten via HTTP/REST an die BPMN Workflow Umgebung weiterleitet.

* In Node-RED
    * `mqtt` Input Node auf Flow 1 platzieren, mit Mosquitto Server verbinden und als Topic `iotkit/alert` eintragen.
    * `debug` Output Node auf Flow 1 platzieren und mit Input Node verbinden - damit können wir die MQTT Messages kontrollieren
    * `change` Node auf Flow 1 platzieren und als `msg.payload` folgenden JSON String `{"variables":{"rnr":{"value":"123","type":"long"},"rbetrag":{"value":"200.00","type":"String"}}}` eintragen.
    * `http request` Node auf Flow 1 platzieren, als Methode `POST` und als URL `http://camunda:8080/engine-rest/process-definition/key/RechnungStep3/start` eintragen
    * Zur Kontrolle ebenfalls eine `debug` noch hinten platzieren.
    * Alle Nodes wie oben in der Grafik verbinden und veröffentlichen (deploy).
* In Camunda BPMN Workflow Engine [https://localhost:30443/camunda](https://localhost:30443/camunda) (URL kann abweichen, je nach Umgebung) einloggen mittels User/Password `demo/demo`. Bei jedem Alarm welcher vom Board (Hall Sensor) mittels Magneten ausgelöst wird, sollte ein neuer Rechnungsprozess gestartet werden.
    
Details zum BPMN Prozess und der URL wie ein Prozess gestartet werden kann steht im [Frontend](https://github.com/mc-b/misegr/tree/master/bpmn) Beispiel bei `$.post`.

Der Flow zum importieren und anpassen, siehe [Node-RED-REST.json](Node-RED-REST.json).

#### Node-RED (MQTT - Kafka Messaging)

![](https://raw.githubusercontent.com/iotkitv3/intro/main/images/NodeREDKafka.png)

- - -

Die MQTT Messages sollen nun an [Apache Kafka](https://kafka.apache.org/) weitergeleitet werden. Das hat den Vorteil, dass wir diese
* in andere Formate, z.B. von Binär nach JSON, umwandeln können
* sie Persistieren können
* ein Eventlog erhalten
* etc.

Um [Apache Kafka](https://kafka.apache.org/) anzusprechen brauchen wir ein paar zusätzliche Plugins.

Diese können in Node-RED mittels Pulldownmenu rechts -> `Palette verwalten`, Tab `Installieren` hinzugefügt werden. Es handelt es sich um die Plugins:
* node-red-contrib-kafka-node-latest - mindestens Version 0.2
* node-red-contrib-kafka-manager - letzte Version

Dadurch erhalten wird neu `Nodes` für die Integration mit [Apache Kafka](https://kafka.apache.org/).    

* In Node-RED
    * `mqtt` Input Node auf Flow 1 platzieren, mit Mosquitto Server verbinden, als Topic `iotkit/#` und bei Output `a String` eintragen.
    * `debug` Output Node auf Flow 1 platzieren und mit Input Node verbinden - damit können wir die MQTT Messages kontrollieren
    * `change` Node auf Flow 1 platzieren und als Regel `Ändern` Wert `msg.topic` von `iotkit/alert` in `broker_message`, weitere Regel hinzufügen und gleich verfahren für `iotkit/sensor` nach `broker_message`.
    * Kafka Producer auf Flow 1 platzieren und mit Kafka Server (kafka:9092) verbinden
    * Alle Nodes wie oben in der Grafik verbinden und veröffentlichen (deploy).
* Kubernetes    
    * Das Ergebnis kann mittels `logs iot-kafka-consumer`, logs iot-kafka-pipe` angeschaut werden. Dort sollten die MQTT Messages umgewandelt als Kafka Messages erscheinen.
* In Camunda BPMN Workflow Engine [https://localhost:30443/camunda](https://localhost:30443/camunda) (URL kann abweichen, je nach Umgebung) einloggen mittels User/Password `demo/demo`. Bei jedem Alarm welcher vom Board (Hall Sensor) mittels Magneten ausgelöst wird, sollte ein neuer Rechnungsprozess gestartet werden.
    

Topics auslesen, lesen und schreiben auf Topics in Kafka Container, siehe [Projekt duk](https://github.com/mc-b/duk/tree/master/kafka).

Der Flow zum importieren und anpassen, siehe [Node-RED-Kafka.json](Node-RED-Kafka.json).

#### Machine Learning mit Juypter Notebooks

![](https://raw.githubusercontent.com/iotkitv3/intro/main/images/MachineLearning.png)

Ein [Jupyter Notebook](https://jupyter.org/) ist eine Open-Source-Webanwendung, mit der Sie wiederholende Abläufe erstellen können. Die Live-Code, Gleichungen, Visualisierungen und Text enthalten können.

Verwendungsmöglichkeiten:
* Datenbereinigung und -transformation
* numerische Simulation
* statistische Modellierung
* Datenvisualisierung
* maschinelles Lernen und vieles mehr.

Jupyter Notebooks laufen lokal, ein einem Container oder in der Cloud.

**Maschinelles Lernen** ist ein Oberbegriff für die «künstliche» Generierung von Wissen aus Erfahrung. Ein künstliches System lernt aus Beispielen und kann diese nach Beendigung der Lernphase verallgemeinern. Das heisst, es werden nicht einfach die Beispiele auswendig gelernt, sondern es «erkennt» Muster und Gesetzmässigkeiten in den Lerndaten. 

Das Juypter Notebook [MLTempHumSensor](https://github.com/mc-b/mlg/blob/master/data/mlg/02-2-MLTempHumSensor.ipynb) demonstriert [Predictive Maintenance](https://de.wikipedia.org/wiki/Pr%C3%A4diktive_Instandhaltung) anhand von Demodaten.

Sollen die Live Daten des IoTKitV3 ausgewertet werden, ist der Code unter "Gegenprüfung mit Testdaten" wie folgt zu ändern:

    test = pd.read_csv('ml-data.csv', header=None, names=['sensor', 'temp', 'hum', 'class'] )
    
Wird das Beispiel jetzt nochmals von vorne durchgespielt (Kernel -> Restart & Run All), erfolgt ein Vergleich mit den Live Daten des IoTKitV3.    
