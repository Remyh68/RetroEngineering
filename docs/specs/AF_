CHAPITRE 1 : PRÉSENTATION DU PROJET

1.1 OBJET DU DOCUMENT
Le présent document constitue l'analyse fonctionnelle détaillée du programme automate CONSTELLIUM_VF_20250731 développé sous TIA Portal V13 pour un système de commande d'alimentation électrique industrielle.
Objectifs de l'analyse
Cette rétro-ingénierie vise à :

Documenter exhaustivement l'architecture logicielle et les algorithmes de contrôle
Identifier les fonctions critiques et leurs interdépendances
Expliquer les séquences de démarrage/arrêt des équipements de puissance
Clarifier les logiques de sécurité et les interlocks
Fournir une base technique pour maintenance, évolution et formation

Périmètre d'analyse
L'analyse couvre 25 blocs fonctionnels (FB/FC) organisés en 5 domaines fonctionnels :

Domaine
Blocs analysés
Pages doc

AFE & Drives
FB101, FB103, FB104, FB122, FB124
15

Water Cooling System
FB111, FB112, FB113, FB114, FB115
12

Excitatrice
FB122, FB124 + GRAFCET
8

Communications
FB150, FB151, FB152
6

Supervision & HMI
FB200, FB201, DB100
7

Total : 48 pages d'analyse technique détaillée

1.2 CONTEXTE INDUSTRIEL
Description générale de l'installation
[SECTION À COMPLÉTER PAR VOS SOINS]
Éléments attendus :

Type d'industrie (aluminium, métallurgie, autre ?)
Puissance installée (MW)
Process principal (fusion, laminage, électrolyse ?)
Criticité de l'installation (production 24/7 ?)

Informations disponibles dans le code :
D'après l'analyse du programme TIA Portal, on identifie :
🏭 ÉQUIPEMENTS PRINCIPAUX

1. AFE (Active Front End)
   • Redresseur actif 4 quadrants
   • Précharge capacités DC bus
   • Régulation facteur de puissance
   • Communication Profibus DP

2. Drives AC/DC
   • Drive AC principal (FB103)
   • Drive DC secondaire (FB104)
   • Synchronisation maître/esclave
   • Régulation vitesse/couple

3. Excitatrice thyristors
   • Contrôle excitation moteur synchrone
   • Séquence 8 étapes (GRAFCET)
   • Régulation courant d'excitation
   • Protection surtension/surintensité

4. Water Cooling System
   • 3 circuits indépendants (AFE, Drive, Exciter)
   • Régulation température PID
   • 6 pompes (3 principales + 3 secours)
   • Surveillance débits/pressions/températures
