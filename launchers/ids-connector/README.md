# EDC Connector mit Integration von IDS-Protokollen

Diese Dokumentation beschreibt wie Komponente eines Datenraums konfiguriert werden sollten, damit
wir einen funktionsfähigen Datenraum implementieren. Diese Komponente sind ein Connector und ein 
DAPS-Server.

**Bemerkung:** Diese Dokumentation basert sich auf die Eclipse-Dokumentation.


# Szenario

Der Datentaum hat die Aufgabe, einen sicheren Datenaustauch und eine Datensouveränität zu ermöglichen.
In diesem Szenario haben wir einen Provider, der Daten durch sein Connector zur Verfügung stellt. Eine 
Person, die auf diese Daten zugreifen möchte, sollte nämlich auch über ein Connector haben, das eine
Verbindung zu einem Endpunkt, wo die Daten gespeichert werden. Die Provider und Consumer-Connectors
sehen nämlich ähnlich aus und enterscheiden sich nur mit ihren Parametern. Die Teilnehmer (Provider und
Consumer) werden entweder lokal oder auf einem Docker-Container installiertg. Was den Container angeht,
wird ein Docker-Image mithilfe der CI/CD Pipeline erstellt. die wir auf Docker Hub speichern und von dort 
heruntergeladen werden kann. Die Connectors haben Besonderheiten, die wir im Rahmen der Entwicklung genauer
angucken werden. Um auch den Datenraum teilzunehmen, werden bestimmte Attribute an jeden neuen Teilnehmer 
geschickt, die von einem Identity-Probider erstellt werden, damit jeder Teilnehmer eindeutig im Datenraum 
identifiziert wird. Nach dieser Phase, wird es möglich sein, andere Schritte durchzuführen, die am Ende
zu einem Datentransfer führen werden. 

## Voraussetzungen

Die Voraussetzungen der Entwicklung können einfach im Szenario des Datenraums verstanden werden. 
Für das definierte Szenario dieses Projekt werden nämlich drei VMs (virtuelle Maschinen) gebraucht.
Die ersten zwei entsprechen den Teilnehmern des Datenraums (Provider und Consumer). Diese Teilnehmer
brauchen jeweils ein Connector für die Realisierung eines Datenaustauchs oder einer Verbindung mit
Datenquellen oder senken. Die letze VM wird für die Installation des DAPS-Servers benötigt, der als
als Identity Provider betrachtet wird. Auf dieser VM wird auch die Zertifikate von jedem Teilnehmer
und DAPS selbst gespeichert. Diese Zertifikate werden in Keystore (Format .p12) gespeichert und and
die Teilnehmer geschickt. Wir benutzen kein SSL-Zerifikat.


## Module

Alle Module, die in der Bacherlorarbeit definiert werden, sind tatsächlich die Module, die wir für 
die Entwicklung des Connectors benutzt haben. Diese sind auch in der Bachelorarbeit sachlich beschreibt.

Das folgende Bild zeigt eine Übersicht dieser Module in unserer [Build-Datei](./build.gradle.kts) : 

```kotlin
// in build.gradle.kts:
    implementation(project(":core:control-plane:control-plane-core"))
    implementation(project(":core:data-plane-selector:data-plane-selector-core"))
    implementation(project(":core:data-plane:data-plane-core"))
    
    implementation(project(":data-protocols:ids"))

    implementation(project(":extensions:common:configuration:configuration-filesystem"))
    implementation(project(":extensions:common:vault:vault-filesystem"))

    implementation(project(":extensions:common:iam:oauth2:oauth2-service"))
    implementation(project(":extensions:common:iam:oauth2:oauth2-daps"))

    implementation(project(":extensions:control-plane:api:management-api"))

    implementation(project(":extensions:common:auth:auth-tokenbased"))
    
    implementation(project(":extensions:control-plane:transfer:transfer-data-plane"))
    
    implementation(project(":extensions:common:api:api-observability"))
    
    implementation(project(":extensions:data-plane:data-plane-http"))
    implementation(project(":extensions:data-plane-selector:data-plane-selector-client"))
```
Die beschreibung dieser Module könnte auch in der Eclipse-Dokumentation gefunden werden.


## Konfiguration des Connectors

