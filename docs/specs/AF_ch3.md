CHAPITRE 3 - MODES DE MARCHE ET SÉQUENCES

3.1 AUTOMATE D'ÉTATS PRINCIPAL
Le système RETRO_TIA implémente un automate d'états à 5 modes principaux, géré par le bloc fonction FB130 "State_Machine_Main".
Vue d'ensemble des états
Architecture globale :
┌─────────────────────────────────────────────────┐
│           AUTOMATE D'ÉTATS PRINCIPAL            │
├─────────────────────────────────────────────────┤
│                                                 │
│  [INIT] → [STOP] → [READY] → [RUN]            │
│              ↑         ↑         ↓              │
│              └─────[FAULT]───────┘              │
│                                                 │
└─────────────────────────────────────────────────┘
Tableau des états et transitions



État actuel
Code
Transition
Condition principale
État suivant
Durée typique



INIT
0
T1
POST complet OK
STOP
5-10 s


STOP
1
T2
Acquittement opérateur + Safety OK
READY
Manuel


STOP
1
T9
Défaut critique résolu
STOP
Variable


READY
2
T3
12 conditions RUN validées
RUN
2-5 s


READY
2
T7
Arrêt demandé opérateur
STOP
Immédiat


RUN
3
T4
Défaut niveau 2 ou 3
FAULT
< 100 ms


RUN
3
T5
Arrêt normal demandé
READY
10-30 s


FAULT
4
T6
Défaut niveau 3 (critique)
STOP
< 50 ms


FAULT
4
T8
Défaut niveau 2 acquitté
READY
Manuel


Variables d'état système
Mémorisées dans DB130 "State_Machine_DB" :



Variable
Type
Fonction



Current_State
INT
État actuel (0-4)


Previous_State
INT
État précédent (historique)


State_Timer
TIME
Temps écoulé dans état actuel


Transition_Request
BOOL
Demande transition active


Transition_ID
INT
Identifiant transition (T1-T9)


Fault_Level
INT
Niveau défaut (0=OK, 1-3=Alarm)



3.2 MODE INITIALISATION (INIT)
Durée nominale : 5 à 10 secondesDéclenchement : Mise sous tension CPU ou restart après défaut majeur
Séquence POST (Power-On Self Test)
Phase 1 [0-2s] : Vérifications matérielles
Tests effectués par FC130 "POST_Hardware" :

✅ Modules E/S : Lecture diagnostics Profinet (32 DI + 32 DO + 8 AI + 4 AO)
✅ Bus Profibus : Scan 4 esclaves (AFE + 3 Drives) – timeout 500ms/esclave
✅ Liaison Modbus : Test communication Exciter (requête echo) – timeout 1s
✅ Mémoire CPU : Test intégrité DB1-DB250 (checksum)

Critères validation Phase 1 :

Tous modules répondent (pas de "X" rouge TIA Portal)
4/4 esclaves Profibus présents
Exciter répond au test echo
Aucune erreur mémoire détectée

Si échec Phase 1 : Passage immédiat en FAULT (défaut niveau 3 "Hardware Failure")

Phase 2 [2-5s] : Chargement paramètres
Lecture DB depuis carte SD ou mémoire rémanente :



DB chargé
Contenu
Taille
Criticité



DB10
Paramètres moteur (Pnom, Nnom, cos φ)
128 bytes
Critique


DB20
Paramètres régulations PID (Kp, Ki, Kd)
256 bytes
Critique


DB30
Seuils alarmes (températures, débits)
64 bytes
Importante


DB40
Recettes (8 profils vitesse prédéfinis)
512 bytes
Optionnel


Validation paramètres :

Vérification plages admissibles (ex: Pnom entre 100-5000 kW)
Détection valeurs incohérentes (Kp < 0 impossible)
Application valeurs par défaut si DB corrompu


Phase 3 [5-8s] : Initialisation sous-systèmes
Séquence d'initialisation ordonnée :

Water Cooling System (WCS) :

Ouverture vannes drain (purge air)
Démarrage pompe principale 30%
Attente établissement débit (10 L/min mini)


Active Front End (AFE) :

Envoi paramètres via Profibus (DB10 → AFE)
Activation pré-charge condensateurs (120s)
Validation tension DC Bus (650 VDC ±5%)


Drives moteur :

Chargement profils rampes (DB40)
Reset défauts antérieurs (commande 0x80)
Passage mode "Ready to operate"


Excitatrice :

