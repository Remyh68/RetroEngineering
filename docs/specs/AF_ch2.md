CHAPITRE 2 : ARCHITECTURE SYSTÈME

2.1 VUE D'ENSEMBLE
Architecture globale du système
Le système de commande est structuré autour d'un automate Siemens S7-1500 pilotant 4 sous-systèmes principaux :
┌─────────────────────────────────────────────────────────────┐
│                    AUTOMATE S7-1500                         │
│                  (CPU 1516-3 PN/DP)                         │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   OB1       │  │   OB100      │  │   OB82       │     │
│  │ Cyclique    │  │ Démarrage    │  │ Diagnostic   │     │
│  │ 100ms       │  │              │  │              │     │
│  └──────┬──────┘  └──────────────┘  └──────────────┘     │
│         │                                                  │
│         ├──► FB101/103/104 ──► AFE + DRIVES              │
│         ├──► FB111→115 ──────► WATER COOLING SYSTEM       │
│         ├──► FB122/124 ──────► EXCITATRICE               │
│         ├──► FB150/151 ──────► COMMUNICATIONS             │
│         └──► FB200/201 ──────► SUPERVISION HMI            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
    ┌────────┐    ┌─────────┐   ┌──────────┐   ┌─────────┐
    │  AFE   │    │  DRIVES │   │ EXCITER  │   │   WCS   │
    │        │    │ MOTEUR  │   │  ROTOR   │   │ REFROID.│
    └────────┘    └─────────┘   └──────────┘   └─────────┘
    Profibus       Profibus      Modbus RTU     TOR/ANA
Figure 2.1 : Architecture fonctionnelle globale
Sous-systèmes



N°
Sous-système
Fonction principale
Interface



1
AFE + Drives
Alimentation moteur + régulation vitesse
Profibus DP


2
Water Cooling System
Refroidissement équipements puissance
TOR + ANA


3
Excitatrice
Magnétisation rotor bobiné
Modbus RTU


4
Supervision
Interface opérateur + historiques
Profinet



2.2 AUTOMATE S7-1500
Configuration matérielle
CPU principale :
Référence : CPU 1516-3 PN/DP
Mémoire travail : 1 MB
Mémoire charge : 4 MB
Temps cycle OB1 : 100 ms (configurable)
Interfaces :
  - 1x Profinet (IRT)
  - 1x Profibus DP (Maître)
  - 1x Ethernet TCP/IP
Modules E/S déportés (ET200SP) :



Emplacement
Module
Type
Quantité



Slot 1-3
DI 16x24VDC
Entrées TOR
48 voies


Slot 4-5
DO 16x24VDC
Sorties TOR
32 voies


Slot 6-7
AI 8x RTD
Entrées analogiques
16 voies


Slot 8
AO 4x20mA
Sorties analogiques
4 voies


Tableau 2.1 : Configuration modules E/S
Organisation mémoire
Blocs organisationnels (OB) :
OB1   : Cyclique principal (100ms)
OB100 : Démarrage à chaud (STARTUP)
OB82  : Alarmes diagnostic
OB121 : Erreurs d'accès programmation
Zones mémoire structurées :



Zone
Utilisation
Taille



DB100
Paramètres globaux
2048 bytes


DB150-159
Communications réseaux
10 KB


DB200-210
Données process temps réel
15 KB


M0.0-M999.7
Mémentos temporaires
1000 bytes



2.3 RÉSEAUX DE COMMUNICATION
Topologie générale
┌──────────────────────────────────────────────────────────┐
│              SALLE DE CONTRÔLE                           │
│                                                          │
│  ┌──────────┐         ┌──────────┐        ┌──────────┐ │
│  │   HMI    │◄─Profinet─►│ S7-1500 │◄─Ethernet─►│ SCADA │ │
│  │ TP1500   │         │   CPU    │        │  PC   │ │
│  └──────────┘         └─────┬────┘        └──────────┘ │
│                             │                            │
└─────────────────────────────┼────────────────────────────┘
                              │
                 ┌────────────┴────────────┐
                 │                         │
            Profibus DP              Modbus RTU
            (Maître)                 (RS485)
                 │                         │
        ┌────────┴────────┐               │
        │                 │               │
    ┌───▼───┐       ┌────▼────┐      ┌───▼────┐
    │  AFE  │       │ DRIVES  │      │EXCITER │
    │ Node 2│       │ Node 3-4│      │ Addr 1 │
    └───────┘       └─────────┘      └────────┘
