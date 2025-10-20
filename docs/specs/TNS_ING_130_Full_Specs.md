# **Spécifications Techniques Complètes - Rétro-Engineering PLC Siemens**
*Standard: AF_TNS_ING_130_v0.3 | Version: 1.1 | Date: 2023-11-XX*
*Extendu avec: Tests SART, Mapping E/S, Exemples de Code*

---
## **Table des Matières**
1. [Exigences Fonctionnelles (FR)](#fr)
2. [Interfaces (I)](#interfaces)
   - [Mapping Complet E/S](#mapping-ios)
   - [Protocoles de Communication](#protocoles)
3. [Exigences Non Fonctionnelles (NFR)](#nfr)
4. [Tests SART](#sart)
5. [Annexes](#annexes)
   - [Exemples de Code](#code)
   - [Templates Standard [R3]](#templates)

---

## <a name="fr"></a>1. **Exigences Fonctionnelles**
*(Section inchangée par rapport à la version précédente, mais voici un ajout pour **FR-004**)*

### **FR-004 : Gestion des Rampes (FB222)**
**Description** : Contrôle progressif des sorties analogiques pour éviter les chocs mécaniques.

| Paramètre       | Adresse PLC          | Valeur par Défaut | Unité  |
|-----------------|----------------------|-------------------|--------|
| Rampe Montée    | `DB_Ramp.DBW 2`      | 5.0               | s      |
| Rampe Descente  | `DB_Ramp.DBW 4`      | 3.0               | s      |
| Sortie Cible    | `%QW100.0`           | 0-100%            | %      |

**Logique** :
```pascal
// Extrait de FB222_RampControl
IF "Start_Ramp" THEN
   IF "Current_Output" < "Target_Output" THEN
      "Current_Output" := "Current_Output" + (Ramp_Rate * Cycle_Time);
   END_IF;
END_IF;