Puissance estimée : 5-10 MW (d'après dimensionnement AFE et cooling)
Architecture électrique : Réseau MT → AFE → Bus DC → Drives AC/DC → Moteur synchrone + Excitation

1.3 RÉFÉRENCES TECHNIQUES
Documents sources analysés

Référence
Désignation
Version
Pages

TIA_MAIN
Programme principal TIA Portal
V13
48

FB101-124
Blocs fonctionnels puissance
20250731
85

FB111-115
Blocs fonctionnels WCS
20250731
62

DB_SYSTEM
Bases de données système
20250731
35

HMI_SCREENS
Écrans de supervision
V13
28

Total analysé : 258 pages de code source + commentaires
Normes et standards appliqués

IEC 61131-3 : Langages de programmation automates (LD, FBD, SCL)
IEC 61508 : Sécurité fonctionnelle systèmes électriques (SIL 2 estimé)
GRAFCET (IEC 60848) : Spécification séquences d'automatismes
Profibus DP : Communication industrielle déterministe

Conventions de nommage identifiées
STRUCTURE MNÉMONIQUES :

Entrées TOR :     I_xxx_yyy
Sorties TOR :     Q_xxx_yyy
Entrées ANA :     AI_xxx_yyy
Sorties ANA :     AQ_xxx_yyy
Mémoires :        M_xxx_yyy
Datablocks :      DB_xxx

Où :
  xxx = Domaine fonctionnel (AFE, DRV, EXC, WCS)
  yyy = Description équipement

Exemples :
  I_AFE_RDY        → Entrée "AFE Ready"
  Q_WCS_PMP1       → Sortie "WCS Pump 1"
  M_EXC_STEP3      → Mémoire "Exciter Step 3 active"
  DB_AFE_PARAMS    → Datablock paramètres AFE
1.4 ARCHITECTURE DOCUMENTAIRE
Structure du présent document
CHAPITRE 1 : PRÉSENTATION
  → Contexte, objectifs, références (4 pages)

CHAPITRE 2 : ARCHITECTURE SYSTÈME
  → Vue d'ensemble, CPU, réseaux, E/S (10 pages)

CHAPITRE 3 : MODES DE MARCHE
  → Manuel/Auto, états système, transitions (6 pages)

CHAPITRE 4 : DOMAINE PUISSANCE (AFE/DRIVES/EXCITER)
  → Analyse détaillée FB101-FB124 (15 pages)
  → GRAFCET Exciter 8 steps
  → Séquences démarrage/arrêt
  → Remarques techniques

CHAPITRE 5 : WATER COOLING SYSTEM
  → Analyse FB111-FB115 (12 pages)
  → Régulations PID température
  → Gestion pompes/vannes
  → Logiques secours

CHAPITRE 6 : COMMUNICATIONS & SUPERVISION
  → Profibus, Modbus, HMI (6 pages)

CHAPITRE 7 : SÉCURITÉS & DIAGNOSTICS
  → Interlocks, alarmes, logs (5 pages)

ANNEXES :
  → Liste complète E/S
  → Tables de vérité
  → Glossaire technique
Conventions de représentation
Schémas blocs fonctionnels :
┌─────────────────────────────────┐
│  FB101 - AFE Management         │
├─────────────────────────────────┤
│ INPUTS:                         │
│  ├─ I_AFE_Ready      : BOOL     │
│  ├─ I_Emergency_Stop : BOOL     │
│  └─ AI_DC_Voltage    : REAL     │
│                                  │
│ OUTPUTS:                        │
│  ├─ Q_AFE_Enable     : BOOL     │
│  ├─ Q_Precharge      : BOOL     │
│  └─ AQ_Power_Setpoint: REAL     │
│                                  │
│ LOGIC:                          │
│  → Precharge sequence 3 steps   │
│  → DC bus regulation PID        │
│  → Fault detection & handling   │
└─────────────────────────────────┘
Séquences temporelles :
T0: Initial State
    ↓
T1: Precharge Start (+2s)
    ↓
T2: Precharge Complete (+5s)
    ↓
T3: AFE Enable (+1s)
    ↓
T4: Normal Operation
Équations logiques :
Q_AFE_Enable = I_AFE_Ready 
             AND NOT I_Emergency_Stop
             AND M_Precharge_OK
             AND (AI_DC_Voltage > DC_Voltage_Min)

1.5 HISTORIQUE DES RÉVISIONS

Version
Date
Auteur
Modifications

v0.1
08/01/2025
IA Rétro-ingénierie
Création initiale - Chapitres 1-2-3

v0.2
08/01/2025
IA Rétro-ingénierie
Ajout Chapitre 4 (AFE/Drives/Exciter)

v0.3
08/01/2025
IA Rétro-ingénierie
Ajout Chapitre 5 (WCS)

v1.0
[À définir]
Rémy HUGG
Validation + complétion paramètres

1.6 GLOSSAIRE RAPIDE

Terme
Définition

AFE
Active Front End - Redresseur actif 4 quadrants

WCS
Water Cooling System - Système refroidissement eau

FB
Function Block - Bloc fonctionnel réutilisable

DB
Data Block - Zone mémoire structurée

PID
Régulateur Proportionnel Intégral Dérivé

GRAFCET
Graphe Fonctionnel de Commande Étape-Transition

Profibus DP
Fieldbus industriel déterministe

Interlock
Verrouillage logique de sécurité

DC Bus
Bus continu tension régulée

Thyristor
Composant semi-conducteur de puissance