Figure 2.2 : Topologie réseaux de communication
Caractéristiques réseaux
Profibus DP (Process Field Bus) :
Vitesse : 1.5 Mbit/s
Topologie : Bus linéaire
Terminaisons : 2x 220Ω actives
Nodes actifs :
  - Node 2 : AFE (Active Front End)
  - Node 3 : Drive moteur principal
  - Node 4 : Drive moteur auxiliaire
Cycle bus : 10 ms
Profinet IO (Industrial Ethernet) :
Protocole : Profinet IRT (Isochronous Real-Time)
Vitesse : 100 Mbit/s Full-Duplex
Topologie : Étoile via switch managé
Devices :
  - HMI TP1500 : IP 192.168.1.10
  - CPU S7-1500 : IP 192.168.1.1
Cycle : 10 ms synchronisé
Modbus RTU (Excitatrice) :
Interface : RS485 2 fils
Vitesse : 19200 bauds, 8N1
Adresse esclave : 1
Polling : 500 ms
Timeout : 1000 ms

2.4 CARTOGRAPHIE E/S (SYNTHÉTIQUE)
Entrées TOR principales



Adresse
Mnémonique
Description
Source



%I0.0
BP_START
Bouton poussoir démarrage
Pupitre local


%I0.1
BP_STOP
Bouton poussoir arrêt
Pupitre local


%I0.2
SEL_AUTO
Sélecteur mode AUTO
Commutateur


%I0.3
SEL_MANUAL
Sélecteur mode MANUAL
Commutateur


%I1.0
DRIVE_RDY
Drive prêt (feedback)
Profibus DP


%I1.1
DRIVE_RUN
Drive en fonctionnement
Profibus DP


%I1.2
DRIVE_FAULT
Défaut drive détecté
Profibus DP


%I2.0
WCS_FLOW_OK
Débit eau nominal
Débitmètre


%I2.1
WCS_PRESS_OK
Pression eau correcte
Pressostat


%I2.2
WCS_TEMP_OK
Température eau normale
Thermostat


Tableau 2.2 : Principales entrées TOR (10/48 affichées)
Sorties TOR principales



Adresse
Mnémonique
Description
Destination



%Q0.0
CMD_DRIVE_START
Commande démarrage drive
Profibus DP


%Q0.1
CMD_DRIVE_STOP
Commande arrêt drive
Profibus DP


%Q0.2
CMD_AFE_ENABLE
Autorisation AFE
Profibus DP


%Q1.0
PUMP_WCS_START
Démarrage pompe WCS
Contacteur


%Q1.1
VALVE_WCS_OPEN
Ouverture vanne eau
Électrovanne


%Q2.0
LAMP_RUN_GREEN
Voyant marche (vert)
Pupitre


%Q2.1
LAMP_FAULT_RED
Voyant défaut (rouge)
Pupitre


Tableau 2.3 : Principales sorties TOR (7/32 affichées)
Entrées analogiques principales



Adresse
Type
Description
Plage
Unité



%IW64
4-20mA
Vitesse moteur mesurée
0-3000
rpm


%IW66
4-20mA
Courant moteur
0-500
A


%IW68
PT100
Température stator
-50 à +200
°C


%IW70
PT100
Température rotor
-50 à +200
°C


%IW72
4-20mA
Débit eau WCS
0-100
m³/h


%IW74
4-20mA
Pression eau WCS
0-10
bar


%IW76
PT100
Température eau entrée
0-100
°C


%IW78
PT100
Température eau sortie
0-100
°C


Tableau 2.4 : Principales entrées analogiques (8/16 affichées)
Sorties analogiques principales



Adresse
Type
Description
Plage
Unité



%QW64
4-20mA
Consigne vitesse moteur
0-3000
rpm


%QW66
4-20mA
Consigne excitation rotor
0-100
%


%QW68
4-20mA
Consigne vanne réglante
0-100
%


Tableau 2.5 : Sorties analogiques (3/4 affichées)

