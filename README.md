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


## Configuration

### rfxcom2mqtt

Dans le fichier `config.yml` de rfxcom2mqtt :

* `mqtt.base_topic` : le topic à renseigner dans dknr-rfxcom (`rfxcom2mqtt` par défaut).
* `rfxcom.receive` : la liste des protocoles à recevoir (par exemple `temperaturehumidity1`, `lighting2`, `security1`...). Seuls les protocoles listés sont publiés.
* `homeassistant.discovery` : à passer à `false` si vous utilisez la discovery de dknr-rfxcom, pour éviter d'avoir chaque appareil en double dans Home Assistant.

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

Le protocole doit aussi être déclaré dans `rfxcom.transmit` de rfxcom2mqtt.

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
