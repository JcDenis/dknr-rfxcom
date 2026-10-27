# README

[![Release](https://img.shields.io/docker/v/jcpd/dknr-rfxcom?sort=semver)](https://img.shields.io/docker/v/jcpd/dknr-rfxcom?sort=semver)
[![Pull](https://img.shields.io/docker/pulls/jcpd/dknr-rfxcom)](https://img.shields.io/docker/pulls/jcpd/dknr-rfxcom)
[![Issue](https://img.shields.io/github/issues/jcdenis/dknr-rfxcom)](https://img.shields.io/github/issues/jcdenis/dknr-rfxcom)
[![License](https://img.shields.io/github/license/jcdenis/dknr-rfxcom)](https://github.com/JcDenis/dknr-rfxcom/blob/master/LICENSE)


## A propos

**dknr-rfxcom** est un projet consacré à la récupération et à l'exploitation des données des appareils radio 433 MHz reçus par un émetteur-récepteur **RFXCom RFXtrx433**.

Ce projet ne pilote pas directement le RFXtrx : il s'appuie sur [rfxcom2mqtt](https://github.com/rfxcom2mqtt/backend) qui publie les trames reçues sur votre serveur MQTT. dknr-rfxcom les traduit dans l'arborescence MQTT commune aux projets DKNR et les déclare à Home Assistant.


## Fonctionnalités

- **Traduction des messages rfxcom2mqtt** vers des valeurs simples, une par entité.
- **Sondes météo** : température, humidité, confort, pression, prévision, pluie, vent (direction, vitesse, rafales, ressenti), UV.
- **Energie** : puissance, énergie, intensité par phase, tension, compteurs, téléinfo Cartelectronic.
- **Sécurité** : état, mouvement, alarme, sabotage.
- **Volets, thermostats, télécommandes** : dernière commande reçue et date du dernier signal.
- **Prises et interrupteurs** (lighting1, lighting2, lighting5) : état et pilotage.
- **Batterie et qualité du signal** en pourcentage.
- **Liste de blocage** pour ignorer les appareils des voisins.
- **Support Home-Assistant Discovery** en lecture et pilotage des prises et interrupteurs.


## Matériel requis

Le matériel exact dépend de l'installation, mais l'utilisation du projet nécessite généralement :

* un émetteur-récepteur **RFXCom RFXtrx433** (ou RFXtrx433XL).
* le logiciel **rfxcom2mqtt** (image docker `rfxcom2mqtt/rfxcom2mqtt`) connecté au RFXtrx et à votre serveur MQTT.
* un serveur MQTT.
* une machine exécutant le logiciel docker.


## Installation

Des images Docker prêtes à l'emploi sont disponibles. 
Utilisez le fichier docker-compose.yaml à la racine du dépôt en fournissant les variables d'environnement suivantes :

* **DKNR_USERNAME** : login de l'interface node-red.
* **DKNR_PASSWORD** : mot de passe de l'interface node-red.
* **DKNR_SECRET**   : un mot secret pour encrypter les credits.

Le tableau de bord est disponible sur `http://<ip du serveur docker>:1880/` et l'éditeur Node-RED sur `http://<ip du serveur docker>:1880/admin`.

### Exemple complet avec rfxcom2mqtt

rfxcom2mqtt peut être entièrement configuré par variables d'environnement, sans fichier `config.yml` :

```yaml
services:
  rfxcom2mqtt:
    container_name: rfxcom2mqtt
    image: rfxcom2mqtt/rfxcom2mqtt
    restart: unless-stopped
    volumes:
      - rfxcom2mqtt:/app/data
    environment:
      - TZ=Europe/Paris
      - MQTT_SERVER=mqtt://192.168.0.11:1883
      - MQTT_USERNAME=${MQTT_USERNAME}
      - MQTT_PASSWORD=${MQTT_PASSWORD}
      - RFXCOM_USB_DEVICE=/dev/ttyUSB0
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    logging:
      driver: "json-file"
      options:
        max-size: "20m"
        max-file: "5"

  dknr_rfxcom:
    container_name: dknr_rfxcom
    image: jcpd/dknr-rfxcom:latest
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    environment:
      TZ: Europe/Paris
      DKNR_USERNAME: ${DKNR_USERNAME}
      DKNR_PASSWORD: ${DKNR_PASSWORD}
      DKNR_SECRET: ${DKNR_SECRET}
    ports:
      - 1880:1880
    volumes:
      - dknr_rfxcom:/data
    healthcheck:
      test: timeout 10s bash -c ':> /dev/tcp/127.0.0.1/1880' || exit 1
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 90s
    logging:
      driver: "json-file"
      options:
        max-size: "20m"
        max-file: "5"

volumes:
  rfxcom2mqtt:
  dknr_rfxcom:
```

Les variables `MQTT_USERNAME`, `MQTT_PASSWORD`, `DKNR_USERNAME`, `DKNR_PASSWORD` et `DKNR_SECRET` sont à définir dans un fichier `.env` à côté du `docker-compose.yaml`. Attention, rfxcom2mqtt attend `MQTT_USERNAME` et non `MQTT_USER`.


## Configuration

### rfxcom2mqtt

Sans fichier `config.yml`, rfxcom2mqtt utilise ses réglages par défaut :

* le topic `rfxcom2mqtt`, à renseigner dans dknr-rfxcom ;
* la réception des protocoles `temperaturehumidity1`, `homeconfort`, `lighting1` à `lighting4`, `remote` et `security1`. Les autres sondes (vent, pluie, énergie, volets...) nécessitent de compléter la liste `rfxcom.receive` dans un fichier `config.yml` placé dans le volume `/app/data` ;
* sa propre discovery Home Assistant, activée. Les appareils apparaissent alors deux fois dans Home Assistant : ceux de rfxcom2mqtt et ceux de dknr-rfxcom. Pour l'éviter, désactivez l'une des deux : laissez vide le champ discovery de dknr-rfxcom, ou mettez `homeassistant.discovery: false` dans un fichier `config.yml` de rfxcom2mqtt.

### dknr-rfxcom

Dans la page **Configurer** :

* **Adresse IP**, **port**, **utilisateur** et **mot de passe** du serveur MQTT.
* **Topic** : racine des topics DKNR (`dknr` par défaut).
* **RFXCom2MQTT topic** : le `base_topic` de rfxcom2mqtt.
* **Home assistant discovery topic** : `homeassistant` par défaut, laisser vide pour désactiver.
* **Activation**.

Les appareils apparaissent dans la page **Voir** dès leur première trame reçue.

### Bloquer des appareils

Un RFXtrx reçoit aussi les sondes et télécommandes du voisinage. Dans la page **Bloquer**, ajoutez une **clé** et une **valeur** du message rfxcom2mqtt à ignorer, par exemple :

* `id` = `0x5C02` pour ignorer un appareil précis.
* `subTypeValue` = `RM174RF` pour ignorer tout un modèle.
* `type` = `security1` pour ignorer tout un protocole.


## MQTT

Avec le topic `dknr` par défaut, l'identifiant d'un appareil est son identifiant RFXCom sans `0x` (`5C02` pour `0x5C02`) :

* `dknr/get/rfxcom/<appareil>/<entité>` : valeurs publiées à chaque changement.
* `dknr/set/rfxcom/<appareil>/command<unité>` : `ON` ou `OFF` pour piloter une prise ou un interrupteur.
* `dknr/discovery/rfxcom/<appareil>/<entité>` : description des entités.
* `homeassistant/<type>/rfxcom/<appareil>_<entité>/config` : Home Assistant Discovery.

Principales entités : `temperature`, `humidity`, `humidityStatus`, `barometer`, `forecast`, `rainfall`, `rainfallRate`, `windDirection`, `windSpeed`, `windGust`, `chillFactor`, `uv`, `co2`, `weight`, `power`, `energy`, `current`, `voltage`, `deviceStatus`, `motion`, `alarm`, `tampered`, `command<unité>`, `group`, `level<unité>`, `lastSeen`, `batteryLevel`, `rssi`.

Les sondes à plusieurs mesures numérotent leurs entités (`temperature1`, `temperature2`, `current1`...).

### Piloter une prise ou un interrupteur

Une prise ne répond pas par radio : dknr-rfxcom doit avoir **reçu au moins une fois sa télécommande** pour connaître son protocole et son code. Appuyez une fois sur la télécommande, l'entité `command<unité>` apparaît alors et devient pilotable, y compris après un redémarrage. La commande est transmise à rfxcom2mqtt sur `rfxcom2mqtt/cmd/<type>/<subtype>/<id>/<unité>`.

L'état affiché après une commande est l'état demandé : la prise ne confirme pas qu'elle l'a reçu.


## A faire

* Pilotage des volets (blinds, rfy).
* Pilotage des groupes et des appareils lighting6.
* Renommer les appareils depuis le tableau de bord.


## Contribuer

Ce projet est **open-source**, vous pouvez participer en créant des [tickets](https://github.com/JcDenis/dknr-rfxcom/issues) ou en proposant de [requêtes](https://github.com/JcDenis/dknr-rfxcom/pulls) sur le dépôt github du projet.


## Voir aussi

- [DKNR Teleinfo](https://github.com/JcDenis/dknr-teleinfo)
- [DKNR APSystems](https://github.com/JcDenis/dknr-apsystems)
- [DKNR Daikin](https://github.com/JcDenis/dknr-daikin)
- [DKNR Zendure](https://github.com/JcDenis/dknr-zendure)
- [DKNR Nuki](https://github.com/JcDenis/dknr-nuki)
- [rfxcom2mqtt](https://github.com/rfxcom2mqtt/backend)
- [node-rfxcom](https://github.com/rfxcom/node-rfxcom)