Initialisation canal Modbus (adresse 1, 19200 bauds)
Lecture version firmware exciter
Consigne flux rotor à 0% (mode standby)




Phase 4 [8-10s] : Tests capteurs critiques
Validation cohérence mesures :



Capteur
Test
Valeur attendue
Action si échec



T_AFE (AI0)
Lecture température
15-35°C (ambiante)
Alarme niveau 2


T_Motor (AI1)
Lecture température
15-40°C (ambiante)
Alarme niveau 2


Flow_WCS (AI4)
Débit mesuré
> 10 L/min
Alarme niveau 3


Press_WCS (AI5)
Pression circuit
2-4 bars
Alarme niveau 2


Validation finale POST :

✅ Tous tests Phase 1-4 validés
✅ Aucun défaut niveau 3 actif
✅ Compteur erreurs < 5 (tolérance glitchs)

Résultat : Transition T1 vers STOP (état sécurisé)

3.3 MODE STOP (Arrêt sécurisé)
Fonction : État sûr maintenu tant que conditions redémarrage non validées
Caractéristiques mode STOP
Sorties système forcées :
Sorties TOR (contacteurs/vannes) :

Q0.3 Contactor_AFE = 0 (ligne ouverte)
Q0.4 Exciter_Enable = 0 (excitation coupée)
Q1.0-1.3 Drive_Enable[1..4] = 0 (drives désarmés)
Q0.0 Pump_WCS_Main = 1 (maintien circulation mini)
Q0.2 Valve_Inlet_WCS = 1 (circuit ouvert refroidissement)

Sorties analogiques (consignes) :

AQ0 Speed_Setpoint = 0.0 (consigne vitesse nulle)
AQ1 Flux_Setpoint = 0.0 (flux rotor nul)
AQ2 WCS_Flow_Setpoint = 30% (débit mini maintenu)

Communications actives :

✅ Profibus DP : Maintien liaison (lecture états drives)
✅ Profinet IRT : Rafraîchissement HMI (affichage états)
❌ Modbus RTU : Polling suspendu (exciter en veille)


Conditions maintien en STOP
Verrouillages actifs (validation FB131) :
Sécurités matérielles (8 conditions) :

⚠️ Arrêt urgence : Au moins 1 circuit AU actif (I0.0 OU I0.1 = 0)
⚠️ Défaut alimentation : Tension réseau < 380V ou > 420V
⚠️ Surchauffe AFE : Température > 80°C (I0.2 = 1)
⚠️ Surchauffe moteur : Température > 120°C (I0.3 = 1)
⚠️ Défaut WCS : Débit < 50 L/min OU Pression < 1.5 bars
⚠️ Défaut drives : Au moins 1 drive signale "Fault" (I1.1-1.4)
⚠️ Défaut Profibus : Communication perdue > 3s
⚠️ Portes ouvertes : Capteurs accès armoires électriques (option)

Conditions logicielles (4 conditions) :

⚠️ Mode forcé STOP : Variable HMI Force_Stop_Mode = TRUE
⚠️ Maintenance active : Clé "Maintenance" tournée (I5.0 = 1)
⚠️ Défaut paramètres : DB10/DB20 invalides (checksum KO)
⚠️ Watchdog cycle : Temps cycle OB1 > 150ms (surcharge CPU)


Sortie du mode STOP (Transition T2)
Procédure réarmement manuelle :
Étape 1 : Acquittement opérateur

Action HMI : Bouton "Acknowledge Faults" pressé
Ou entrée TOR : I4.0 "Reset_Button_Local" = 1 pendant 2s

Étape 2 : Vérification disparition défauts

Toutes les 12 conditions ci-dessus = OK
Historique alarmes archivé (DB201)
Compteur défauts < 10 en 1h (anti-oscillations)

Étape 3 : Confirmation transition

Validation par FB130 : Transition_Request = TRUE
Passage en READY (état préparation démarrage)
Journalisation événement (timestamp + user ID)

Blocage transition si :

Défaut niveau 3 toujours actif
Compteur réarmements > 5 en 10 min (nécessite intervention maintenance)
Mode "Maintenance forcée" actif


3.4 MODE READY (Prêt au démarrage)
Fonction : Validation 12 conditions autorisant passage en RUN
Checklist pré-démarrage (FB131)
12 conditions obligatoires vérifiées cycliquement (100ms) :
Groupe A : Sécurités électriques (4 conditions)



#
Condition
Signal vérifié
Seuil
Action si KO



1
Arrêts urgence
I0.0 ET I0.1
= 1 (relâchés)
Blocage + alarme


