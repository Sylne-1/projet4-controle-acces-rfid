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

**Captures d'écran et simulation de la video sur Wokwi :**

*( dans le dossier `docs/`)*
## Le code source complet (fichier .ino)

/*
  Projet n°4 - Contrôle d'accès RFID multi-utilisateurs
  Partie 2 : ajout du WiFi, de l'horodatage (NTP) et de l'envoi
  de chaque accès vers Firebase Realtime Database.
*/

#include <WiFi.h>
#include <HTTPClient.h>
#include <SPI.h>
#include <SD.h>
#include <MFRC522.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <ESP32Servo.h>
#include <time.h>

// ---------- WiFi (réseau simulé par Wokwi) ----------
const char* WIFI_SSID = "Wokwi-GUEST";
const char* WIFI_PASS = "";

// ---------- Firebase ----------
// Remplace par l'URL de TA base, SANS le "/" à la fin
const char* FIREBASE_HOST = "https://controle-acces-rfid-521df-default-rtdb.firebaseio.com/";
// ---------- Broches ----------
#define RFID_SS   5
#define RFID_RST  27
#define SD_CS     4
#define SERVO_PIN 13
#define OLED_SDA  21
#define OLED_SCL  22

// ---------- Objets ----------
MFRC522 rfid(RFID_SS, RFID_RST);
Adafruit_SSD1306 oled(128, 64, &Wire, -1);
Servo servo;

const char* FICHIER_BADGES = "/badges.txt";
bool modeAjout = false;
String nomAjout = "";

// ---------- Affichage OLED ----------
void afficher(const String& l1, const String& l2 = "", const String& l3 = "") {
  oled.clearDisplay();
  oled.setTextColor(SSD1306_WHITE);
  oled.setTextSize(2);
  oled.setCursor(0, 0);
  oled.println(l1);
  oled.setTextSize(1);
  oled.setCursor(0, 30);
  oled.println(l2);
  oled.println(l3);
  oled.display();
}

// ---------- WiFi ----------
void connecterWiFi() {
  afficher("WiFi...", "Connexion en cours");
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  int tentatives = 0;
  while (WiFi.status() != WL_CONNECTED && tentatives < 20) {
    delay(500);
    Serial.print(".");
    tentatives++;
  }
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi connecte, IP : " + WiFi.localIP().toString());
  } else {
    Serial.println("\nWiFi NON connecte (le journal ne sera pas envoye)");
  }
}

// ---------- Heure (NTP) ----------
void configurerHeure() {
  configTime(3600, 0, "pool.ntp.org", "time.nist.gov");
  Serial.print("Synchronisation de l'heure");
  struct tm infoHeure;
  int tentatives = 0;
  while (!getLocalTime(&infoHeure) && tentatives < 10) {
    Serial.print(".");
    delay(500);
    tentatives++;
  }
  Serial.println();
}

String dateHeureActuelle() {
  struct tm infoHeure;
  if (!getLocalTime(&infoHeure)) return "date-inconnue";
  char buf[25];
  strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", &infoHeure);
  return String(buf);
}

// ---------- Envoi d'un accès vers Firebase ----------
void envoyerFirebase(const String& uid, const String& nom, const String& statut) {
  if (WiFi.status() != WL_CONNECTED) return;

  HTTPClient http;
  String url = String(FIREBASE_HOST) + "/acces/" + String(millis()) + ".json";
  http.begin(url);
  http.addHeader("Content-Type", "application/json");

  String json = "{";
  json += "\"uid\":\"" + uid + "\",";
  json += "\"nom\":\"" + nom + "\",";
  json += "\"statut\":\"" + statut + "\",";
  json += "\"date_heure\":\"" + dateHeureActuelle() + "\"";
  json += "}";

  int code = http.PUT(json);
  Serial.println("Envoi Firebase (" + statut + ") -> code : " + String(code));
  http.end();
}

// ---------- UID du badge lu, en texte ----------
String uidEnTexte() {
  String s = "";
  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] < 0x10) s += "0";
    s += String(rfid.uid.uidByte[i], HEX);
  }
  s.toUpperCase();
  return s;
}

// ---------- Gestion de la liste sur SD ----------
bool chercherBadge(const String& uid, String& nom) {
  File f = SD.open(FICHIER_BADGES, FILE_READ);
  if (!f) return false;
  while (f.available()) {
    String ligne = f.readStringUntil('\n');
    ligne.trim();
    int p = ligne.indexOf(';');
    if (p < 0) continue;
    if (ligne.substring(0, p) == uid) {
      nom = ligne.substring(p + 1);
      f.close();
      return true;
    }
  }
  f.close();
  return false;
}