Die Module, die in der Build-Datei vom Connector defniert werden, sind nicht ausreichend, um ein Connector
mühelos installieren. Manche Modulen brauchen zusätzliche Parameter, die erforderlich sein werden, um ein 
Connector sorglos zu implementieren. Alle benötigte Parameter sind in unserer [config.properties](./config.properties) Datei 
gespeichert und eine konkrete Beschreibung jedes Parameters könnte auf der [Eclipse-Dokumentation](https://github.com/eclipse-edc/Connector/tree/main/launchers/ids-connector#readme) gefunden werden. Ein wichtiger Punkt dabei wäre zu erwähnen, dass die Definition dieser
Parameter nur ermöglicht wird, wenn das Modul **extensions:common:configuration:configuration-filesystem**
in der Konfiguration vorhanden ist.

## Ausführen des Connectors

Da wir das Connector jetzt konfiguriert haben, sind wir normalerweise in der Lage dieses zu starten.
Das Connector kann auf zwei verschiedene Arten (Lokal oder auf einem Container) gestartet werden und 
daher werden wir diese zwei Möglichkeiten vorstellen. Es gibt noch manche Parameter, die für Ausführung 
notwendig sind, egal welche Art der Ausführung wir benutzten.

Diese Parametern sind:

* `edc.fs.config`: Der Pfad der `config.properties` Datei.
* `edc.vault`: Der Pfad der `vault.properties` Datei (kann auch den gleichen Pfad
wie den Pfad der `config.properties` Datei).
* `edc.keystore`: Der Pfad bis zum Keystore.
* `edc.keystore.password`: Passwort vom Keystore.


### Lokaler Aufbau

Zur Ausführung des Connectors ist die .jar Datei erforderlich. Diese Datei wird mithilfe des Gradle wrapper 
erstellt. Also Zur Erstellung des Connestors werden folgende Kommandos durchgeführt: 

```shell
./gradlew clean :launchers:ids-connector:build
java -Dedc.fs.config=<path-to-config.properties> \
    -Dedc.vault=<path-to-config.properties> \
    -Dedc.keystore=<path-to-keystore> \
    -Dedc.keystore.password=<keystore-password> \
    -jar launchers/ids-connector/build/libs/dataspace-connector.jar
```

### Docker Image

Um das Doker Image zu erstellen haben wir eine [CI/CD-Pipeline](https://github.com/Thierry1405/DataSpaceConnector/blob/edc-connector/.github/workflows/ci.yml)
erstellt, die erstmal die Konfigurationen überprüft und dann erstellt ein Docker Image auf Basis
der resultierenden .jar Datei, das auf Docker Hub gespeichert wird. Von dort kann das Image mit dem
folgenden Kommando heruntergeladen:

```shell
docker pull djeutchou/edc-ids-connector:latest
```
Wenn wir das Image erfolgreich heruntergeladen haben, sind wir jetzt in der Lage dieses in einem
Container mit notwendigen Parametern wie folgt starten:

```shell
docker run -p 8181:8181 -p 8182:8182 -p 8282:8282  \
    --env-file ./launchers/ids-connector/ids-connector.env \
    -v '/directory/with/properties:/config/config.properties' \
    -v '/directory/with/keystore:/config/keystore.p12' \
    edc-ids-connector
```
**Bemerkung:** In der [ids-connector.env](https://github.com/Thierry1405/DataSpaceConnector/blob/edc-connector/launchers/ids-connector/ids-connector.env) Datei werden nur Variable definiert.



### Aufbau des lokalen DAPS-Servers

Nachdem wir eine Lösung für die Erstellung eines Connectors gefunden haben, ist es jetzt wesentlich
einen DAPS-Server einzurichten. Die folgenden Schritte definieren wie wir unseren DAPS-Server eingerichtet
haben(alle Kommandos werden im Omejdn Repository ausgeführt werden):

1. Überprüfen Sie [Omejdn DAPS repository](https://github.com/International-Data-Spaces-Association/omejdn-daps)
Dieses Repository erklärt die Prinzipe eines DAPS-Servers und wie er läuft.
2. Rufen Sie die Submodule ab: 
   ```bash
   git submodule update --init --remote
   ```
3. Generieren Sie einen Schlüssel und ein Zertifikat für die DAPS-Instanz:
   ```bash
    openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout keys/omejdn/omejdn.key -out daps.cert
   ```
Wie wir oben schon erklärt haben, verwenden wir keine Zertifizierungsstelle und deshalb soll das DAPS-Zertfikat selbst erstellt.</br>
4. Im Projektstamm ändern Sie die `.env` Datei: `DAPS_DOMAIN` auf die URL setzen, unter der Ihre DAPS-Instanz ausgeführt wird.
Die `.env` Datei enthält die Konfigurationen des DAPS-Servers.</br>
5. Registrieren Sie einen Connector (das Sicherheitsprofil ist optional und wird standardmäßig auf *idsc:BASE_SECURITY_PROFILE* gesetzt, 
wenn es nicht angegeben WIRD):  
  ```bash
   scripts/register_connector.sh <client-name-for-connector> <security-profile>
   ```
Nach der Regestrierung des Connectors werden zwei Dateien erstellt (Ein Schlüssel <client-name-from-step-5>.key und ein Zertifikat <client-name-from-step-5>.cert).
und im Verzeichnis `keys` gespeichert.</br>
6. Erstellen Sie ein Keystore, das dei beide Dateien speichert.
    ```bash
    openssl pkcs12 -export -in keys/<client-name>.cert -inkey keys/<client-name>.key -out <client-name>.p12
    ```
Im resultierenden Schlüsselspeicher (Keystore) hat das Zertifikat den Alias 1​​.</br>
7. Schicken Sie das Keystore an das Connector.
   ```bash
   scp omejdn-daps/keystor.p12 user@xxx.xxx.xxx.xxx:/home/remote_dir
   ```
  </br>
8. Optional können Sie weitere Connectors registrieren, indem Sie Schritt 5 mehrmals mit unterschiedlichen Clientnamen ausführen. </br>
9. Führen Sie das DAPS aus:
   ```bash
   docker compose -f compose.yml up
   ```
  </br>
10. Wenn Sie `omejdn-server_1  | == Sinatra (v2.1.0) has taken the stage on 4567 for development with backup from Thin`
   in die Logs sehen, ist das DAPS bereit, Anfragen anzunehmen.
   
Die URL, unter der der Konnektor das DAPS erreichen kann, ergibt sich aus `http://localhost:80` (akann aber in der `.env` Datei verändert werden),
der NGINX in der `docker-compose` Datei verwendtet.

Die obenstehenden Schritte können auch in der [Eclipse-Dokumentation](https://github.com/eclipse-edc/Connector/tree/main/launchers/ids-connector#readme) gefunden werden.
