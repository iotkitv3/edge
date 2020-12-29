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
    
Installation und Konfiguration
------------------------------

### Internet of Things (IoTKitV3)
***

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

### Edge (Raspberry Pi)
***

#### Raspberry Pi 2 oder 3 als Edge aufsetzen

Der Raspberry Pi (2, 3) eignet sich als Edge. 

Dazu ist der Raspbery Pi (2, 3) wie folgt aufzusetzen:

* Raspbian Lite wie auf [raspberrypi.org](https://www.raspberrypi.org/) beschrieben, downloaden und auf SD Karte speichern.
* Auf der SD Karte eine Datei «ssh» ohne Endung erstellen.
* Auf Windows Git/Bash oder Putty o.ä. installieren. Mac und Linux brauchen nur Git.
* Raspberry Pi via Kabel mit dem Router LERNKUBE verbinden und via ssh Verbinden
    
Grundinstallation:
  
    ssh 192.168.2.xx -l pi
    PW: raspberry
    WLAN, Zeitzone etc. in raspi-config Einstellen
    sudo raspi-config

Quelle: https://howtoraspberrypi.com/how-to-raspberry-pi-headless-setup/ 

Anschliessend bietet sich [Docker](https://docker.com) als Container Umgebung an. Docker kann wie folgt installiert werden:

    ssh 192.168.2.xx -l pi
    sudo -i
    cd /tmp
    wget -O get-docker.sh http://get.docker.com
    sh get-docker.sh
    sudo usermod -aG docker pi
    exit

Testen der Docker Umgebung:

    ssh 192.168.2.xx -l pi
    docker run hello-world
    
Nach der Installation der benötigten Grundinfrastruktur können wir loslegen und die eigentliche Software als Container starten.

Wir verwenden wie im [Workflow Beispiel](../workflow) [Node-RED](https://nodered.org/) und Mosquitto und bauen das Beispiel dann Stück für Stück aus.

    docker run -d -p 1880:1880 nodered/node-red
    docker run -d -p 1883:1883 eclipse-mosquitto
    
Die eigentliche Applikation Node-RED ist via Browser <IP-Raspberry Pi:1880> zugreifbar.


#### Raspberry Pi 4 als Edge aufsetzen

Grundsätzlich kann der Raspberry Pi 4 wie die Version 2, 3 installiert werden. Aber statt Docker installieren wir, die leichtgewichtige Kubernetes Variante [k3s](https://k3s.io/).

Installationsschritte wie oben aber ohne Docker ausführen, stattdessen installieren wir [k3s](https://k3s.io/).

    #   Installiert Rancher k3s ohne Ingress Controller traefik
    curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--no-deploy=traefik" sh -s -

    # pi User als Admin zulassen
    sudo mkdir -p /home/pi/.kube
    sudo cp /etc/rancher/k3s/k3s.yaml /home/pi/.kube/config
    sudo chown -R pi:pi /home/pi/.kube
    sudo chmod 700 /home/pi/.kube
    sudo echo 'export KUBECONFIG=$HOME/.kube/config' >>/home/pi/.bashrc 
    
Optional lohnt es sich dem Raspberry Pi ein Web-UI zu spendieren. Das vereinfacht den Zugriff auf die Kubernetes Ressourcen.

    # Apache Server als Web UI installieren
    sudo apt install -y apache2 jq markdown
    sudo a2enmod cgi
    sudo systemctl restart apache2

    # Web-UI einrichten
    git clone https://github.com/mc-b/IoTKitV3
    cd IoTKitV3/edge/
    sudo cp cgi-bin/* /usr/lib/cgi-bin/
    sudo chmod +x /usr/lib/cgi-bin/*
    sudo cp html/* /var/www/html/
    
    # www-data User Zugriff auf Kubernetes erlauben
    sudo mkdir -p /var/www/.kube
    sudo cp /etc/rancher/k3s/k3s.yaml /var/www/.kube/config
    sudo chown -R www-data:www-data /var/www/.kube
    sudo chmod 700 /var/www/.kube    
   
Nach der Installation der benötigten Grundinfrastruktur können wir loslegen und die eigentliche Software als Container starten.

Wir verwenden wie im [Workflow Beispiel](../workflow) [Node-RED](https://nodered.org/) und Mosquitto und bauen das Beispiel dann Stück für Stück aus.
    
    # IoT Umgebung 
    kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/iot/mosquitto.yaml
    kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/iot/nodered.yaml
    
Über welche Ports die Services verfügbar sind kann im Web-UI nachgeschaut oder mittels des nachfolgenden Befehls ermittelt werden:

    kubectl get services
    
Werden mehrere Node-RED Umgebungen benötigt, können diese in neuen Kubernetes Namespaces gestartet werden.   

    kubectl create namespace nr1
    kubectl apply -n nr1 -f https://raw.githubusercontent.com/mc-b/duk/master/iot/nodered.yaml   

### Cloud Umgebung
***

**Variante a) lernMAAS oder ModTec Umgebung**

In einer [lernMAAS](https://github.com/mc-b/lernmaas) Umgebung ist eine VM mit Namen `modtec-XX` (XX = Hostanteil für VPN) zu erstellen und auf dem lokalen Notebook kann die 
Umgebung vom [ModTec Kurs](https://github.com/mc-b/modtec) verwendet werden. 

Bei beiden Umgebungen sind alle benötigten Services wie Camunda, Node-RED, Kafka etc. bereits gestartet.

Evtl. muss der BPMN Prozess noch veröffentlicht werden.

**Variante b) Manuelle Installation**

Installieren von 
* [Camunda BPMN Workflow Engine](https://camunda.com/)
* [Camunda Modeler](https://camunda.com/)
* Download des Rechnungsprozesses vom Projekt [misegr](https://raw.githubusercontent.com/mc-b/misegr/master/bpmn/RechnungStep3.bpmn)
* Import des Rechnungsprozesses [RechnungStep3.bpmn](https://raw.githubusercontent.com/mc-b/misegr/master/bpmn/RechnungStep3.bpmn) in den Modeler
* Export des Rechnungsprozesse vom Modeler in die BPMN Workflow Engine.

**Variante c) in einer Kubernetes Umgebung** 

Starten der [Camunda BPMN Workflow Engine](https://camunda.com/). Download Rechnungsprozesses [RechnungStep3.bpmn](https://raw.githubusercontent.com/mc-b/misegr/master/bpmn/RechnungStep3.bpmn). 

    kubectl apply -f https://raw.githubusercontent.com/mc-b/misegr/master/bpmn/camunda.yaml
    wget https://raw.githubusercontent.com/mc-b/misegr/master/bpmn/RechnungStep3.bpmn -O RechnungStep3.bpmn
    
warten bis die Workflow Umgebung gestartet ist und veröffentlichen des Rechnungsprozesses, mittels REST Schnittstelle:

    curl -k -w "\n" \
    -H "Accept: application/json" \
    -F "deployment-name=rechnung" \
    -F "enable-duplicate-filtering=true" \
    -F "deploy-changed-only=true" \
    -F "Rechnung.bpmn=@RechnungStep3.bpmn" \
    https://localhost:30443/engine-rest/deployment/create    

Details zu BPMN und dem Prozess [siehe](https://github.com/mc-b/misegr/tree/master/bpmn).
    
Starten der weiteren Services wie Node-RED, Kafka etc.

    # IoT Umgebung 
    kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/iot/nodered.yaml

    # Messaging Umgebung 
    kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/kafka/zookeeper.yaml
    kubectl apply -f https://raw.githubusercontent.com/mc-b/duk/master/kafka/kafka.yaml

    # Kafka Streams 
    kubectl apply -f https://raw.githubusercontent.com/mc-b/iot.kafka/master/iot-kafka-alert.yaml
    kubectl apply -f https://raw.githubusercontent.com/mc-b/iot.kafka/master/iot-kafka-consumer.yaml
    kubectl apply -f https://raw.githubusercontent.com/mc-b/iot.kafka/master/iot-kafka-pipe.yaml     
       

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
