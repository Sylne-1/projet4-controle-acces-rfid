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
puis présenter le badge au lecteur. Il est alors enregistré sur la carte SD.

### Lister les badges enregistrés

Taper `LIST` dans le moniteur série.

## Tableau de bord web

Le fichier `dashboard/index.html` affiche en temps réel :
- Le nombre total d'accès, autorisés et refusés
- Un tableau complet de tous les accès (nom, UID, statut, date/heure)
- Un onglet dédié aux tentatives d'accès refusées

Il se connecte directement à la base Firebase Realtime Database et s'actualise automatiquement toutes les 5 secondes.

**Captures d'écran :**

*( dans le dossier `docs/`)*

## Installation et utilisation

### 1. Simulation

1. Ouvrir le projet sur Wokwi : https://wokwi.com/projects/476445924677929985
2. Dans l'onglet **Library Manager**, vérifier que les bibliothèques suivantes sont installées :
   - MFRC522
   - Adafruit SSD1306
   - Adafruit GFX Library
   - ESP32Servo
3. Lancer la simulation avec le bouton **▶ Play**.

### 2. Base de données Firebase

1. Créer un projet sur [Firebase Console](https://console.firebase.google.com)
2. Activer **Realtime Database** en mode test
3. Copier l'URL de la base et la renseigner dans la variable `FIREBASE_HOST` du fichier `src/controle_acces.ino` et de `dashboard/index.html`

### 3. Tableau de bord

Ouvrir `dashboard/index.html` dans un navigateur, ou héberger le dossier `dashboard/` via **GitHub Pages** (Settings → Pages → source : dossier `/dashboard`).

## Limites du projet

- **Dépendance à la connexion réseau :** Le transfert des données d'accès vers Firebase nécessite une connexion Wi-Fi stable. En cas de coupure réseau, les logs instantanés ne sont pas enregistrés en ligne.
- **Sécurité RFID (UID) :** Les cartes/badges RFID 13.56 MHz standards (MFRC522) utilisent une identification basée sur l'UID, ce qui les rend vulnérables au clonage par un équipement spécialisé.
- **Stockage de la carte SD :** Si la carte SD est retirée ou endommagée, le système ne peut plus vérifier la liste d'autorisations locale.
- **Capacité de la mémoire de simulation :** Dans l'environnement de simulation Wokwi, la gestion simultanée des bibliothèques Wi-Fi, SPI et OLED peut approcher les limites de mémoire et de temps de réponse du simulateur.

## Améliorations futures

- **Gestion du mode hors-ligne (Offline Storage) :** Sauvegarder les logs d'accès localement sur la carte SD en cas de perte Wi-Fi, puis les synchroniser automatiquement dès le rétablissement de la connexion avec Firebase.
- **Sécurisation renforcée :** Implémenter des cartes avec chiffrement (ex: MIFARE DESFire) ou ajouter un second facteur d'authentification (ex: code PIN sur clavier à membrane ou reconnaissance biométrique).
- **Interface de gestion d'utilisateurs via le Dashboard :** Permettre l'ajout, la modification ou la révocation de badges directement depuis le tableau de bord web sans passer par la console série.
- **Notifications instantanées :** Intégrer un service de notifications (ex: Pushbullet, Telegram ou e-mail) lors d'une tentative d'accès non autorisée répétée.
- **Chiffrement des données :** Sécuriser les échanges entre l'ESP32 et Firebase à l'aide de jetons d'authentification et de liaisons HTTPS/TLS.