2.5 ORGANISATION LOGICIELLE (SYNTHÉTIQUE)
Arborescence par domaines fonctionnels
Programme principal (OB1 - 100ms)
│
├─── [DOMAINE 1] AFE + DRIVES (5 blocs)
│    ├─ FB101 : Gestion AFE (Active Front End)
│    ├─ FB103 : Régulation vitesse drive principal
│    ├─ FB104 : Régulation courant moteur
│    ├─ FB122 : Séquenceur démarrage
│    └─ FB124 : Gestion défauts drives
│
├─── [DOMAINE 2] WATER COOLING SYSTEM (5 blocs)
│    ├─ FB111 : Séquenceur pompes WCS
│    ├─ FB112 : Régulation température PID
│    ├─ FB113 : Gestion vannes proportionnelles
│    ├─ FB114 : Surveillance débits
│    └─ FB115 : Alarmes WCS
│
├─── [DOMAINE 3] EXCITATRICE (2 blocs + GRAFCET)
│    ├─ FB122 : Communication Modbus RTU
│    ├─ FB124 : Régulation excitation rotor
│    └─ SFC_EXCITER : GRAFCET démarrage/arrêt
│
├─── [DOMAINE 4] COMMUNICATIONS (3 blocs)
│    ├─ FB150 : Maître Profibus DP
│    ├─ FB151 : Serveur Profinet IO
│    └─ FB152 : Client Modbus RTU
│
└─── [DOMAINE 5] SUPERVISION (3 blocs)
     ├─ FB200 : Interface HMI (variables process)
     ├─ FB201 : Historisation alarmes
     └─ DB100 : Base données paramètres
Figure 2.3 : Arborescence logicielle par domaines
Matrice d'interdépendances entre blocs



Bloc émetteur
Blocs récepteurs
Type échange
Criticité



FB101 (AFE)
FB103, FB104, FB122
État AFE, tension DC Bus
⚠️ Haute


FB103 (Drive)
FB104, FB124, FB200
Vitesse, courant, état
⚠️ Haute


FB111 (WCS)
FB112, FB113, FB114
État pompes, débits
🔵 Moyenne


FB122 (Exciter)
FB124, FB200
Courant rotor, défauts
⚠️ Haute


FB150 (Profibus)
FB101, FB103, FB104
Données process cycliques
⚠️ Haute


FB200 (HMI)
Tous FB
Commandes opérateur
🔵 Moyenne


Tableau 2.6 : Principales interdépendances entre blocs fonctionnels
Légende criticité
⚠️  Haute    : Échange temps réel < 100ms, impact sécurité
🔵 Moyenne  : Échange < 1s, impact disponibilité
🟢 Faible   : Échange > 1s, impact confort opérateur

2.6 FLUX DE DONNÉES TEMPS RÉEL
Pipeline de traitement (cycle 100ms)
┌─────────────────────────────────────────────────────────┐
│  CYCLE OB1 (100 ms)                                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [0-10ms]   Lecture E/S physiques                      │
│             ├─ TOR (48 entrées)                        │
│             ├─ ANA (16 mesures 4-20mA/PT100)           │
│             └─ Profibus DP (AFE + Drives)              │
│                                                         │
│  [10-40ms]  Traitement logique                         │
│             ├─ FB101-104 : Calculs régulations         │
│             ├─ FB111-115 : Séquences WCS               │
│             ├─ FB122-124 : Gestion exciter             │
│             └─ Interlocks sécurité                     │
│                                                         │
│  [40-60ms]  Communications                             │
│             ├─ Profibus : Mise à jour consignes        │
│             ├─ Modbus RTU : Polling exciter (500ms)    │
│             └─ Profinet : Rafraîchissement HMI         │
│                                                         │
│  [60-90ms]  Écriture sorties                           │
│             ├─ TOR (32 sorties)                        │
│             ├─ ANA (4 consignes 4-20mA)                │
│             └─ Profibus DP (commandes drives)          │
│                                                         │
│  [90-100ms] Supervision & diagnostics                  │
│             ├─ FB200 : Actualisation HMI               │
│             ├─ FB201 : Archivage alarmes               │
│             └─ Watchdog cycle                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
Figure 2.4 : Découpage temporel du cycle automate
Synchronisation multi-réseaux
HORLOGE MAÎTRE : CPU S7-1500 (NTP synchronisé)
│
├─ Profibus DP   : Cycle 10 ms (synchrone CPU)
├─ Profinet IRT  : Cycle 10 ms (isochronous)
└─ Modbus RTU    : Polling 500 ms (asynchrone)

LATENCES MAXIMALES :
  CPU → Drive   : 20 ms
  Drive → CPU   : 20 ms
  CPU ↔ HMI     : 100 ms
  CPU ↔ Exciter : 1000 ms

✅ FIN CHAPITRE 2