2
Tension réseau
AI6 (Vmeas)
380-420 V
Blocage + alarme


3
DC Bus AFE
Profibus AFE.DC_Voltage
630-670 VDC
Blocage + alarme


4
Isolation moteur
Test mégohmmètre (optionnel)
> 1 MΩ
Warning niveau 1


Groupe B : Refroidissement (3 conditions)



#
Condition
Signal vérifié
Seuil
Action si KO



5
Débit WCS
AI4 (Flow_WCS)
> 50 L/min
Blocage + alarme


6
Pression WCS
AI5 (Press_WCS)
2-4 bars
Blocage + alarme


7
Températures OK
AI0 < 70°C ET AI1 < 100°C
T_AFE et T_Motor
Blocage + alarme


Groupe C : Drives et excitation (3 conditions)



#
Condition
Signal vérifié
Seuil
Action si KO



8
Drives prêts
Profibus Drive[1..4].Ready
= TRUE (tous)
Blocage + alarme


9
Drives sans défaut
I1.1-1.4 (Fault flags)
= 0 (aucun)
Blocage + alarme


10
Exciter réponse
Modbus Exciter timeout
< 1000 ms
Warning niveau 1


Groupe D : Autorisation opérateur (2 conditions)



#
Condition
Signal vérifié
Seuil
Action si KO



11
Sélecteur local/distant
I4.1 (Mode_Selector)
= 1 (Auto)
Blocage


12
Commande START
HMI Button "Start"
= TRUE
Attente



Indicateurs visuels état READY
LED panneau opérateur local :

🟢 LED verte "READY" : Clignotante 1 Hz
🔴 LED rouge "FAULT" : Éteinte
🟡 LED jaune "WARNING" : Si conditions 4 ou 10 KO (non bloquantes)

Affichage HMI page principale :
┌─────────────────────────────────────────┐
│  ÉTAT SYSTÈME : READY                   │
├─────────────────────────────────────────┤
│  ✅ Sécurités électriques      [4/4]    │
│  ✅ Système refroidissement    [3/3]    │
│  ✅ Drives moteur              [3/3]    │
│  ⏳ Attente commande START     [1/2]    │
│                                         │
│  [DÉMARRER]  [RETOUR STOP]             │
└─────────────────────────────────────────┘

Transitions possibles depuis READY
Transition T3 → RUN (Démarrage) :

Condition : Les 12 conditions validées simultanément pendant 2s
Action : Lancement séquence démarrage moteur (voir section 3.5)
Délai : 2-5 secondes selon vitesse cible

Transition T7 → STOP (Abandon) :

Déclenchement : Bouton "Cancel/Stop" HMI ou entrée I4.2
Action : Retour état sûr sans séquence arrêt
Délai : Immédiat (< 100ms)

Transition T4 → FAULT (Défaut survenu) :

Déclenchement : Une condition groupes A/B/C devient KO
Action : Sauvegarde contexte + alarme + passage FAULT
Délai : Réaction < 100ms


3.5 MODE RUN (Fonctionnement normal)
Fonction : Exploitation normale avec régulations actives
Séquence démarrage moteur (Transition T3)
6 étapes successives orchestrées par FC135 "Motor_Startup" :
Étape 1 : Pré-magnétisation [0-5s]
Actions :

Activation exciter : Consigne flux rotor = 50% (Modbus AQ1)
Attente établissement flux : 3-5 secondes
Validation courant d'excitation : I_exc > 2 A (lecture Modbus)

Critère passage étape 2 :

Flux rotor stabilisé à ±2% consigne pendant 1s


Étape 2 : Fermeture contacteur AFE [5-6s]
Actions :

Activation sortie Q0.3 "Contactor_AFE" = 1
Attente retour contact auxiliaire : I2.0 = 1 (confirmé fermé)
Surveillance tension DC Bus : maintien 650 VDC ±5%

Critère passage étape 3 :

Contacteur confirmé fermé + DC Bus stable 500ms


Étape 3 : Armement drives [6-7s]
Actions :

Activation Q1.0-1.3 "Drive_Enable[1..4]" = 1
Envoi commande Profibus "Enable" (mot de commande 0x047F)
Attente acquittement drives : lecture statut "Ready to operate"

Critère passage étape 4 :

4/4 drives répondent "Operation enabled" (bit 2 statut = 1)


Étape 4 : Montée vitesse progressive [7-37s]
Actions :

