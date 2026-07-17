# RFID Access Control System

> **Zweifaktor-Zutrittskontrollsystem auf Basis eines Raspberry Pi 5**  
> Two-factor access control system based on Raspberry Pi 5

Entwickelt im Rahmen eines Kursprojekts · **Gruppe 02 · FH Ostfalia**

![Python](https://img.shields.io/badge/Python-3.x-blue?logo=python&logoColor=white)
![SQLite](https://img.shields.io/badge/Database-SQLite-lightblue?logo=sqlite)
![Platform](https://img.shields.io/badge/Platform-Raspberry%20Pi%205-red?logo=raspberrypi)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Inhaltsverzeichnis

- [Projektbeschreibung](#-projektbeschreibung)
- [Features](#-features)
- [Hardware](#-hardware)
- [Projektstruktur](#-projektstruktur)
- [Installation](#-installation)
- [Verwendung](#-verwendung)
- [Datenbankschema](#-datenbankschema)
- [Authentifizierungsablauf](#-authentifizierungsablauf)
- [Admin-Modus](#-admin-modus)
- [Bekannte Hinweise](#-bekannte-hinweise)
- [Autoren](#-autoren)

---

##  Projektbeschreibung

Dieses Projekt implementiert ein **zweistufiges Zutrittskontrollsystem** auf einem Raspberry Pi 5. Die Authentifizierung erfolgt durch die Kombination einer **RFID-Karte** (Faktor 1) und einer **vierstelligen PIN** (Faktor 2). Bei erfolgreicher Verifikation wird ein Servomotor angesteuert, der symbolisch eine Tür öffnet.

Alle Zugriffsversuche (gewährt oder verweigert) werden mit Zeitstempel in einer **SQLite-Datenbank** protokolliert. Administratoren erhalten nach der Anmeldung Zugriff auf ein **Verwaltungsmenü** zur Benutzerpflege.

---

##  Features

| Feature | Beschreibung |
|---|---|
| RFID-Erkennung | Kartenauslese via RC522 über SPI |
| PIN-Eingabe | 4-stellige PIN über 4×4-Tastenfeld |
| Benutzerverwaltung | SQLite-Datenbank mit Benutzer- & Log-Tabellen |
| Türsteuerung | Servo öffnet/schließt bei Authentifizierung |
| LCD-Anzeige | Echtzeit-Statusanzeige mit Uhrzeit (I2C) |
| LED-Feedback | Grün = Zugang gewährt · Rot = Zugang verweigert |
| Buzzer | Akustisches Feedback bei Erfolg und Misserfolg |
| Admin-Modus | Benutzerverwaltung direkt über Keypad & LCD |
| Zugriffslog | Protokollierung aller Ereignisse mit Zeitstempel |

---

## 🛠️ Hardware

| Komponente | Pin / Schnittstelle |
|---|---|
| RC522 RFID-Leser | SPI (GPIO 11, 10, 9, 8) |
| 4×4 Tastenfeld | GPIO 5, 6, 13, 19 (Reihen) · 26, 16, 20, 21 (Spalten) |
| Servomotor | GPIO 18 (PWM) |
| Grüne LED | GPIO 24 |
| Rote LED | GPIO 23 |
| Passiver Buzzer | GPIO 12 (PWM via lgpio) |
| LCD-Display (16×2) | I2C · Adresse 0x27 (PCF8574) |

### Schaltungsübersicht

```
Raspberry Pi 5
├── SPI  ──────────── RC522 RFID Reader
├── I2C  ──────────── LCD 16x2 (PCF8574)
├── GPIO 18 ────────── Servo Motor
├── GPIO 24 ────────── LED (grün)
├── GPIO 23 ────────── LED (rot)
├── GPIO 12 ────────── Buzzer (passiv, PWM)
└── GPIO 5,6,13,19 ─── Keypad Zeilen
    GPIO 26,16,20,21 ── Keypad Spalten
```

---

## Projektstruktur

```
controle_acces_rfid/
│
├── access.db            ← SQLite-Datenbank (Benutzer + Logs)
├── venv/                ← Virtuelles Python-Environment
│
└── modules/
    ├── main.py          ← Hauptprogramm & Systemsteuerung
    ├── database.py      ← Datenbankzugriff (CRUD-Operationen)
    ├── init_db.py       ← DB-Initialisierung & Testdaten
    ├── rfid_reader.py   ← RFID-Kartenauslese (UID)
    ├── MFRC522.py       ← RC522 Hardware-Treiber (lgpio/SPI)
    ├── SimpleMFRC522.py ← Vereinfachte RFID-Schnittstelle
    ├── keypad.py        ← 4×4 Tastenfeld-Steuerung
    ├── servo.py         ← Servo, LEDs & Buzzer
    ├── lcd.py           ← LCD-Anzeige (I2C)
    └── admin.py         ← CLI-Administrationstool (Terminal)
```

---

## ⚙️ Installation

### 1. Repository klonen

```bash
git clone https://github.com/id650236/controle_acces_rfid.git
cd controle_acces_rfid
```

### 2. Virtuelles Environment erstellen & aktivieren

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Abhängigkeiten installieren

```bash
pip install lgpio spidev gpiozero RPLCD
```

### 4. SPI aktivieren (einmalig)

```bash
sudo raspi-config
# → Interface Options → SPI → Enable
```

### 5. Datenbank initialisieren

```bash
python3 modules/init_db.py
```

---

## Verwendung

### System starten

```bash
python3 modules/main.py
```

### Admin-Tool (Terminal)

```bash
python3 modules/admin.py
```

### Ablauf

```
1. LCD zeigt "Karte auflegen" + aktuelle Uhrzeit
2. RFID-Karte an den Leser halten
3. Vierstellige PIN über Tastenfeld eingeben
4. System prüft UID + PIN in der Datenbank
   ✅ Korrekt  → grüne LED, Servo öffnet Tür, Log: GRANTED
   ❌ Falsch   → rote LED, 3× Buzzer, Log: DENIED
5. Admins erhalten Zugriff auf das Admin-Menü
```

---

##Datenbankschema

```sql
-- Registrierte Benutzer
CREATE TABLE users (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    uid_rfid  TEXT UNIQUE NOT NULL,   -- UID der RFID-Karte
    pin_code  TEXT NOT NULL,           -- 4-stellige PIN
    name      TEXT NOT NULL,           -- Anzeigename
    is_admin  INTEGER DEFAULT 0        -- 1 = Administrator
);

-- Zugriffsprotokoll
CREATE TABLE logs (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    uid_rfid  TEXT NOT NULL,           -- Karte, die verwendet wurde
    action    TEXT NOT NULL,           -- "GRANTED" oder "DENIED"
    timestamp TEXT DEFAULT CURRENT_TIMESTAMP
);
```

### Vordefinierte Testbenutzer

| Name | UID | PIN | Admin |
|---|---|---|---|
| Gruppe02 | 58184332 | 1234 | Nein |
| Yvan Kameni | 1458566131 | 5678 | Nein |
| Arnaud Fotsa | 28136183115 | 2035 | **Ja** |
| Armand Fotso | 18221222163 | 2030 | Nein |

---

## Authentifizierungsablauf

```
┌─────────────────────────────────────────────┐
│              System bereit                  │
│         LCD: "Karte auflegen"               │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
         RFID-Karte erkannt
         UID wird ausgelesen
                   │
                   ▼
         PIN-Eingabe (4 Ziffern)
         über 4×4 Keypad
                   │
                   ▼
         ┌─────────────────┐
         │ UID + PIN in DB │
         │    gefunden?    │
         └────────┬────────┘
         JA ──────┤──────── NEIN
         │                  │
         ▼                  ▼
   ✅ GRANTED          ❌ DENIED
   Grüne LED           Rote LED
   Servo öffnet        3× Buzzer
   Log: GRANTED        Log: DENIED
         │
         ▼
   is_admin == 1?
   → Admin-Menü
```

---

## Admin-Modus

Nach erfolgreicher Anmeldung eines Administrators öffnet sich ein Menü über LCD + Keypad:

| Taste | Aktion |
|---|---|
| `1` | Alle Benutzer anzeigen |
| `2` | Neuen Benutzer per RFID + PIN hinzufügen |
| `3` | Benutzer löschen |
| `4` | Letzte 5 Log-Einträge anzeigen |
| `#` | Admin-Menü verlassen |

Das Terminal-basierte Admin-Tool (`admin.py`) bietet dieselben Funktionen zusätzlich über die Kommandozeile.

---

## Bekannte Hinweise

> **RPi.GPIO wird auf dem Raspberry Pi 5 nicht unterstützt.**  
> Dieses Projekt verwendet `lgpio` als vollständigen Ersatz für alle GPIO-Operationen, einschließlich SPI-Reset-Pin, Keypad-Scanning und PWM für den Buzzer.

```bash
# lgpio statt RPi.GPIO
pip install lgpio
```

---

## Autoren

| Name | Rolle |
|---|---|
| **Arnaud Fotsa** | Entwicklung, Hardware-Integration, Dokumentation |
| **Armand Homsi Fotso** | Entwicklung, Testing |
| **Kameni Beaudexe Yvan** | Entwicklung, Testing |

**Betreuer:** Prof. Dr. Jürgen Kreyssig   
**Institution:** Fachhochschule Ostfalia · Masterstudiengang Informatik   
**Gruppe:** 02

---

## Lizenz

Dieses Projekt ist für akademische Zwecke erstellt.  
MFRC522-Treiber basiert auf der Open-Source-Bibliothek von [pimylifeup](https://github.com/pimylifeup/MFRC522-python).