void ajouterBadge(const String& uid, const String& nom) {
  File f = SD.open(FICHIER_BADGES, FILE_APPEND);
  if (f) {
    f.println(uid + ";" + nom);
    f.close();
  }
}

void listerBadges() {
  File f = SD.open(FICHIER_BADGES, FILE_READ);
  if (!f) {
    Serial.println("Aucun badge enregistre.");
    return;
  }
  Serial.println("--- Badges autorises ---");
  while (f.available()) Serial.write(f.read());
  f.close();
  Serial.println("------------------------");
}

// ---------- Servo ----------
void ouvrirPorte() {
  servo.write(90);
  delay(3000);
  servo.write(0);
}

// ---------- Commandes série ----------
void lireCommandes() {
  if (!Serial.available()) return;
  String cmd = Serial.readStringUntil('\n');
  cmd.trim();
  if (cmd.startsWith("ADD ")) {
    nomAjout = cmd.substring(4);
    modeAjout = true;
    Serial.println("Presentez le badge de : " + nomAjout);
    afficher("MODE AJOUT", nomAjout, "Presentez le badge");
  } else if (cmd == "LIST") {
    listerBadges();
  }
}

void setup() {
  Serial.begin(115200);

  pinMode(RFID_SS, OUTPUT);
  digitalWrite(RFID_SS, HIGH);
  pinMode(SD_CS, OUTPUT);
  digitalWrite(SD_CS, HIGH);

  Wire.begin(OLED_SDA, OLED_SCL);
  if (!oled.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("Erreur : OLED introuvable");
  }
  afficher("Demarrage", "Veuillez patienter");

  SPI.begin(18, 19, 23);

  if (!SD.begin(SD_CS)) {
    Serial.println("Erreur : carte SD introuvable");
    afficher("ERREUR", "Carte SD absente");
    while (true) delay(1000);
  }

  rfid.PCD_Init();

  servo.attach(SERVO_PIN, 500, 2400);
  servo.write(0);

  connecterWiFi();
  if (WiFi.status() == WL_CONNECTED) configurerHeure();

  Serial.println("Systeme pret.");
  Serial.println("Commandes : ADD Nom  |  LIST");
  afficher("Pret", "Presentez un badge");
}

void loop() {
  lireCommandes();

  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return;

  String uid = uidEnTexte();
  Serial.println("Badge lu : " + uid);

  if (modeAjout) {
    ajouterBadge(uid, nomAjout);
    Serial.println("Badge ajoute : " + nomAjout);
    afficher("BADGE AJOUTE", nomAjout);
    modeAjout = false;
    delay(2000);
  } else {
    String nom;
    if (chercherBadge(uid, nom)) {
      Serial.println("ACCES AUTORISE : " + nom);
      afficher("AUTORISE", nom);
      envoyerFirebase(uid, nom, "autorise");
      ouvrirPorte();
    } else {
      Serial.println("ACCES REFUSE : " + uid);
      afficher("REFUSE", "UID inconnu", uid);
      envoyerFirebase(uid, "inconnu", "refuse");
      delay(2000);
    }
  }

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
  afficher("Pret", "Presentez un badge");
}

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

- **Risque de talonnage (*tailgating*) :** La porte reste ouverte pendant une durée fixe (3 secondes) après la validation d'un badge. Cela permet à une deuxième personne de s'infiltrer à la suite d'un utilisateur autorisé sans présenter de badge.
- **Dépendance à la connexion réseau :** Le transfert des données vers Firebase nécessite un réseau Wi-Fi stable. En cas de coupure, les tentatives d'accès ne sont pas transmises au tableau de bord.
- **Sécurité des badges (UID) :** Le lecteur MFRC522 utilise uniquement l'identifiant unique (UID) du badge, ce qui rend les cartes vulnérables au clonage par un équipement tiers.
- **Dépendance à la carte SD :** Si la carte SD est absente ou défectueuse, le système ne peut plus vérifier la liste d'autorisations locale.

## Améliorations futures

- **Système anti-talonnage par capteurs de présence :** Ajouter deux capteurs de présence (infrarouges ou ultrasons HC-SR04) pour mesurer le passage individuel :
  - Fermeture immédiate du servomoteur dès que la première personne a franchi la porte, sans attendre la fin des 3 secondes.
  - Déclenchement d'une alerte sonore/visuelle et enregistrement d'une tentative de fraude dans Firebase si une deuxième personne tente d'entrer.
- **Gestion du mode hors-ligne :** Enregistrer l'historique des accès sur la carte SD en cas de perte de connexion Wi-Fi, puis le synchroniser automatiquement dès le retour du réseau.
- **Sécurisation des badges :** Implémenter des cartes chiffrées (type MIFARE DESFire) ou ajouter un second facteur d'authentification (code PIN sur clavier).






