# AmpereHeure

Compteur de capacité en **ampères-heures** pour ESP8266 — **intégrateur pur**.

Version **2.0.1** · Licence **GPL-3.0** · Auteur : Olivier FOURNET (olivier.fournet@free.fr) · Dépôt : [github.com/Fo170/AmpereHeure](https://github.com/Fo170/AmpereHeure)

> ## Cette librairie ne fait que COMPTER
>
> ```
> ΔAh = I · Δt / 3600
> ```
>
> Aucune physique : ni rendement faradique, ni Peukert, ni état de charge, ni
> capacité batterie, ni autonomie, ni cycles. Ces politiques appartiennent au
> **projet** qui utilise la librairie.
>
> ```cpp
> AmpereHeure ah;
> ah.addMeasurement(I * eta);    // le projet pondère s'il le souhaite
> ```
>
> Dépendance unique : `<Arduino.h>`.

## Contenu

| Header | Classe | Rôle |
|---|---|---|
| `AmpereHeure.h` | `AmpereHeure` | intégration et cumul de la capacité en Ah (1 Ah = 3600 C) |

> ### 📦 Dépôt séparé
>
> Ce header a vécu dans `Fo170/CoulombsAh` jusqu'à la 3.0.1, aux côtés de
> `Coulomb.h` et `CoulombsAh.h`. Les trois étaient **indépendants** (aucun
> include croisé) : ils sont désormais dans trois dépôts distincts.
>
> | Dépôt | Header | Classe | Version |
> |---|---|---|---|
> | [Fo170/CoulombsAh](https://github.com/Fo170/CoulombsAh) | `CoulombsAh.h` | `Ah_GD` | 3.1.0 |
> | [Fo170/Coulomb](https://github.com/Fo170/Coulomb) | `Coulomb.h` | `Coulomb` | 2.0.1 |
> | **Fo170/AmpereHeure** | **`AmpereHeure.h`** | **`AmpereHeure`** | **2.0.1** |

---

## ⚠️ Précision — pourquoi préférer `CoulombsAh` pour un cumul long

`AmpereHeure` cumule en **`float32`**. Un `float32` ne peut pas incrémenter une
grande valeur par petits pas : l'ULP croît avec la magnitude.

| `total` | ULP float32 | Incrément 0,1 A × 0,1 s | Conservé ? |
|---|---|---|---|
| 1 Ah | 1,19e-07 Ah | 2,78e-06 Ah | oui |
| 26 Ah | 1,91e-06 Ah | 2,78e-06 Ah | oui (1,5 ULP) |
| **100 Ah** | **7,63e-06 Ah** | 2,78e-06 Ah | **non** |
| **910 Ah** | **6,10e-05 Ah** | 2,78e-06 Ah | **non (22× sous l'ULP)** |
| 10 000 Ah | 9,77e-04 Ah | 2,78e-06 Ah | non (350× sous l'ULP) |

Conséquence : au-delà de quelques dizaines d'ampères-heures, chaque échantillon
est **arrondi à zéro** et le compteur se fige silencieusement. Sur 10 000 000
d'échantillons (exact 27,777778 Ah), un cumul `float32` donne **26,578040 Ah,
soit −1,20 Ah (−4,3 %)**.

`AmpereHeure` reste parfaitement adapté à un **compteur remis à zéro
périodiquement**, où la valeur reste petite. Pour un **odomètre** qui grandit
indéfiniment, utiliser [`Fo170/CoulombsAh`](https://github.com/Fo170/CoulombsAh) :
accumulateur `int64` à deux étages, exact à toute magnitude.

---

## Utilisation

```cpp
#include "AmpereHeure.h"

AmpereHeure ah;

void setup() { ah.begin(); }

void loop() {
    float I = lireCourant();      // A, > 0 = charge
    ah.addMeasurement(I);         // timestamp = micros()
    // ou, si l'intervalle est connu :
    ah.addMeasurementWithInterval(I, 0.1f);
}
```

### Injection d'une charge déjà intégrée

```cpp
float addCharge(float ahDelta);   // signé : < 0 = décharge
```

Utile quand le projet calcule lui-même son incrément (rendement, Peukert…).
Ne touche ni `getCurrent()` ni l'horloge interne. Les NaN/Inf sont rejetés.

### Ajouter une politique locale

La librairie ne connaît ni la capacité de la batterie ni son état de charge.
C'est au sketch de les tenir :

```cpp
#define BATTERY_CAPACITY_AH  14.0f
#define ETA_CHARGE           0.96f

float soc = 0.90f;                       // tenu par le projet

void loop() {
    float I = readCurrent();
    float eta = (I > 0.0f) ? ETA_CHARGE : 1.0f;
    ah.addMeasurement(I * eta, micros());

    float ahNet = ah.getTotalCharged() - ah.getTotalDischarged();
    soc = 0.90f + ahNet / BATTERY_CAPACITY_AH;
    if (soc > 1.0f) soc = 1.0f;
    if (soc < 0.0f) soc = 0.0f;
}
```

---

## API

| Méthode | Rôle |
|---|---|
| `begin(initialAh = 0)` | Initialise |
| `addMeasurement(I)` / `addMeasurement(I, microsTimestamp)` | Intègre |
| `addMeasurementWithInterval(I, dt_s)` | Intègre sur un intervalle (0 < Δt ≤ 3600 s) |
| `addCharge(ahDelta)` | Ajoute des Ah déjà intégrés (signés) |
| `reset()` / `reset(bool)` | Remet à zéro |
| `setTotalAh(ah)` | Fixe la **base** (persistance) |
| `getAh()` / `getTotalAh()` | Session / base + session |
| **`getTotalCharged()` / `getTotalDischarged()`** | Cumuls par sens, **tous deux positifs** |
| `getCurrent()` / `getAverageCurrent()` | Courant |
| `getCurrentDirection()` | `1` charge, `−1` décharge, `0` repos |
| `isCharging()` / `isDischarging()` | Seuil ±100 mA |
| `getLastInterval()` / `getAverageInterval()` | Intervalles |
| `getSampleCount()` | Nombre d'échantillons |
| `getMaxAh()` / `getMinAh()` | Bornes |
| `setFilterAlpha(α)` | Coefficient EMA (0–1) |
| `printDebug()` | État interne sur le port série |

### ⚠️ Cumuls bruts, pas un solde

`getTotalCharged()` et `getTotalDischarged()` sont **indépendants** : ils
cumulent chacun de leur côté et restent positifs. Le net se calcule par
différence :

```cpp
float net = ah.getTotalCharged() - ah.getTotalDischarged();
```

### Garde-fous

- **Δt hors `]0 ; 3600]`** rejeté.
- **`I` = NaN ou ±Inf** rejeté : un seul échantillon corrompu empoisonnerait
  définitivement le cumul.
- **Débordement de `micros()`** (~70 min) géré.

---

## Persistance

Déléguée à [`Fo170/Persistance`](https://github.com/Fo170/Persistance)
(EEPROM, FRAM ou LittleFS). Le header **ne l'inclut pas**.

```cpp
float saved;
if (persistance.loadFloat("ah", &saved)) ah.setTotalAh(saved);
// …
persistance.saveFloat("ah", ah.getTotalAh());
```

⚠️ `setTotalAh()` définit une **base** sur laquelle l'intégration s'ajoute.

---

## Migration

| Avant | Après |
|---|---|
| `#include "AmpereHeure.h"` (depuis `Fo170/CoulombsAh`) | `#include "AmpereHeure.h"` (depuis `Fo170/AmpereHeure`) — **aucun changement de code** |
| `AmpereHeure(float capacityAh, float alpha)` | `AmpereHeure(float alpha = 0.99)` — la capacité n'est plus un paramètre (2.0.0) |
| `setBatteryCapacity()` / `getBatteryCapacity()` | supprimés — politique du projet |
| `setEfficiencies()` / `getChargeEfficiency()` / `getDischargeEfficiency()` | supprimés — politique du projet |
| `setPeukertExponent()` / `setPeukertReferenceRate()` + getters | supprimés — politique du projet |
| `getStateOfCharge()` / `getStateOfChargePercent()` / `calibrateStateOfCharge()` | supprimés — politique du projet |
| `getRemainingCapacity()` / `setLowBatteryThreshold()` / `isLowBattery()` | supprimés — politique du projet |
| `getEstimatedCycles()` / `getEstimatedRuntimeHours()` / `getEstimatedRuntimeFormatted()` | supprimés — dérivés de la capacité |
| `getPower(V)` | supprimé — ne relève pas du comptage |
| `begin(initialAh, initialSoC)` | `begin(initialAh = 0)` |
| — | **`addCharge(ahDelta)` ajouté** |

Pour passer à l'accumulateur exact, voir la table de migration de
[`Fo170/CoulombsAh`](https://github.com/Fo170/CoulombsAh).

---

## Sources et références

Consultées le **2026-10-19**. Toutes répondent HTTP 200 à cette date.

| Sujet | Source | Ce qui en est tiré |
|---|---|---|
| Rendement coulombique du plomb | [Battery University — BU-808c](https://batteryuniversity.com/article/bu-808c-coulombic-and-energy-efficiency-with-the-battery) | Plomb : CE ≈ **90 %** ; un courant très faible **abaisse** le rendement (auto-décharge) — il ne remonte pas à 100 % |
| Loi de Peukert | [Wikipedia — Peukert's law](https://en.wikipedia.org/wiki/Peukert%27s_law) | `C/C₀ = (I₀/I)^(k−1)` : la capacité **disponible décroît** avec le courant. Ne s'applique jamais comme rendement |
| Représentation IEEE 754 (ULP) | [Wikipedia — Unit in the last place](https://en.wikipedia.org/wiki/Unit_in_the_last_place) | L'ULP d'un float32 croît avec la magnitude ⇒ justifie de préférer un accumulateur `int64` pour un cumul long |

---

## Historique synthétique des versions

| Version | Changement majeur |
|---|---|
| **2.0.1** | `getLastInterval()` n'était pas mis à jour par `addMeasurementWithInterval()`. |
| **2.0.0** | **RUPTURE** — la librairie devient un **intégrateur pur**. SoC, capacité, rendements, Peukert, cycles et autonomie sont supprimés et migrent dans le projet. Le constructeur ne prend plus la capacité. `addCharge()` ajouté. |
| 1.2.0 | Peukert dynamique + rendement η variable |
| 1.1.0 | Persistance déléguée à `Fo170/Persistance` (EEPROM/FRAM/LittleFS) |
| 1.0.0 | Version initiale |

> 📦 **Séparation des dépôts (2026-10-19)** : `AmpereHeure.h` était dans
> `Fo170/CoulombsAh` aux côtés de `Coulomb.h` et `CoulombsAh.h`. Les trois
> headers étant indépendants, chacun a désormais son dépôt. L'historique de ce
> fichier est conservé dans `Fo170/CoulombsAh` jusqu'au tag `v3.0.1`.
