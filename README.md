[![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/ysard/telewatt)](https://github.com/ysard/telewatt/releases/latest/)
[![CC BY-NC-SA 4.0][cc-by-nc-sa-shield]][cc-by-nc-sa]

# TeleWatt

## Projet et avant-propos

Ce circuit a pour but de traiter les informations émises depuis
la sortie Télé-Information Client (TIC) des compteurs
d'électricité installés par le fournisseur français Enedis.

La sortie TIC offre aux clients la possibilité d’être informés en temps réel
des grandeurs électriques mesurées (périodes tarifaires, contacts virtuels,
puissance instantanée, etc.).

Ce système avec son firmware interprète les données, les conserve sur un ESP8266
et/ou les envoie sur un serveur personnel. Ceci garantit une pleine maîtrise
de celles-ci sans passer par un prestataire **coûteux** et parfois opaque.


Par ailleurs, les pas de collecte de 1h ou 30 minutes sont déjà particulièrement
intrusifs s'ils ont été souscrits auprès du fournisseur.
Un pas de collecte plus court permit par les modules Téléinfo l'est encore plus.
Par ailleurs, les modules du commerce ne viennent en général jamais seuls :
ils peuvent être couplés à des mesures de température, d'humidité, etc.
Des économies sont promises, mais c'est surtout un profilage à l'usage qui est obtenu.


## Circuit

### Généralités

Le raccordement se fait sur le bornier du compteur comprenant 3 bornes (L1, L2, A).
La borne A permet la conception de circuits autoalimentés à l'aide d'un
super condensateur garantissant une certaine autonomie (mais pas un fonctionnement continu).

Les anciens compteurs avec seulement les bornes L1 et L2 sont compatibles,
à condition d'opter pour la version du circuit requérant une alimentation externe (USB - 5V).

La carte se charge à 100% en 11 minutes et autorise 4 minutes d'autonomie
avec un condensateur de 3.5F.
Un tel condensateur est très largement surdimensionné pour une brève
connexion MQTT toutes les 5 minutes.

Les composants sont choisis pour être courants, peu coûteux et faciles
à utiliser lors d'un assemblage manuel.

### Fonctionnalités

- ESP8266 1Mb ou 4Mb
- Autoalimenté
- Veille profonde
- Suivi de tension du condensateur
- Broches UART accessibles : GND, 3V3, 5V, RX, TX, DTR, RST
(Note : la broche 3V3 **n'est pas protégée**; bien veiller à fournir cette tension exacte).
- GPIOs exposés : 12, 13, 14

Options :

- LED optionnelle pour visualiser la réception Téléinfo
(couper la piste droite du pad "Auto TIC LED", et souder le pad central avec celui de gauche).
- Emplacement QWIIC I2C
- Circuit d'alimentation USB


### Pinout

Pin     | Fonction
|:--- |:--- |
ADC     | Surveillance de la tension du condensateur
GPIO0   | Démarrage automatique en mode flash
GPIO1   | UART TX
GPIO2   | Résistance de Pull-up / tirage
GPIO3   | UART RX / Téléinfo RX
GPIO4   | I2C SDA
GPIO5   | I2C SCL
GPIO7   | Correctif optionnel deepsleep pour ESP8266 [défaillants](https://github.com/esp8266/Arduino/issues/6007)
GPIO12  | Pad libre
GPIO13  | Pad libre
GPIO14  | Gestion optionnelle de la LED TIC, pad libre
GPIO15  | Résistance de Pull-down / rappel
GPIO16  | Réveil deepsleep


### Datasheet

[![schematic][telewatt_v1.1/schematic.svg]][telewatt_v1.1/schematic.svg]


## Firmware

Le circuit est compatible avec l'extension Teleinfo de Tasmota :
[Tasmota - Teleinfo](https://tasmota.github.io/docs/Teleinfo/).

Mais aussi avec une version non officielle améliorée :
<https://github.com/NicolasBernaerts/tasmota/tree/master/teleinfo)>.

ESPHome est probablement aussi supporté.

Tous ces firmwares sont compatibles avec HomeAssistant.

### Procédure de flash

Matériel requis : interface USB vers TTL/UART.
Exemples de chipsets : CH341 CH340, CP2102, FT232RL, etc.

Installation d'`esptool` via le gestionnaire système ou Python :

    $ apt install esptool

ou :

    $ pip install esptool

Envoi du binaire (pré)compilé :

    $ esptool write_flash -fs 4MB -fm dout 0x0 tasmota-teleinfo-**.bin

Où `**` correspond au type d'ESP8266 utilisé (normalement 4M).
Toutefois pour un usage avec HomeAssistant, les options incluses dans cette version
(courbes et historique) sont inutiles; opter pour la version 1M
ou une compilation personnalisée est donc envisageable.

Les mises à jour ultérieures pourront se faire en ligne (Over The Air - OTA).


## Configuration

Voici l'essentiel des modifications pour Tasmota + HomeAssistant.

Pour plus d'informations, voir les documentations complètes des firmwares respectifs :

- https://tasmota.github.io/docs/Teleinfo/#configure-gpios-for-teleinfo
- https://github.com/NicolasBernaerts/tasmota/tree/master/teleinfo#fonctionnalit%C3%A9s


Dès le branchement, l'ESP8266 démarre en mode configuration WiFi.
Si le condensateur est chargé, quelques minutes seront octroyées
pour faire les modifications requises.

### Configuration du WiFi

Sélectionner le point d'accès initié par le module et s'y connecter avec
un ordinateur ou un téléphone.

L'adresse de la page de configuration est `http://192.168.4.1`.
Renseigner le mot de passer réseau puis laisser l'appareil redémarrer.


### Configuration du module

Se connecter à l'appareil depuis le réseau WiFi du domicile
(rechercher son  adresse IP via les pages d'administration de la box
ou via `nmap -sP 192.168.1.0/24`).

Ne pas oublier de configurer la liaison MQTT (IP, port, login, mot de passe),
la connexion réseau (une IP fixe limite le temps passé hors veille)
et les options Téléinfo (mode standard vs historique, etc.).

L'essentiel des configurations se fait dans la console Tasmota.

Pour configurer les GPIOs et le type de module simplement en une fois, voici le
[template à importer](https://tasmota.github.io/docs/Templates/#importing-templates) :

    {"NAME":"TeleWatt","GPIO":[1,1,1,5152,1,1,1,1,1,1,1,1,1,1],"FLAG":0,"BASE":18}


- Configuration du type de module :
Type de module : "Generic(18)" (i.e remplacer "Generic (0)")

- Configuration des GPIOs :

    - Config données: GPIO3 (RXD) = "TInfo RX"
    - Config deepsleep: GPIO16 = "None"

- Activer l'auto-découverte des entités HomeAssistant :

    $ hass_enable 1

- Désactiver le serveur TCP :

    $ tcp_stop

- Désactiver l'affichage des données sur la page d'accueil :

    $ websensor3 0

- Durée de mise en sommeil (le module entrera en veille 60s après cette commande) :

    $ deepsleeptime xxx   (<= ~30s, != 300s)


## Autre documentation

La démodulation Téléinfo est tirée du travail de Charles Hallard :

- [Démystifier le décodage Téléinformation](http://hallard.me/demystifier-la-teleinfo/)
- [Récupérer les données de son compteur linky](https://miniprojets.net/index.php/2019/06/28/recuperer-les-donnees-de-son-compteur-linky/)


## Licence

L'interface matérielle est publiée sous licence [Creative Commons CC BY-NC-SA 4.0][cc-by-nc-sa].

[cc-by-nc-sa]: http://creativecommons.org/licenses/by-nc-sa/4.0/
[cc-by-nc-sa-image]: https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png
[cc-by-nc-sa-shield]: https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg
