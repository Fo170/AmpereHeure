# Changelog - AmpereHeure

## [Séparation des dépôts] - 2026-10-19

Le dépôt historique `Fo170/CoulombsAh` contenait **trois** headers
(`Coulomb.h`, `AmpereHeure.h`, `CoulombsAh.h`) alors qu'ils sont parfaitement
indépendants : aucun include croisé, dépendance unique `<Arduino.h>`.

Chacun est désormais dans **son propre dépôt**, avec son historique propre :

| Dépôt | Header | Classe | Version |
|---|---|---|---|
| [Fo170/CoulombsAh](https://github.com/Fo170/CoulombsAh) | `CoulombsAh.h` | `Ah_GD` | 3.1.0 |
| [Fo170/Coulomb](https://github.com/Fo170/Coulomb) | `Coulomb.h` | `Coulomb` | 2.0.1 |
| **Fo170/AmpereHeure** | **`AmpereHeure.h`** | **`AmpereHeure`** | **2.0.1** |

Ce dépôt démarre à la version **2.0.1**, qui était la dernière version publiée de
`AmpereHeure.h` dans l'ancien dépôt commun. L'historique antérieur (1.x → 2.x)
reste consultable dans `Fo170/CoulombsAh` jusqu'au tag `v3.0.1`.

Les entrées ci-dessous sont reprises de l'ancien CHANGELOG commun, **filtrées**
pour ne garder que ce qui concerne `AmpereHeure.h`.

## [2.0.1] - 2026-10-19

### 🏷️ Divers

- Version `2.0.0` → `2.0.1` (`library.json` + macro)
- Aucun changement fonctionnel sur `AmpereHeure.h` : le correctif de la 2.0.1
  concernait `Coulomb.h` (`getLastInterval()` non mis à jour par
  `addMeasurementWithInterval()`). La version est alignée par cohérence, les
  deux headers étant publiés ensemble à l'époque.

## [2.0.0] - 2026-10-19

### 💥 BREAKING — `AmpereHeure` redevient un INTÉGRATEUR PUR

**Principe.** Une librairie de comptage doit savoir faire `Σ = Σ + I·Δt/3600` et
rien d'autre. Toute **politique physique** — rendement faradique, Peukert, état
de charge, capacité batterie, autonomie, cycles — est une décision du **projet**
qui utilise la librairie. Elle a donc été **supprimée**.

### ❌ API SUPPRIMÉE

| Méthode retirée | Motif |
|---|---|
| `setBatteryCapacity()` / `getBatteryCapacity()` | notion de capacité = politique du projet |
| `setEfficiencies()` / `getChargeEfficiency()` / `getDischargeEfficiency()` / `variableEfficiency()` | rendement = politique du projet |
| `setPeukertExponent()` / `getPeukertExponent()` / `setPeukertReferenceRate()` / `getPeukertReferenceRate()` | Peukert = politique du projet |
| `updateStateOfCharge()` (privé) | SoC = politique du projet |
| `getStateOfCharge()` / `getStateOfChargePercent()` | idem |
| `calibrateStateOfCharge()` | idem |
| `getRemainingCapacity()` | idem |
| `setLowBatteryThreshold()` / `isLowBattery()` | idem |
| `getEstimatedCycles()` | dérivé de la capacité |
| `getEstimatedRuntimeHours()` / `getEstimatedRuntimeFormatted()` | dérivé de la capacité |
| `getPower(V)` | ne relève pas du comptage |

**Signature du constructeur** : `AmpereHeure(float filterAlpha = 0.99)`.
La capacité n'est plus un paramètre (elle n'a plus de sens ici).

### ✅ API AJOUTÉE

```cpp
float addCharge(float ahDelta);   // Ah déjà intégrés, signés
```

Ne touche ni `_current` ni `_lastMicros` : aucune notion de temps. Rejette
NaN/Inf.

### 🧪 Ce qui est CONSERVÉ

`begin`, `addMeasurement` (×2), `addMeasurementWithInterval`, `addCharge`,
`reset` (×2), `setTotalAh`, `getAh`, `getTotalAh`, `getCurrent`,
`getAverageCurrent`, `getCurrentDirection`, `isCharging`, `isDischarging`,
`getTotalCharged`, `getTotalDischarged`, `getLastInterval`,
`getAverageInterval`, `getSampleCount`, `getMaxAh`, `getMinAh`,
`setFilterAlpha`, `printDebug`.

### ⚠️ Migration

```cpp
// AVANT (1.2.x)
AmpereHeure ah(14.0f);                    // capacité au constructeur
ah.setEfficiencies(0.96f, 0.98f);
ah.setLowBatteryThreshold(0.15f);
float soc = ah.getStateOfCharge();

// APRÈS (2.0.0) — la politique est dans le projet
AmpereHeure ah;                           // plus de capacité
float eta = (I > 0.0f) ? 0.96f : 0.98f;
ah.addMeasurement(I * eta, micros());

float ahNet = ah.getTotalCharged() - ah.getTotalDischarged();
float soc = 0.90f + ahNet / BATTERY_CAPACITY_AH;   // tenu par le projet
```

### 🏷️ Divers

- `library.json` : `1.2.1` → `2.0.0`
- `__AMPEREHEURE_H_VERSION__` : `1.2.1` → `2.0.0`
