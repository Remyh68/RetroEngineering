# **Spécifications Techniques - Rétro-Engineering PLC Siemens (TIA Portal)**
*Standard : AF_TNS_ING_130_v0.3 | Version : 1.0 | Date : 2023-11-XX*

---
## **Table des Matières**
1. [Exigences Fonctionnelles (FR)](#fr)
   - [FR-001 : Démarrage](#fr-001)
   - [FR-002 : State Machine](#fr-002)
   - [FR-003 : Gestion des Faults](#fr-003)
2. [Interfaces (I)](#interfaces)
3. [Exigences Non Fonctionnelles (NFR)](#nfr)
4. [Annexes](#annexes)

---

## <a name="fr"></a>1. **Exigences Fonctionnelles**

### <a name="fr-001"></a>FR-001 : Démarrage Séquentiel
**Description** : Vérification des conditions pré-démarrage et initialisation des watchdogs.

| Étape               | Condition                     | Bloc PLC Associé       | Sortie Attendue          |
|---------------------|-------------------------------|------------------------|--------------------------|
| Vérif Alimentations | `%IW100.0` = TRUE             | `FC100_CheckPower`     | `Power_OK` (Bool)        |
| Init Watchdog       | `DB_Watchdog.DBW 10` = 0      | `FB200_InitSystem`     | Watchdog démarré        |
| Interlocks          | `%IX200.5` (Sécurité) = TRUE  | `FC150_CheckInterlock` | `Interlock_OK` (Bool)    |

**Notes** :
- Le démarrage est bloqué si `Power_OK` ou `Interlock_OK` = FALSE.
- Temps max d'initialisation : **500 ms** (NFR-001).

---

### <a name="fr-002"></a>FR-002 : State Machine
**Diagramme Mermaid** (à visualiser [ici](https://mermaid.live/)) :
```mermaid
stateDiagram-v2
    [*] --> Step0_Reset
    Step0_Reset --> Step1_AFEReady : AFE_Status = OK
    Step1_AFEReady --> Step40_Operation : Start_CMD + Interlocks_OK
    Step40_Operation --> Step50_Fault : Alarme critique (ex: AFE_TripHW)
    Step50_Fault --> Step0_Reset : Reset manuel
