# Projet n°4 — Contrôle d'accès RFID multi-utilisateurs avec journal en ligne

**Étudiant :** Emilie SYLNÉ
**Projet réalisé :** Projet n°4

## Description

Version connectée et évolutive de la serrure à code : un système de contrôle d'accès basé sur un lecteur RFID, capable d'identifier plusieurs utilisateurs individuellement grâce à leur badge. Chaque tentative d'accès (autorisée ou refusée) est enregistrée en temps réel dans une base de données en ligne et consultable depuis un tableau de bord web.

## Objectifs

- Identifier chaque utilisateur individuellement par badge RFID
- Gérer une liste de badges autorisés, stockée et modifiable sur carte SD
- Afficher localement le statut d'accès (autorisé / refusé) sur un écran OLED
- Actionner un servomoteur pour simuler l'ouverture de la porte
- Envoyer chaque accès vers une base de données en ligne (Firebase), avec identité, date et heure
- Consulter l'historique complet et les tentatives refusées via un tableau de bord web

## Composants utilisés (simulation Wokwi)

| Composant | Rôle |
|---|---|
| ESP32 | Microcontrôleur principal (WiFi intégré) |
| Lecteur RFID MFRC522 | Lecture des badges |
| Servomoteur | Simulation de l'ouverture de la porte |
| Écran OLED (SSD1306, I2C) | Affichage du statut d'accès |
| Module carte SD | Stockage de la liste des badges autorisés |

## Schéma de câblage

Simulation complète disponible sur Wokwi :
🔗 **https://wokwi.com/projects/476445924677929985**

| Composant | Broche | ESP32 |
|---|---|---|
| MFRC522 | SDA | GPIO 5 |
| MFRC522 | SCK | GPIO 18 |
| MFRC522 | MOSI | GPIO 23 |
| MFRC522 | MISO | GPIO 19 |
| MFRC522 | RST | GPIO 27 |
| MFRC522 | 3.3V / GND | 3V3 / GND |
| Carte SD | SCK / DI / DO | GPIO 18 / 23 / 19 |
| Carte SD | CS | GPIO 4 |
| Carte SD | VCC / GND | 3V3 / GND |
| OLED | SDA | GPIO 21 |
| OLED | SCL | GPIO 22 |
| OLED | VCC / GND | 3V3 / GND |
| Servomoteur | Signal | GPIO 13 |
| Servomoteur | V+ / GND | VIN / GND |

*(RFID et carte SD partagent le même bus SPI, avec des broches CS distinctes.)*

## Fonctionnement

1. Un badge est présenté au lecteur RFID.
2. L'ESP32 compare son UID à la liste des badges enregistrés sur la carte SD.
3. Si le badge est reconnu :
   - L'écran OLED affiche « AUTORISÉ » avec le nom de la personne
   - Le servomoteur s'ouvre pendant 3 secondes
   - L'accès est envoyé vers Firebase avec le statut `autorise`
4. Si le badge est inconnu :
   - L'écran OLED affiche « REFUSÉ »
   - L'accès est envoyé vers Firebase avec le statut `refuse`
5. Chaque accès est horodaté grâce à une synchronisation NTP.

### Ajouter un nouveau badge

Depuis le moniteur série, taper :