Application rampe accélération (FB104 "Speed_Ramp_Control")
Vitesse cible initiale : 10% Nnom (seuil mini fonctionnement)
Durée rampe configurable : 10-30 secondes (DB40.Ramp_Time)

Profil rampe (exemple 1500 tr/min cible) :



Temps
Vitesse consigne
Couple moteur
Flux rotor



0 s
0 tr/min
0 %
50 %


5 s
150 tr/min
15 %
60 %


10 s
450 tr/min
35 %
75 %


20 s
1050 tr/min
65 %
90 %


30 s
1500 tr/min
100 %
100 %


Surveillance durant rampe :

Écart vitesse réelle/consigne < 5% (tolérance glissement)
Couple moteur < 120% nominal (surcharge acceptée)
Températures AFE/moteur < seuils alarmes


Étape 5 : Stabilisation [37-42s]
Actions :

Maintien vitesse cible ±1%
Ajustement fin régulation PID (FB104)
Validation flux rotor optimal (lecture Modbus Exciter)

Critère passage étape 6 :

Vitesse stable pendant 5 secondes consécutives
Aucune alarme niveau 2/3 active


Étape 6 : Fonctionnement établi [>42s]
État final :

✅ Mode RUN confirmé (DB130.Current_State = 3)
✅ Régulations toutes actives (vitesse, flux, refroidissement)
✅ LED verte "RUN" allumée fixe
✅ Supervision HMI affiche courbes temps réel

Journalisation :

Timestamp démarrage complet
Durée totale séquence
Paramètres appliqués (vitesse, flux, rampe)


Régulations actives en mode RUN
3 boucles PID principales (cycle 100ms OB1) :
Boucle 1 : Régulation vitesse moteur (FB104)
Paramètres (DB20.PID_Speed) :



Paramètre
Valeur
Unité
Fonction



Kp
0.8
-
Gain proportionnel


Ki
0.3
1/s
Gain intégral (anti-erreur statique)


Kd
0.05
s
Gain dérivé (amortissement)


Setpoint
Variable
tr/min
Consigne opérateur (150-3000 tr/min)


Process Value
Feedback
tr/min
Lecture encodeur via Profibus


Sorties régulation :

AQ0 "Speed_Setpoint" : 0-10 V vers drives (consigne couple)
Limitation rampe : ±50 tr/min/s (évite à-coups mécaniques)


Boucle 2 : Régulation flux rotor (FB123)
Paramètres (DB20.PID_Flux) :



Paramètre
Valeur
Unité
Fonction



Kp
1.2
-
Gain proportionnel


Ki
0.1
1/s
Gain intégral lent (temps caractéristique long)


Kd
0.0
s
Pas de gain dérivé (bruit mesure)


Setpoint
100 %
%
Flux nominal optimal


Process Value
Modbus
%
Lecture exciter (polling 500ms)


Sorties régulation :

AQ1 "Flux_Setpoint" : 4-20 mA vers exciter (consigne courant d'excitation)
Limitation : 50-110% flux nominal (protection surexcitation)


Boucle 3 : Régulation température WCS (FB112)
Paramètres (DB20.PID_Cooling) :



Paramètre
Valeur
Unité
Fonction



Kp
2.5
-
Gain proportionnel fort (réactivité)


Ki
0.05
1/s
Gain intégral très lent (inertie thermique)


Kd
1.0
s
Gain dérivé (anticipation montée température)


Setpoint
60 °C
°C
Température cible AFE


Process Value
AI0
°C
Mesure PT100 (filtre 10s)


Sorties régulation :

AQ3 "WCS_Flow_Setpoint" : 4-20 mA consigne débit (30-100%)
Activation pompe secours si T > 70°C (backup automatique)


Surveillance continue (alarmes niveau 1)
10 paramètres surveillés cycliquement (200ms) par FB201 :
Warnings non bloquants (niveau 1) :



Paramètre
Seuil warning
Action automatique



Température AFE
> 75°C
Augmentation débit WCS +20%


Température moteur
> 110°C
Réduction vitesse -5%


Vibrations moteur
> 8 mm/s
Alarme + enregistrement


Courant stator
> 110% Inom
Réduction couple -10%


Débit WCS
< 60 L/min
Activation pompe secours


Pression WCS
< 1.8 bars
Ouverture vanne appoint


Flux rotor
< 95% consigne
Notification maintenance


Écart vitesse
> 3%
Ajustement gains PID


Temps cycle OB1
> 120 ms
Alarme charge CPU


Mémoire CPU
> 80%
Purge buffer alarmes


Affichage HMI temps réel :

Courbes tendances 10 min glissantes
Compteur alarmes niveau 1 (journalier)
Indicateurs colorés (vert/jaune/rouge)


Transitions possibles depuis RUN
Transition T5 → READY (Arrêt normal) :
Séquence arrêt contrôlé (10-30s) :

Décélération moteur : Rampe -50 tr/min/s jusqu'à 0 tr/min
Réduction flux : Consigne exciter 100% → 0% en 5s
Maintien drives : État "Operation enabled" maintenu 5s
Désarmement drives : Q1.0-1.3 = 0 après arrêt confirmé
Ouverture contacteur AFE : Q0.3 = 0 (coupure ligne)
Passage READY : État sûr avec WCS maintenu actif

Déclenchement :

Bouton HMI "Stop normal"
Fin de séquence automatique (si configuré)
Commande SCADA (si supervision externe)


Transition T4 → FAULT (Défaut niveau 2/3) :
Arrêt d'urgence automatique (< 1s) :

Coupure immédiate : Q0.3 Contactor_AFE = 0 (< 50ms)
Désarmement drives : Commande Profibus "Quick stop" (< 100ms)
Flux nul : Consigne exciter = 0% (protection rotor)
WCS maintenu : Pompe 100% (évacuation chaleur résiduelle)
Sauvegarde contexte : Archivage états + courbes (DB201)
Passage FAULT : État diagnostic avec blocage redémarrage

Déclenchements typiques :

Surchauffe > 80°C AFE ou > 120°C moteur
Défaut drive (court-circuit, surcharge, perte encodeur)
Perte communication Profibus > 3s
Arrêt urgence actionné (I0.0 ou I0.1 = 0)


3.6 MODE FAULT (Défaut)
Fonction : Diagnostic et classification défauts + procédures réarmement
Classification alarmes (3 niveaux)
Hiérarchie des défauts :
Niveau 1 : Warning (⚠️)
Caractéristiques :

Non bloquant pour fonctionnement
Notification opérateur uniquement
Acquittement optionnel
Comptabilisé statistiques maintenance

Exemples (15 warnings définis) :



Code
Description
Seuil
Action auto



W01
Température AFE élevée
75-80°C
Débit WCS +20%


W02
Température moteur élevée
110-120°C
Vitesse -5%


W03
Vibrations anormales
8-10 mm/s
Enregistrement


W04
Débit WCS réduit
50-60 L/min
Pompe secours


W05
Communication Exciter lente
1-2 s timeout
Retry +2


W06
Flux rotor sous-optimal
92-95%
Ajustement Kp


W07
Écart vitesse temporaire
3-5%
Gains PID auto


W08
Charge CPU élevée
80-90%
Purge buffers


W09
Mémoire saturée
> 80%
Archivage logs


W10
Isolation moteur faible
1-5 MΩ
Planif maintenance


Gestion warnings :

Compteur journalier (reset minuit)
Si > 50 warnings/jour → Alarme niveau 2 "Maintenance requise"
Historique 30 jours dans DB202


Niveau 2 : Alarm (🔶)
Caractéristiques :

Arrêt automatique si en RUN
Passage FAULT (mais pas STOP)
Réarmement possible après acquittement
Nécessite validation disparition cause

Exemples (25 alarmes définies) :



Code
Description
Condition
Temps réaction
Réarmement



A01
Surchauffe AFE
T > 80°C
< 100 ms
T < 70°C + Ack


A02
Surchauffe moteur
T > 120°C
< 100 ms
T < 100°C + Ack


A03
Défaut drive (non critique)
Fault flag = 1
< 100 ms
Reset drive + Ack


A04
Perte comm Profibus
Timeout > 3s
< 200 ms
Comm rétablie + Ack


A05
Débit WCS insuffisant
< 50 L/min
< 500 ms
Débit OK + Ack


A06
Pression WCS basse
< 1.5 bars
< 500 ms
Pression OK + Ack


A07
Surcharge moteur
Couple > 130%
5 s intégré
Charge réduite + Ack


A08
Déséquilibre phases
> 5% écart
2 s intégré
Équilibre OK + Ack


A09
Flux rotor instable
±10% oscillations
3 s moyenné
Stabilisation + Ack


A10
Vitesse incontrôlable
Écart > 10%
1 s intégré
Régulation OK + Ack


Procédure acquittement alarme niveau 2 :

✅ Diagnostic : Affichage cause racine sur HMI (texte explicite)
✅ Attente disparition : Condition déclenchante = OK pendant 10s
✅ Acquittement opérateur : Bouton "Acknowledge" + saisie commentaire (optionnel)
✅ Validation système : FB130 vérifie compteur alarmes < 10/heure
✅ Transition T8 : Passage FAULT → READY (redémarrage autorisé)

Blocage réarmement si :

Condition déclenchante toujours présente

10 alarmes identiques en 1h (oscillations)


Mode "Maintenance forcée" actif


Niveau 3 : Critical Fault (🔴)
Caractéristiques :

Arrêt d'urgence immédiat (< 50ms)
Passage STOP obligatoire (via T6)
Réarmement nécessite intervention maintenance
Verrouillage redémarrage logiciel

Exemples (12 défauts critiques) :



Code
Description
Condition
Action immédiate
Réarmement



C01
Arrêt urgence actionné
I0.0 OU I0.1 = 0
Coupure AFE + drives
Maintenance physique


C02
Surtension DC Bus
> 750 VDC
Coupure AFE
Vérif condensateurs


C03
Court-circuit moteur
Drive fault 0x8400
Coupure ligne
Inspection moteur


C04
Défaut terre/isolation
< 1 MΩ
Coupure complète
Test mégohmmètre


C05
Perte encoder vitesse
Drive fault 0x7121
Arrêt drives
Remplacement encoder


C06
Défaut refroidissement
Débit = 0 L/min
Arrêt + alarme externe
Vérif pompes/vannes


C07
Température critique
T > 150°C moteur
Arrêt + ventilation forcée
Refroidissement + insp


C08
Défaut hardware CPU
Diagnostic module
Arrêt complet
Redémarrage CPU


C09
Défaut mémoire
Checksum KO
Sauvegarde + arrêt
Rechargement DB


C10
Défaut communication
Perte tous réseaux
Mode dégradé local
Vérif câblages


C11
Surcharge mécanique
Couple > 200%
Arrêt immédiat
Inspection accouplement


C12
Défaut excitatrice
Pas de réponse 5s
Arrêt flux
Remplacement exciter


Procédure réarmement défaut critique :

🔴 Intervention maintenance obligatoire : Clé physique "Maintenance" tournée (I5.0 = 1)
🔴 Diagnostic approfondi : Téléchargement logs CPU (via TIA Portal)
🔴 Résolution cause : Réparation/remplacement matériel si nécessaire
🔴 Tests validation : Exécution séquence POST complète (mode INIT)
🔴 Réinitialisation compteurs : Reset compteur défauts critiques (DB201)
🔴 Autorisation logicielle : Variable Critical_Fault_Override = TRUE (durée 10 min)
🔴 Transition T9 : Passage FAULT → STOP (après validation maintenance)

Journalisation défauts critiques :

Archivage permanent (non effaçable)
Horodatage précis (ms)
Sauvegarde courbes 10s avant défaut
Notification email automatique (si SCADA connecté)


Affichage HMI page défauts
Vue synthétique (page principale) :
┌──────────────────────────────────────────────────┐
│  ⚠️  DÉFAUTS ACTIFS : 3                          │
├──────────────────────────────────────────────────┤
│  🔴 [C01] Arrêt urgence circuit 1 (12:34:56)    │
│  🔶 [A03] Défaut Drive#2 - Surcharge (12:34:52) │
│  ⚠️  [W01] Température AFE élevée (12:30:15)     │
├──────────────────────────────────────────────────┤
│  [DÉTAILS]  [ACQUITTER NIVEAU 2]  [HISTORIQUE]  │
└──────────────────────────────────────────────────┘
Vue détaillée (page diagnostics) :



Timestamp
Code
Niveau
Description
État
Actions



12:34:56
C01
🔴 Critical
Arrêt urgence circuit 1
ACTIF
Maintenance


12:34:52
A03
🔶 Alarm
Défaut Drive#2
ACTIF
[Acquitter]


12:30:15
W01
⚠️ Warning
Temp AFE 76°C
ACTIF
Auto-régulé


12:15:03
W07
⚠️ Warning
Écart vitesse 3.2%
Résolu
-


11:58:22
A05
🔶 Alarm
Débit WCS 48 L/min
Acquitté
Logs


Filtres disponibles :

Par niveau (1/2/3)
Par état (actif/résolu/acquitté)
Par plage horaire (dernière heure/jour/semaine)
Par sous-système (AFE/Drive/WCS/Exciter)
