# 🛰️ Airspy R2 — Optimisations Firmware LEO Satellite

<p align="center">
  <img src="https://img.shields.io/badge/Callsign-F4TNK-yellow?style=for-the-badge&logo=radio" />
  <img src="https://img.shields.io/badge/Cible-Satellite%20LEO-blue?style=for-the-badge&logo=satellite" />
  <img src="https://img.shields.io/badge/MCU-LPC4370-green?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Tuner-R820T2-red?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Risque%20Brick-ZERO-brightgreen?style=for-the-badge" />
</p>

---

> **Objectif** : Maximiser la sensibilité de réception de l'Airspy R2 pour les signaux faibles des satellites LEO (NOAA, Meteor, CubeSats, ISS, HRPT).
> Toutes les modifications sont réversibles par reflash ou DFU.

---

## 📋 Table des matières

- [🏗️ Architecture matérielle](#-architecture-matérielle)
- [📡 Chaîne RF complète](#-chaîne-rf-complète)
- [🛰️ Caractéristiques des signaux LEO](#-caractéristiques-des-signaux-leo)
- [🔧 Modifications appliquées](#-modifications-appliquées)
  - [⚡ Mod 1 — Gain filtre IF +3dB (R0x06)](#-modification-1--gain-filtre-if-3db-r0x06)
  - [⚡ Mod 2 — Courant mixer maximal (R0x07)](#-modification-2--courant-mixer-maximal-r0x07)
  - [⚡ Mod 3 — Courant filtre IF maximal (R0x09)](#-modification-3--courant-filtre-if-maximal-r0x09)
  - [⚡ Mod 4 — Seuils AGC LNA relevés (R0x0D)](#-modification-4--seuils-agc-lna-relevés-r0x0d)
  - [⚡ Mod 5 — Seuil AGC mixer relevé (R0x0E)](#-modification-5--seuil-agc-mixer-relevé-r0x0e)
  - [⚡ Mod 6 — TOP mixer plus élevé (R0x1C)](#-modification-6--top-mixer-plus-élevé-r0x1c)
  - [⚡ Mod 7 — TOP LNA plus élevé (R0x1D)](#-modification-7--top-lna-plus-élevé-r0x1d)
  - [⚡ Mod 8 — FILTER_EXT activé (R0x1E)](#-modification-8--filter_ext-activé-r0x1e)
  - [⚡ Mod 9 — OPTIM_SET_MUX (Makefile)](#-modification-9--optim_set_mux-makefile)
  - [⚡ Mod 10 — CLK7 drive 8mA (SI5351C)](#-modification-10--clk7-drive-8ma-si5351c)
- [�️ GPSDO — Référence externe 10 MHz](#-gpsdo--référence-externe-10-mhz)
  - [⚡ Mod 11 — CLK7 8mA en mode CLKIN](#-modification-11--clk7-8ma-en-mode-clkin-gpsdo)
- [📊 Tableau récapitulatif des 11 modifications](#-tableau-récapitulatif-des-11-modifications)
- [🔬 Analyse ADC / DMA / Horloge](#-analyse-adc--dma--horloge)
- [🧪 Configuration recommandée pour LEO](#-configuration-recommandée-pour-leo)
- [🆘 Procédure de récupération DFU](#-procédure-de-récupération-dfu)
- [📚 Annexe — Carte des registres R820T2](#-annexe--carte-des-registres-r820t2)

---

## 🏗️ Architecture matérielle

```mermaid
graph TB
    subgraph "🖥️ NXP LPC4370FET100"
        M4["🔴 Cortex-M4F<br/>140 MHz<br/>ADC + DMA + Packing"]
        M0["🔵 Cortex-M0<br/>USB + R820T ctrl"]
        M0S["⚪ Cortex-M0sub<br/>DÉSACTIVÉ<br/>(disponible DSP futur)"]
        ADC["📐 ADCHS<br/>12-bit ADC"]
        DMA["💾 DMA<br/>4 LLI round-robin<br/>32 KB buffer"]
        USB["🔌 USB 2.0 HS<br/>480 MHz bulk"]
    end

    subgraph "📻 Frontal RF"
        ANT["📡 Antenne"]
        R820T["🎛️ R820T2<br/>Tuner RF"]
        SI["🕐 SI5351C<br/>Clock Gen"]
        XTAL["💎 TCXO 25 MHz"]
    end

    ANT -->|"RF Signal"| R820T
    R820T -->|"IF analogique"| ADC
    ADC -->|"12-bit samples"| DMA
    DMA -->|"RAM buffer"| M4
    M4 -->|"packed data"| USB
    M0 -->|"I2C1 400kHz"| R820T
    M0 -->|"I2C0 400kHz"| SI
    XTAL --> SI
    SI -->|"CH0: 25 MHz"| R820T
    SI -->|"CH7: 20 MHz"| ADC
    M0 <-->|"IPC"| M4

    style M4 fill:#ff6b6b,stroke:#333,color:#fff
    style M0 fill:#4dabf7,stroke:#333,color:#fff
    style M0S fill:#868e96,stroke:#333,color:#fff
    style R820T fill:#ffd43b,stroke:#333,color:#000
    style ADC fill:#69db7c,stroke:#333,color:#000
    style USB fill:#845ef7,stroke:#333,color:#fff
    style SI fill:#ff922b,stroke:#333,color:#fff
```

### 📋 Composants clés

| Composant | Interface | Horloge | Rôle |
|-----------|-----------|---------|------|
| 🎛️ R820T2 | I2C1 (400 kHz) | 25 MHz (TCXO via SI5351C CH0) | Tuner RF : LNA → Mixer → Filtre IF → VGA |
| 🕐 SI5351C | I2C0 (400 kHz) | 25 MHz TCXO | Génération des horloges |
| 📐 ADCHS | Interne | 20 MHz (SI5351C CH7 → GP_CLKIN) | Conversion analogique-numérique 12-bit |
| 🔌 USB 2.0 HS | PLL0USB | 480 MHz | Transfert bulk vers le PC |
| 🔴 Cortex-M4F | PLL1 | 140 MHz (HS) / 40 MHz (LS) | ADC, DMA, packing en ASM ARM |
| 🔵 Cortex-M0 | — | 140 MHz | USB, contrôle R820T via I2C |

---

## 📡 Chaîne RF complète

```mermaid
flowchart LR
    subgraph "🔊 Étage LNA"
        A1["📡 Antenne<br/>Signal RF"]
        A2["🔺 LNA<br/>Low Noise Amp<br/>Gain: 0-47 dB"]
        A3["🔄 Tracking<br/>Filter"]
    end

    subgraph "🎚️ Étage Mixeur"
        B1["✖️ Mixer<br/>RF → IF"]
        B2["🕐 PLL Local<br/>Oscillator"]
    end

    subgraph "🔎 Étage IF"
        C1["🎛️ Filtre IF<br/>Passband"]
        C2["🔺 VGA<br/>Variable Gain<br/>-12 à +40.5 dB"]
    end

    subgraph "📐 ADC"
        D1["⚡ ADCHS<br/>12-bit<br/>20 MSPS max"]
        D2["💾 DMA<br/>4×8KB<br/>Round-robin"]
    end

    A1 --> A3 --> A2 --> B1
    B2 --> B1
    B1 --> C1 --> C2 --> D1 --> D2

    style A2 fill:#ff6b6b,stroke:#333,color:#fff
    style B1 fill:#ffd43b,stroke:#333,color:#000
    style C1 fill:#69db7c,stroke:#333,color:#000
    style C2 fill:#4dabf7,stroke:#333,color:#fff
    style D1 fill:#845ef7,stroke:#333,color:#fff
```

### 🔢 Taux d'échantillonnage disponibles

| Config | Débit | Fréq IF | BW R820T | Source horloge ADC |
|--------|-------|---------|----------|--------------------|
| 📌 Default 0 | **10 MSPS** | 5 MHz | 59 | GP_CLKIN direct (20 MHz) |
| 📌 Default 1 | **2.5 MSPS** | 1.25 MHz | 0 | GP_CLKIN / 4 (IDIVB=3) |
| 🔄 ALT 0 | 12 MSPS | 6 MHz | 63 | PLL0AUDIO (24 MHz) |
| 🔄 ALT 1 | 6 MSPS | 3 MHz | 32 | PLL0AUDIO (12 MHz) |
| 🔄 ALT 2 | 4.096 MSPS | 2.048 MHz | 25 | PLL0AUDIO (8.192 MHz) |

---

## 🛰️ Caractéristiques des signaux LEO

```mermaid
mindmap
  root((🛰️ Signaux<br/>LEO Sat))
    📻 Fréquences
      137 MHz NOAA/Meteor APT
      400-437 MHz CubeSats/ISS
      1.7 GHz HRPT
    📉 Puissance
      -120 à -140 dBm
      Très faibles
      Noyés dans le bruit
    📊 Bande passante
      15-50 kHz BPSK
      34 kHz APT
      120 kHz LRPT
    🌀 Doppler
      ±5-10 kHz à 137 MHz
      ±15-30 kHz à 437 MHz
      Variation continue
    ⏱️ Durée passage
      5-15 minutes
      Gain stable requis
    📧 Modulations
      BPSK / QPSK
      GFSK
      FM APT
```

### 🎯 Objectifs d'optimisation firmware

| # | Objectif | Raison |
|---|----------|--------|
| 1 | 🔺 **Maximiser la sensibilité** | Signaux à -130 dBm, chaque dB compte |
| 2 | 🔇 **Minimiser le bruit de phase** | Doppler continu → PLL propre nécessaire |
| 3 | 🎚️ **Optimiser la bande IF** | Signaux 15-150 kHz, pas du broadcast TV |
| 4 | ⚖️ **Stabiliser le gain AGC** | Éviter les oscillations sur signaux faibles |
| 5 | 🔋 **Courant maximal chaîne RF** | Plus de courant = moins de bruit thermique |
| 6 | ⚡ **Accélérer re-tuning Doppler** | Moins de latence I2C à chaque correction |

---

## 🔧 Modifications appliquées

> 📁 **Fichiers modifiés** : `common/airspy_nos_conf.c` et `common/Makefile_M0_inc.mk`
>
> ⚠️ **Toutes les modifications sont réversibles** par simple reflash du firmware d'origine

### Vue d'ensemble — Localisation des 10 modifications dans la chaîne RF

```mermaid
flowchart LR
    ANT["📡"] --> TF["Tracking<br/>Filter"]

    subgraph "🔧 Modifications appliquées"
        direction LR
        TF --> LNA["🔺 LNA<br/>Mod 4: AGC seuils ↑<br/>Mod 7: TOP ↑"]
        LNA --> MIX["✖️ Mixer<br/>Mod 2: Courant MAX<br/>Mod 5: AGC seuil ↑<br/>Mod 6: TOP ↑"]
        MIX --> FILT["🎛️ Filtre IF<br/>Mod 1: +3dB gain<br/>Mod 3: Courant MAX<br/>Mod 8: ⭐ FILTER_EXT"]
        FILT --> VGA["🔺 VGA"]
    end

    VGA --> ADC["📐 ADC<br/>Mod 10: CLK XTAL 8mA<br/>Mod 11: CLK GPSDO 8mA"]

    DOPPLER["🌀 Doppler<br/>Mod 9: OPTIM_SET_MUX"]
    DOPPLER -.->|"tuning rapide"| MIX

    GPSDO["🛰️ GPSDO<br/>10 MHz ±0.01 ppb<br/>Mod 11 ⭐"]
    GPSDO -.->|"auto-détecté au boot"| ADC

    style LNA fill:#ff6b6b,stroke:#333,color:#fff
    style MIX fill:#ffd43b,stroke:#333,color:#000
    style FILT fill:#69db7c,stroke:#333,color:#000
    style ADC fill:#845ef7,stroke:#333,color:#fff
    style DOPPLER fill:#4dabf7,stroke:#333,color:#fff
    style GPSDO fill:#2b8a3e,stroke:#333,color:#fff
```

---

### ⚡ Modification 1 — Gain filtre IF +3dB (R0x06)

**📁 Fichier** : `common/airspy_nos_conf.c` — Registre R820T `0x06`

```diff
- /* 06 */ 0x80,
+ /* 06 */ 0xA0, // F4TNK: FILT_GAIN=1 (+3dB filter gain for weak LEO sat signals)
```

#### 📐 Décomposition bit-à-bit

| Bit | Nom | Avant (0x80) | Après (0xA0) | Effet |
|-----|-----|:------------:|:------------:|-------|
| [7] | PWD_PDET1 | 1 (OFF) | 1 (OFF) | Détecteur puissance OFF ✅ inchangé |
| [6] | — | 0 | 0 | Réservé |
| **[5]** | **FILT_GAIN** | **0 (0 dB)** | **1 (+3 dB)** | **🔺 +3 dB gain filtre IF** |
| [4:3] | — | 00 | 00 | Réservé |
| [2:0] | PW_LNA | 000 (max) | 000 (max) | Puissance LNA max ✅ inchangé |

#### 🧠 Justification technique

```mermaid
flowchart LR
    A["🎛️ Filtre IF<br/>sortie"] --> B{"FILT_GAIN ?"}
    B -->|"0 (stock)"| C["0 dB<br/>Pas de boost"]
    B -->|"1 (F4TNK)"| D["✅ +3 dB<br/>Signal amplifié"]
    D --> E["📈 SNR amélioré<br/>pour < -130 dBm"]
    C --> F["❌ Signal faible<br/>proche plancher<br/>bruit ADC"]

    style D fill:#69db7c,stroke:#333,color:#000
    style F fill:#ff6b6b,stroke:#333,color:#fff
```

Le bit `FILT_GAIN` ajoute **+3 dB** au gain du filtre de canal IF, **avant le VGA**. Pour des signaux LEO à -130 dBm, cette amplification supplémentaire place le signal plus haut au-dessus du plancher de bruit de l'ADC 12 bits, améliorant directement le rapport signal/bruit numérique.

**🟢 Impact** : Aucun risque — ne modifie ni la linéarité, ni la stabilité.

---

### ⚡ Modification 2 — Courant mixer maximal (R0x07)

**📁 Fichier** : `common/airspy_nos_conf.c` — Registre R820T `0x07`

```diff
- /* 07 */ 0x60,
+ /* 07 */ 0x40, // F4TNK: PW0_MIX=0 (max mixer current, lower noise figure)
```

#### 📐 Décomposition bit-à-bit

| Bit | Nom | Avant (0x60) | Après (0x40) | Effet |
|-----|-----|:------------:|:------------:|-------|
| [6] | PWD_MIX | 1 (ON) | 1 (ON) | Mixer allumé ✅ |
| **[5]** | **PW0_MIX** | **1 (normal)** | **0 (MAX)** | **🔺 Courant mixer au max** |
| [4] | MIXGAIN_MODE | 1 (auto) | 0 (manual) | Mode gain mixer |
| [3:0] | MIX_GAIN | 0000 | 0000 | Gain mixer init ✅ |

#### 🧠 Justification technique

```mermaid
flowchart TD
    subgraph "⚠️ Courant normal (PW0_MIX=1)"
        N1["Courant réduit"] --> N2["NF mixer ~ 10 dB"]
        N2 --> N3["❌ Bruit thermique<br/>masque signaux LEO"]
    end

    subgraph "✅ Courant MAX (PW0_MIX=0)"
        M1["Courant maximum"] --> M2["NF mixer ~ 7-8 dB"]
        M2 --> M3["✅ Bruit thermique<br/>réduit de ~2-3 dB"]
    end

    style N3 fill:#ff6b6b,stroke:#333,color:#fff
    style M3 fill:#69db7c,stroke:#333,color:#000
```

Le mixer est le composant le plus critique après le LNA dans la chaîne de bruit de Friis. En augmentant son courant de bias, on réduit le bruit thermique du transistor du mixer. Le gain en noise figure est de l'ordre de **2-3 dB**, significatif pour des signaux à -130/-140 dBm.

**🟢 Impact** : Légère augmentation consommation (~5 mW). Aucun risque de dommage.

---

### ⚡ Modification 3 — Courant filtre IF maximal (R0x09)

**📁 Fichier** : `common/airspy_nos_conf.c` — Registre R820T `0x09`

```diff
- /* 09 */ 0x40, // Image Phase Adjustment
+ /* 09 */ 0x00, // F4TNK: PW1_IFFILT=0 (high current IF filter, lower noise)
```

#### 📐 Décomposition bit-à-bit

| Bit | Nom | Avant (0x40) | Après (0x00) | Effet |
|-----|-----|:------------:|:------------:|-------|
| [7] | PWD_IFFILT | 0 (ON) | 0 (ON) | Filtre IF allumé ✅ |
| **[6]** | **PW1_IFFILT** | **1 (faible)** | **0 (FORT)** | **🔺 Courant filtre IF max** |
| [5:0] | IMR_P | 000000 | 000000 | Phase calibration ✅ |

#### 🧠 Justification technique

Le filtre IF est un filtre actif à base d'op-amp intégré. En mode courant fort :

- **Meilleure réponse en fréquence** du filtre actif
- **Moins de bruit** ajouté par le filtre lui-même
- **Meilleure réjection hors-bande** grâce à des pentes plus raides

Particulièrement important pour les signaux LEO narrowband (15-150 kHz) qui doivent traverser un filtre IF propre.

**🟢 Impact** : Légère augmentation consommation. Aucun risque.

---

### ⚡ Modification 4 — Seuils AGC LNA relevés (R0x0D)

**📁 Fichier** : `common/airspy_nos_conf.c` — Registre R820T `0x0D`

```diff
- /* 0D */ 0x63, // LNA AGC settings
+ /* 0D */ 0x75, // F4TNK: LNA AGC thresholds raised (VTHH=7,VTHL=5)
```

#### 📐 Décomposition bit-à-bit

| Bit | Nom | Avant (0x63) | Après (0x75) | Effet |
|-----|-----|:------------:|:------------:|-------|
| **[7:4]** | **LNA_VTHH** | **6 (1.14V)** | **7 (1.24V)** | **🔺 Seuil haut AGC relevé** |
| **[3:0]** | **LNA_VTHL** | **3 (0.64V)** | **5 (0.84V)** | **🔺 Seuil bas AGC relevé** |

#### 🧠 Justification technique

```mermaid
flowchart TD
    SIG["🛰️ Signal LEO<br/>-130 dBm"] --> AGC{"AGC LNA<br/>compare au seuil"}

    AGC -->|"Stock: VTHH=6<br/>(1.14V)"| RED["⚠️ AGC réduit le gain<br/>trop tôt !<br/>Signal perdu"]
    AGC -->|"F4TNK: VTHH=7<br/>(1.24V)"| KEEP["✅ AGC maintient<br/>gain élevé<br/>Signal préservé"]

    RED --> BAD["❌ Décodage échoué"]
    KEEP --> GOOD["✅ Décodage réussi"]

    style RED fill:#ff6b6b,stroke:#333,color:#fff
    style KEEP fill:#69db7c,stroke:#333,color:#000
    style BAD fill:#c92a2a,stroke:#333,color:#fff
    style GOOD fill:#2b8a3e,stroke:#333,color:#fff
```

L'AGC du LNA compare le niveau de sortie à deux seuils :
- **VTHH** (seuil haut) : au-dessus → l'AGC réduit le gain
- **VTHL** (seuil bas) : en-dessous → l'AGC augmente le gain

En relevant ces seuils, l'AGC **tolère un niveau plus élevé** avant de réduire le gain. Pour des signaux LEO faibles, le LNA reste à gain élevé plus longtemps, sans risque de saturation (signaux à -130 dBm, loin de la saturation).

L'hystérésis augmentée réduit aussi les **oscillations d'AGC** (gain hunting) pendant un passage satellite.

**🟡 Impact** : Risque faible — en présence de signaux FM forts, le LNA pourrait saturer légèrement plus. Acceptable sur antenne satellite dédiée.

---

### ⚡ Modification 5 — Seuil AGC mixer relevé (R0x0E)

**📁 Fichier** : `common/airspy_nos_conf.c` — Registre R820T `0x0E`

```diff
- /* 0E */ 0x75,
+ /* 0E */ 0x85, // F4TNK: Mixer AGC VTH_H=8 (more gain before AGC reduces)
```

#### 📐 Décomposition bit-à-bit

| Bit | Nom | Avant (0x75) | Après (0x85) | Effet |
|-----|-----|:------------:|:------------:|-------|
| **[7:4]** | **MIX_VTH_H** | **7 (1.24V)** | **8 (1.34V)** | **🔺 Seuil haut mixer relevé** |
| [3:0] | MIX_VTH_L | 5 (0.84V) | 5 (0.84V) | Seuil bas ✅ inchangé |

#### 🧠 Justification technique

Même logique que la modification 4, appliquée à l'étage mixer. Le mixer AGC réduit le gain si le signal IF dépasse le seuil. En le relevant de 1.24V à 1.34V, le mixer reste à gain plus élevé pour les signaux faibles. **Complète la modification 4** : les deux étages AGC (LNA + mixer) sont coordonnés.

**🟡 Impact** : Risque faible — même logique que mod 4.

---

### ⚡ Modification 6 — TOP mixer plus élevé (R0x1C)

**📁 Fichier** : `common/airspy_nos_conf.c` — Registre R820T `0x1C`

```diff
- /* 1C */ 0x54,
+ /* 1C */ 0x24, // F4TNK: MIXER_TOP=2 (higher TOP → more gain before compression)
```

#### 📐 Décomposition bit-à-bit

| Bit | Nom | Avant (0x54) | Après (0x24) | Effet |
|-----|-----|:------------:|:------------:|-------|
| **[7:4]** | **MIXER_TOP** | **5 (moyen)** | **2 (élevé)** | **🔺 Point compression relevé** |
| [3:0] | Divers | 0x4 | 0x4 | ✅ Inchangé |

#### 🧠 Justification technique

```mermaid
graph LR
    subgraph "MIXER_TOP = Take-Off Point"
        direction TB
        T0["TOP=0 🟢<br/>Maximum gain<br/>avant compression"]
        T2["TOP=2 ✅<br/>F4TNK<br/>Gain élevé"]
        T5["TOP=5 ⚠️<br/>Stock<br/>Gain modéré"]
        T15["TOP=15 🔴<br/>Minimum<br/>Compression rapide"]
    end

    style T2 fill:#69db7c,stroke:#333,color:#000
    style T5 fill:#ffd43b,stroke:#333,color:#000
```

Le **TOP (Take-Off Point)** définit le niveau auquel le mixer commence à compresser. Valeur basse = plus de headroom. Avec signaux LEO à -130 dBm, la compression n'est **jamais** atteinte → on peut pousser le TOP.

**🟢 Impact** : Aucun risque pour signaux faibles.

---

### ⚡ Modification 7 — TOP LNA plus élevé (R0x1D)

**📁 Fichier** : `common/airspy_nos_conf.c` — Registre R820T `0x1D`

```diff
- /* 1D */ 0xAE,
+ /* 1D */ 0x8E, // F4TNK: LNA_TOP=1 (higher gain before AGC kicks in)
```

#### 📐 Décomposition bit-à-bit

| Bit | Nom | Avant (0xAE) | Après (0x8E) | Effet |
|-----|-----|:------------:|:------------:|-------|
| [7:6] | — | 10 | 10 | Réservé ✅ |
| **[5:3]** | **LNA_TOP** | **5 (moyen)** | **1 (élevé)** | **🔺 TOP LNA fortement relevé** |
| [2:0] | PDET2_GAIN | 110 | 110 | ✅ Inchangé |

Même logique que mod 6, appliquée au LNA. Échelle inversée : 0=max, 7=min. Passer de 5 à 1 donne bien plus de headroom au LNA.

**🟡 Impact** : Risque faible — avec signaux forts proches, le LNA pourrait saturer. Acceptable sur antenne sat.

---

### ⚡ Modification 8 — FILTER_EXT activé (R0x1E)

**📁 Fichier** : `common/airspy_nos_conf.c` — Registre R820T `0x1E`

```diff
- /* 1E */ 0x0A,
+ /* 1E */ 0x4A, // F4TNK: FILTER_EXT=1 (CRITICAL: enable IF filter extension!)
```

#### 📐 Décomposition bit-à-bit

| Bit | Nom | Avant (0x0A) | Après (0x4A) | Effet |
|-----|-----|:------------:|:------------:|-------|
| [7] | sw_pdect | 0 | 0 | ✅ |
| **[6]** | **FILTER_EXT** | **0 (OFF !)** | **1 (ON)** | **🔺🔺🔺 Extension filtre faible signal** |
| [5:0] | PDET_CLK | 001010 | 001010 | Timing ✅ |

#### 🧠 Justification technique

```mermaid
flowchart TD
    SIG["🛰️ Signal LEO faible<br/>-135 dBm, 40 kHz"] --> FILT{"Filtre IF R820T"}

    FILT --> OFF["❌ FILTER_EXT = 0<br/>(stock firmware)"]
    FILT --> ON["✅ FILTER_EXT = 1<br/>(F4TNK firmware)"]

    OFF --> CLIP["⚠️ Bords du signal<br/>coupés par le filtre<br/>si bande trop étroite"]
    ON --> ADAPT["🔄 Filtre détecte<br/>signal faible et<br/>ÉLARGIT sa bande<br/>automatiquement"]

    CLIP --> LOSS["❌ Perte de données<br/>aux extrémités spectre"]
    ADAPT --> SAVE["✅ Signal complet<br/>préservé"]

    style OFF fill:#ff6b6b,stroke:#333,color:#fff
    style ON fill:#69db7c,stroke:#333,color:#000
    style LOSS fill:#c92a2a,stroke:#333,color:#fff
    style SAVE fill:#2b8a3e,stroke:#333,color:#fff
```

> ⭐ **C'est la modification la plus critique de toutes.**

`FILTER_EXT` active une fonctionnalité interne du R820T2 qui **étend automatiquement la bande passante du filtre IF quand le signal est faible**. Cela empêche le filtre de couper un signal déjà au niveau du bruit.

**Ce bit était DÉSACTIVÉ dans le firmware stock de l'Airspy R2.** C'est exactement la fonctionnalité conçue pour les signaux faibles comme la réception satellite.

**🟢 Impact** : Aucun risque — fonctionnalité native du R820T2.

---

### ⚡ Modification 9 — OPTIM_SET_MUX (Makefile)

**📁 Fichier** : `common/Makefile_M0_inc.mk`

```diff
- AIRSPY_OPTS = -DLPC43XX -DLPC43XX_M0 -DCORE_M0 -D__CORTEX_M=0
+ AIRSPY_OPTS = -DLPC43XX -DLPC43XX_M0 -DCORE_M0 -D__CORTEX_M=0 -DOPTIM_SET_MUX
```

#### 🧠 Justification technique

```mermaid
sequenceDiagram
    participant HOST as 🖥️ PC (satnogs)
    participant M0 as 🔵 M0 (USB)
    participant R820T as 🎛️ R820T2 (I2C)

    Note over HOST,R820T: ⏱️ Correction Doppler toutes les ~100ms

    rect rgb(255, 230, 230)
        Note over HOST,R820T: ❌ SANS OPTIM_SET_MUX
        HOST->>M0: set_freq(435.100 MHz)
        M0->>R820T: I2C: R0x17 open_drain
        M0->>R820T: I2C: R0x1A rf_mux_poly
        M0->>R820T: I2C: R0x1B tracking_filter
        M0->>R820T: I2C: R0x10 xtal_cap
        M0->>R820T: I2C: R0x08 imr
        M0->>R820T: I2C: R0x09 imr
        M0->>R820T: I2C: PLL registers...
        Note right of R820T: 🐌 6+N transactions I2C<br/>~500 µs latence totale
    end

    rect rgb(230, 255, 230)
        Note over HOST,R820T: ✅ AVEC OPTIM_SET_MUX
        HOST->>M0: set_freq(435.105 MHz)
        Note over M0: Index bande = 17<br/>Identique au précédent !<br/>→ SKIP 6 registres
        M0->>R820T: I2C: PLL registers seulement
        Note right of R820T: 🚀 N transactions seulement<br/>~150 µs latence
    end
```

La fonction `r820t_set_tf()` dans `r820t.c` écrit **6 registres I2C** à chaque changement de fréquence pour reconfigurer le tracking filter et le MUX RF. Or ces registres ne changent que lors d'un changement de **bande** (ex: VHF → UHF).

Avec `OPTIM_SET_MUX`, le firmware **cache l'index de bande** et saute les 6 écritures I2C si la fréquence reste dans la même bande. Exactement le cas du suivi Doppler LEO : la fréquence varie de ±30 kHz autour d'une porteuse dans la bande 310-450 MHz.

**Résultat** : latence de re-tuning ÷ **~3** → suivi Doppler plus fluide et précis.

**🟢 Impact** : Aucun risque — optimisation cache I2C purement logicielle.

---

### ⚡ Modification 10 — CLK7 drive 8mA (SI5351C)

**📁 Fichier** : `common/airspy_nos_conf.c` — Table SI5351C, registre 23 (CLK7 control)

```diff
- 0x00, 0x00, 0x00, 0x6C, 0x00, ... // 20 - 29
+ 0x00, 0x00, 0x00, 0x6F, 0x00, ... // 20 - 29 (CLK7 drive 8mA)
```

#### 📐 Décomposition du registre 23 (CLK7_CTRL) du SI5351C

| Bit | Nom | Avant (0x6C) | Après (0x6F) | Effet |
|-----|-----|:------------:|:------------:|-------|
| [7] | CLK7_PDN | 0 (ON) | 0 (ON) | Active ✅ |
| [6] | MS7_SRC | 1 (PLL_B) | 1 (PLL_B) | Source ✅ |
| [5] | MS7_INT | 1 (Integer) | 1 (Integer) | Mode entier ✅ |
| [4:2] | CLK7 config | ... | ... | ✅ |
| **[1:0]** | **CLK7_IDRV** | **00 (2mA)** | **11 (8mA)** | **🔺 Courant ×4** |

#### 🧠 Justification technique

```mermaid
flowchart LR
    SI["🕐 SI5351C<br/>CLK7: 20 MHz"] -->|"Signal carré"| PIN["📌 GP_CLKIN<br/>LPC4370"]
    PIN --> ADC["📐 ADCHS<br/>12-bit ADC"]

    subgraph "📊 Impact courant de sortie"
        direction TB
        LOW["2mA (stock)<br/>⚠️ Fronts mous<br/>Jitter ~50 ps"]
        HIGH["8mA (F4TNK)<br/>✅ Fronts raides<br/>Jitter ~20 ps"]
    end

    style LOW fill:#ffd43b,stroke:#333,color:#000
    style HIGH fill:#69db7c,stroke:#333,color:#000
```

En mode **10 MSPS**, l'horloge GP_CLKIN (20 MHz) va **directement** à l'ADC sans PLL. La qualité de cette horloge détermine directement le jitter. Un courant 4× plus fort produit des fronts **4× plus raides** → moins de jitter → meilleure résolution effective (ENOB) → meilleur SNR.

**🟢 Impact** : +24 mW consommation SI5351C. Le SI5351C supporte 8mA par canal.

---

## �️ GPSDO — Référence externe 10 MHz

> 🔑 **L'Airspy R2 supporte nativement un GPSDO 10 MHz.** Le firmware détecte automatiquement sa présence au démarrage et bascule de configuration — **sans intervention utilisateur**.

### 📡 Connecteur CLK sur l'Airspy R2

```
┌─────────────────────────────────────────────────────────┐
│           VUE DESSUS — PCB Airspy R2                    │
│                                                         │
│  [ANT MCX]                              [USB Type B]    │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │                                                  │   │
│  │    [SI5351C]         [LPC4370]     [R820T2]      │   │
│  │                                                  │   │
│  │  ← Pad/Point de test CLK (CLKIN)                 │   │
│  │    Signal: 10 MHz CMOS 3.3V                       │   │
│  │    Plusieurs Airspy R2 ont un SMA ou pad nu       │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

> ⚠️ La position exacte du pad CLK varie selon la révision du PCB. Chercher le pad/SMA étiqueté **"CLK"** ou **"CLKIN"** sur le bord du PCB, proche du SI5351C.

### ⚙️ Architecture de détection automatique GPSDO

```mermaid
flowchart TD
    BOOT["⚡ Démarrage firmware"] --> INIT["SI5351C init\n(disable all, power down)"]
    INIT --> READ["📖 Lire reg0 SI5351C\n(CLKIN_LOS bit)"]

    READ --> CHECK{"CLKIN_LOS = 1 ?\n(pas de signal GPSDO)"}

    CHECK -->|"✅ GPSDO absent\n(LOS = 1)"| XTAL["🔧 Config XTAL\nTCXO 25 MHz\n(AIRSPY_SI5351C_CONFIG_XTAL)"]
    CHECK -->|"🛰️ GPSDO présent\n(LOS = 0)"| CLKIN["🛰️ Config CLKIN\nGPSDO 10 MHz\n(AIRSPY_SI5351C_CONFIG_CLKIN)"]

    XTAL --> CALIB{"Calibration flash\nvalide ?"}
    CALIB -->|"Oui"| PPB["📐 Appliquer\ncorrection PPB\nr820t xtal_freq"]
    CALIB -->|"Non"| NOPPB["25.000000 MHz\nnominal"]
    PPB --> CONT["➡️ Démarrage normal"]
    NOPPB --> CONT

    CLKIN --> NOCALIB["⚡ Pas de correction PPB\n(GPS est LA référence)\nr820t xtal_freq = 25 MHz"]
    NOCALIB --> CONT

    style CLKIN fill:#2b8a3e,stroke:#333,color:#fff
    style XTAL fill:#1971c2,stroke:#333,color:#fff
    style NOCALIB fill:#2b8a3e,stroke:#333,color:#fff
    style PPB fill:#1971c2,stroke:#333,color:#fff
```

### 📡 Chaîne d'horloges en mode GPSDO

```mermaid
flowchart LR
    GPS["🌍 GPS"] -->|"signal GPS"| GPSDO_MOD["📡 Module GPSDO"]
    GPSDO_MOD -->|"10 MHz ±0.01 ppb\nCMOS 3.3V"| CLKIN_PIN["📌 CLKIN\nSI5351C pin 6"]

    subgraph "🕐 SI5351C (mode CLKIN)"
        CLKIN_PIN -->|"CLKIN_DIV=1"| PLLA["🔄 PLL_A\n10×80=800 MHz"]
        CLKIN_PIN --> PLLB["🔄 PLL_B\n10×64=640 MHz"]
        PLLA -->|"÷32 integer"| CH0["📻 CLK0: 25 MHz\n→ R820T XTAL\n2mA drive"]
        PLLB -->|"÷32 integer"| CH7["📐 CLK7: 20 MHz\n→ LPC GP_CLKIN\n🔧 8mA drive (F4TNK Mod11)"]
    end

    subgraph "🔴 MCU LPC4370"
        CH7 --> ADC["ADCHS\n12-bit"]
        CH7 --> PLL1["PLL1 ×7\n= 140 MHz MCU"]
        CH7 --> PLLOUSB["PLL0USB\n= 480 MHz USB"]
    end

    CH0 --> R820T["🎛️ R820T2\nPLL VCO\n→ fréquence de réception"]

    style GPSDO_MOD fill:#2b8a3e,stroke:#333,color:#fff
    style CH7 fill:#4dabf7,stroke:#333,color:#fff
    style CH0 fill:#ff6b6b,stroke:#333,color:#fff
```

### 📊 Comparaison TCXO vs GPSDO

| Paramètre | TCXO 25 MHz (stock) | GPSDO 10 MHz (CLKIN) |
|-----------|:-------------------:|:--------------------:|
| Précision fréquence | ~1-2 ppm (±25 Hz à 25 MHz) | **< 0.01 ppb** (tracé GPS) |
| Maintien (holdover) | Dérive avec temp | TCXO interne au GPSDO |
| Bruit de phase PLL R820T | Ref. | **Meilleur** (source externe |
| Stabilité Doppler | Correction logicielle | **Correction hardware** |
| Usage calibration PPB | Obligatoire | **Inutile** (GPS discipline) |
| Correction freq auto | Non (firmware) | **Oui** (GPSDO verrouillé GPS) |

> 💡 **Cas d'usage GPSDO** : réception BPSK précise, mesure de fréquence satellite, systèmes multi-stations cohérentes (VLBI amateur), comparaison de fréquences.
> Pour la réception satellite APT/LRPT standard, le TCXO corrigé par PPB est suffisant.

### 🔧 GPSDOs compatibles testés

| Module GPSDO | Fréquence sortie | Niveau logique | Notes |
|-------------|:----------------:|:--------------:|-------|
| Leo Bodnar GPS-GPSDO | 10 MHz | 3.3V CMOS | ✅ Compatible direct |
| u-blox NEO-M8N + buffer | 10 MHz | 3.3V CMOS | ✅ Compatible |
| OCXO 10 MHz | 10 MHz | Sinusoïdal | ⚠️ Adapter niveau |
| Pendulum CNT-91 | 10 MHz | LVTTL | ✅ Compatible |

> ⚠️ Signal sinusoïdal : utiliser un buffer CMOS (74LV14 ou SN74LV1T34) pour adapter.

---

### ⚡ Modification 11 — CLK7 8mA en mode CLKIN (GPSDO)

**📁 Fichier** : `common/airspy_nos_conf.c` — Table SI5351C CLKIN, registre 23 (CLK7 control)

```diff
  /* Conf 1 (AIRSPY_SI5351C_CONFIG_CLKIN) AIRSPY_NOS CLKIN 10MHz CLK */
- 0x00, 0x00, 0x40, 0x6C, 0x00, ... // 20 - 29 (CLK7 drive 2mA)
+ 0x00, 0x00, 0x40, 0x6F, 0x00, ... // 20 - 29 (F4TNK MOD11: CLK7 drive 8mA)
```

#### 📐 Décomposition du registre 23 (CLK7_CTRL) — mode CLKIN

| Bit | Nom | Avant (0x6C) | Après (0x6F) | Effet |
|-----|-----|:------------:|:------------:|-------|
| [7] | CLK7_PDN | 0 (ON) | 0 (ON) | Active ✅ |
| [6] | MS7_SRC | 1 (PLL_B) | 1 (PLL_B) | Source PLL_B (640 MHz) ✅ |
| [5] | MS7_INT | 1 (Integer) | 1 (Integer) | Division entière ✅ |
| [4:2] | CLK7 multisynth | 011 | 011 | ✅ inchangé |
| **[1:0]** | **CLK7_IDRV** | **00 (2mA)** | **11 (8mA)** | **🔺 Courant sortie ×4** |

#### 🧠 Justification

Même logique que **Mod 10** (mode XTAL) : un courant plus faible en sortie CLK7 produit des **fronts d'horloge moins raides**, ce qui se traduit par plus de jitter sur le GP_CLKIN du LPC4370.

```mermaid
flowchart LR
    subgraph "GPSDO super-précis"
        GPS_SRC["🌍 ±0.01 ppb"] --> CLKIN_IN["CH7 = 20 MHz"]
    end
    CLKIN_IN --> DRIVE{"Drive CLK7"}
    DRIVE -->|"2mA (stock)"| SOFT["Fronts mous\nJitter ~50 ps\n⚠️ Annule l'avantage GPSDO"]
    DRIVE -->|"8mA (F4TNK)"| SHARP["Fronts raides\nJitter ~20 ps\n✅ Préserve la qualité GPSDO"]
    SOFT --> ADC_BAD["❌ ADC gâché par\nle jitter d'horloge"]
    SHARP --> ADC_GOOD["✅ ENOB maximal\nSNR optimal"]

    style SOFT fill:#ff6b6b,stroke:#333,color:#fff
    style SHARP fill:#69db7c,stroke:#333,color:#000
    style ADC_BAD fill:#c92a2a,stroke:#333,color:#fff
    style ADC_GOOD fill:#2b8a3e,stroke:#333,color:#fff
```

> ⭐ **Important** : Sans cette modification, même un GPSDO parfait aurait son avantage de phase noise annulé par le jitter d'interface vers l'ADC.

**🟢 Impact** : Aucun risque — même modification que Mod 10 pour le mode GPSDO.

---

## 📊 Tableau récapitulatif des 11 modifications

```mermaid
pie title Impact estimé des modifications sur le SNR
    "⭐ R0x1E FILTER_EXT" : 23
    "R0x09 IF current MAX" : 14
    "R0x07 Mixer current MAX" : 11
    "R0x06 +3dB IF gain" : 9
    "R0x0D LNA AGC thresholds" : 8
    "R0x0E Mixer AGC threshold" : 7
    "R0x1D LNA TOP" : 7
    "R0x1C Mixer TOP" : 5
    "OPTIM_SET_MUX Doppler" : 5
    "CLK7 8mA XTAL (Mod10)" : 5
    "CLK7 8mA CLKIN (Mod11)" : 6
```

| # | Fichier | Registre | Avant → Après | Description | Risque |
|:-:|---------|----------|:-------------:|-------------|:------:|
| 1 | `airspy_nos_conf.c` | R0x06 | `0x80` → `0xA0` | +3 dB gain filtre IF | 🟢 Nul |
| 2 | `airspy_nos_conf.c` | R0x07 | `0x60` → `0x40` | Courant mixer maximal | 🟢 Nul |
| 3 | `airspy_nos_conf.c` | R0x09 | `0x40` → `0x00` | Courant filtre IF maximal | 🟢 Nul |
| 4 | `airspy_nos_conf.c` | R0x0D | `0x63` → `0x75` | Seuils AGC LNA relevés | 🟡 Faible |
| 5 | `airspy_nos_conf.c` | R0x0E | `0x75` → `0x85` | Seuil AGC mixer relevé | 🟡 Faible |
| 6 | `airspy_nos_conf.c` | R0x1C | `0x54` → `0x24` | TOP mixer élevé | 🟢 Nul |
| 7 | `airspy_nos_conf.c` | R0x1D | `0xAE` → `0x8E` | TOP LNA élevé | 🟡 Faible |
| 8 | `airspy_nos_conf.c` | R0x1E | `0x0A` → `0x4A` | ⭐ FILTER_EXT activé | 🟢 Nul |
| 9 | `Makefile_M0_inc.mk` | -D flag | — | OPTIM_SET_MUX (Doppler) | 🟢 Nul |
| 10 | `airspy_nos_conf.c` | SI5351C reg23 XTAL | `0x6C` → `0x6F` | CLK7 drive 8mA (TCXO) | 🟢 Nul |
| 11 | `airspy_nos_conf.c` | SI5351C reg23 CLKIN | `0x6C` → `0x6F` | ⭐ CLK7 8mA (GPSDO) | 🟢 Nul |

> 🟢 = Aucun risque | 🟡 = Risque faible (réversible par reflash)

> 🛰️ **Mod 11 s'applique automatiquement si un GPSDO 10 MHz est connecté au pad CLK du PCB.** Sans GPSDO, seules les Mods 1-10 sont actives.

---

## 🔬 Analyse ADC / DMA / Horloge

### 📐 Configuration ADCHS — Inchangée (déjà optimale)

```mermaid
flowchart LR
    CLK["🕐 20 MHz<br/>GP_CLKIN"] --> ADC["📐 ADCHS"]
    ADC --> FIFO["📦 FIFO<br/>Level = 8"]
    FIFO --> DMA["💾 DMA Ch0"]
    DMA --> BUF["🗃️ Buffer 32 KB<br/>@ 0x20004000"]

    subgraph "🔄 DMA Round-Robin (4 LLI)"
        L0["LLI 0<br/>8 KB"] --> L1["LLI 1<br/>8 KB"]
        L1 --> L2["LLI 2<br/>8 KB"]
        L2 --> L3["LLI 3<br/>8 KB"]
        L3 --> L0
    end

    BUF --> M4["🔴 M4 Core<br/>Pack 12→compressed<br/>(ASM ARM)"]
    M4 --> USB["🔌 USB Bulk"]
```

| Paramètre | Valeur | Analyse |
|-----------|--------|---------|
| CONFIG.DGEC | 0x90 (144) | Correction gain/erreur digitale — ✅ optimal |
| POWER_CONTROL.CRS | 0x3 | Current Reduction Setting — ✅ standard |
| ADC_SPEED | 0x0 | Vitesse max — ✅ approprié ≤20 MSPS |
| FIFO_LEVEL | 8 | Bon compromis latence/overhead IRQ — ✅ |
| DMA LLI | 4 round-robin | Flux continu garanti — ✅ |
| Buffer | 32 KB double-buffer | Pack pendant DMA écrit — ✅ |

**Verdict** : La config ADC est déjà bien optimisée par l'équipe AirSpy. Aucune modification nécessaire.

### 🕐 Configuration SI5351C — Quasi-optimale

```mermaid
flowchart LR
    XTAL["💎 TCXO<br/>25 MHz"] --> PLLB["🔄 PLL_B<br/>25 × 32 = 800 MHz"]
    XTAL -->|"Direct passthrough<br/>(pas de PLL!)"| CH0["📻 CH0: 25 MHz<br/>→ R820T XTAL"]
    PLLB --> DIV["÷ 40<br/>(integer mode)"]
    DIV --> CH7["📐 CH7: 20 MHz<br/>→ LPC GP_CLKIN<br/>🔧 Drive: 8mA (F4TNK)"]

    style XTAL fill:#ffd43b,stroke:#333,color:#000
    style CH0 fill:#ff6b6b,stroke:#333,color:#fff
    style CH7 fill:#4dabf7,stroke:#333,color:#fff
```

| Paramètre | Valeur | Analyse |
|-----------|--------|---------|
| PLL_B | 25 MHz × 32 = 800 MHz | Mode entier → bruit de phase min ✅ |
| CH0 | 25 MHz TCXO direct | Pas de PLL pour R820T → NF min ✅ |
| CH7 | 800/40 = 20 MHz | Division entière → bruit min ✅ |
| CH7 drive | ~~2mA~~ → **8mA** | 🔧 Modifié (mod 10) → jitter ADC réduit |

---

## 🧪 Configuration recommandée pour LEO

### 🛰️ Configuration SatNOGS / GNURadio

```
┌──────────────────────────────────────────────────────────────┐
│  📻 Airspy R2 avec firmware F4TNK                           │
│                                                              │
│  Sample Rate : 10 MSPS (default 0)                          │
│  → puis décimation côté host à ~50-200 ksps                 │
│                                                              │
│  Gains recommandés (via airspy_rx ou API) :                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Mode Sensitivity = 18-20 (sur 21)                     │  │
│  │  ou mode manuel :                                      │  │
│  │   LNA Gain  = 12-14  (max 14)                         │  │
│  │   Mixer Gain = 10-12  (max 15)                         │  │
│  │   VGA Gain  = 8-10   (max 15)                          │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Bias-T : Activer si LNA externe (airspy_set_rf_bias 1)     │
│  Packing : Activer (réduit bande passante USB de 33%)        │
└──────────────────────────────────────────────────────────────┘
```

### 💡 Pourquoi 10 MSPS et pas 2.5 MSPS pour LEO ?

```mermaid
graph TD
    subgraph "📊 10 MSPS"
        A1["🕐 GP_CLKIN direct<br/>20 MHz, pas de diviseur"]
        A2["🔢 Oversampling: 200×<br/>(10M / 50k)"]
        A3["📈 Processing gain:<br/>10·log₁₀(200) = 23 dB"]
        A1 --> A2 --> A3
    end

    subgraph "📊 2.5 MSPS"
        B1["🕐 GP_CLKIN ÷ 4<br/>IDIVB ajoute du jitter"]
        B2["🔢 Oversampling: 50×<br/>(2.5M / 50k)"]
        B3["📈 Processing gain:<br/>10·log₁₀(50) = 17 dB"]
        B1 --> B2 --> B3
    end

    A3 --> DIFF["✅ 10 MSPS = +6 dB<br/>de processing gain<br/>après décimation"]

    style A3 fill:#69db7c,stroke:#333,color:#000
    style B3 fill:#ffd43b,stroke:#333,color:#000
    style DIFF fill:#2b8a3e,stroke:#333,color:#fff
```

---

## 🆘 Procédure de récupération DFU

### ⚠️ À LIRE AVANT TOUTE MODIFICATION FIRMWARE

> 🔒 **L'Airspy R2 NE PEUT PAS être brick de façon permanente.** Le bootloader est en ROM (non effaçable).

```mermaid
flowchart TD
    START["🔧 Problème firmware ?"] --> Q1{"L'Airspy<br/>est reconnu<br/>en USB ?"}

    Q1 -->|"✅ Oui"| NORMAL["📌 Récupération normale"]
    Q1 -->|"❌ Non"| DFU["🆘 Récupération DFU"]

    NORMAL --> CMD1["$ airspy_spiflash -w airspy_rom_to_ram/airspy_rom_to_ram.bin"]
    CMD1 --> DONE["✅ Firmware restauré !"]

    DFU --> STEP1["1️⃣ Ouvrir le boîtier<br/>(4 vis)"]
    STEP1 --> STEP2["2️⃣ Jumper P5 →<br/>position 1-2"]
    STEP2 --> STEP3["3️⃣ Brancher USB"]
    STEP3 --> STEP4["4️⃣ $ dfu-util -d 1fc9:000c<br/>-D firmware.bin"]
    STEP4 --> STEP5["5️⃣ Jumper P5 →<br/>position 2-3"]
    STEP5 --> STEP6["6️⃣ $ airspy_spiflash -w<br/>airspy_rom_to_ram/airspy_rom_to_ram.bin"]
    STEP6 --> DONE

    style DONE fill:#2b8a3e,stroke:#333,color:#fff
    style DFU fill:#e03131,stroke:#333,color:#fff
    style NORMAL fill:#1971c2,stroke:#333,color:#fff
```

### 📍 Positions du jumper P5

| Position | Mode | Usage |
|----------|------|-------|
| **1-2** | Boot USB0 (DFU) | 🆘 Récupération d'urgence |
| **2-3** | Boot SPIFI (Flash) | 📌 Fonctionnement normal |

### 📋 Commandes de récupération

```bash
# 1. Récupération normale (Airspy reconnu en USB)
airspy_spiflash -w ~/dev/airspyone_firmware/airspy_rom_to_ram/airspy_rom_to_ram.bin

# 2. Récupération DFU (Airspy non reconnu — jumper P5 en 1-2)
dfu-util -d 1fc9:000c -a 0 -D ~/dev/airspyone_firmware/airspy_rom_to_ram/airspy_rom_to_ram.bin

# 3. Vérification après récupération
airspy_info
# → Firmware Version: AirSpy NOS <git-tag> <date>
```

---

## 📚 Annexe — Carte des registres R820T2

### 📋 Registres modifiés vs inchangés (Regs 5-31)

| Reg | Bits | Nom | Description | Stock | F4TNK | Modifié ? |
|:---:|------|-----|-------------|:-----:|:-----:|:---------:|
| 0x05 | [4] | LNA_GAIN_MODE | 0=auto, 1=manual | 0x90 | 0x90 | — |
| **0x06** | **[5]** | **FILT_GAIN** | **0=0dB, 1=+3dB** | **0x80** | **0xA0** | ⚡ |
| **0x07** | **[5]** | **PW0_MIX** | **0=max, 1=normal** | **0x60** | **0x40** | ⚡ |
| 0x08 | [7:6] | PWD/PW0_AMP | Buffer mixer power | 0x80 | 0x80 | — |
| **0x09** | **[6]** | **PW1_IFFILT** | **0=high, 1=low** | **0x40** | **0x00** | ⚡ |
| 0x0A | [3:0] | FILT_CODE | 0=widest, 15=narrow | 0xA8 | 0xA8 | — |
| 0x0B | [7:5] | FILT_BW | 000=wide, 111=narrow | 0x0F | 0x0F | — |
| 0x0C | [3:0] | VGA_CODE | -12 à +40.5 dB | 0x40 | 0x40 | — |
| **0x0D** | **[7:4][3:0]** | **LNA_VTH H/L** | **Seuils AGC LNA** | **0x63** | **0x75** | ⚡ |
| **0x0E** | **[7:4][3:0]** | **MIX_VTH H/L** | **Seuils AGC mixer** | **0x75** | **0x85** | ⚡ |
| 0x0F | [7] | FLT_EXT_WIDEST | Extension max | 0xF8 | 0xF8 | — |
| 0x10 | — | — | PLL divider | 0x7C | 0x7C | — |
| 0x11 | — | PW_LDO_A | PLL LDO 2.1V | 0x42 | 0x42 | — |
| **0x1C** | **[7:4]** | **MIXER_TOP** | **0=max, 15=min** | **0x54** | **0x24** | ⚡ |
| **0x1D** | **[5:3]** | **LNA_TOP** | **0=max, 7=min** | **0xAE** | **0x8E** | ⚡ |
| **0x1E** | **[6]** | **FILTER_EXT** | **⭐ Weak signal ext** | **0x0A** | **0x4A** | ⚡ |
| 0x1F | [7] | LT_ATT | Loop-through atten | 0xC0 | 0xC0 | — |

### 🧮 Registres NON modifiés — Justification

| Reg | Raison |
|-----|--------|
| 0x08 | PWD_AMP=1, PW0_AMP=0 **déjà optimal** (courant max buffer mixer) |
| 0x0A, 0x0B | Filtre IF contrôlé dynamiquement par `r820t_set_if_bandwidth()` — ne pas toucher |
| 0x0F | `FLT_EXT_WIDEST=1` **déjà activé** |
| 0x10-0x16 | Registres PLL — calculés par `r820t_set_pll()` — **NE JAMAIS MODIFIER** |
| 0x11 | LDO 2.1V — baisser augmente le bruit de phase PLL |
| 0x1F | Loop-through désactivé — correct en réception seule |

### 🔗 Chaîne de bruit de Friis

$$NF_{total} = NF_{LNA} + \frac{NF_{Mixer} - 1}{G_{LNA}} + \frac{NF_{IF} - 1}{G_{LNA} \cdot G_{Mixer}} + \frac{NF_{VGA} - 1}{G_{LNA} \cdot G_{Mixer} \cdot G_{IF}}$$

Nos modifications réduisent $NF_{Mixer}$, $NF_{IF}$ et augmentent $G_{LNA}$, $G_{Mixer}$, $G_{IF}$ — tous les termes contribuent à un $NF_{total}$ plus faible.

---

### 🏗️ Compilation et flash

#### Prérequis (Debian Trixie / Ubuntu 24+)

```bash
# Toolchain ARM + outils Python/git
sudo apt install gcc-arm-none-eabi python3-git xxd airspy

# Debian Trixie n'a pas 'python', seulement python3
sudo ln -sf /usr/bin/python3 /usr/local/bin/python
```

> ⚠️ **GCC 14 + LTO** : Le firmware contient un correctif pour la double définition de `set_freq_params_t`  
> (supprimée de `airspy_m0.c`, gardée dans `airspy_usb_req.c`). Sans ce fix, GCC 14 refuse de linker.

#### Compilation

```bash
cd ~/dev/airspyone_firmware

# 1. Compiler libopencm3 (obligatoire — fournit les .ld et .a)
cd libopencm3
make TARGETS="lpc43xx/m0 lpc43xx/m0s lpc43xx/m4"
cd ..

# 2. Compiler le firmware F4TNK
make
# → Produit : airspy_rom_to_ram/airspy_rom_to_ram.bin
```

#### Flash (mode normal — Airspy reconnu en USB)

```bash
# Arrêter tout process qui monopolise l'Airspy (ex: Docker)
cd ~/station-3762 && docker compose down

# Flasher
airspy_spiflash -w ~/dev/airspyone_firmware/airspy_rom_to_ram/airspy_rom_to_ram.bin

# Power-cycle physique (débrancher/rebrancher USB)
# Puis vérifier — la version doit afficher le tag git F4TNK
airspy_info
# → Firmware Version: AirSpy NOS <git-tag> <date>

# Relancer Docker
cd ~/station-3762 && docker compose up -d
```

---

> 📝 *Document créé par F4TNK — Analyse approfondie du firmware Airspy R2 pour optimisation satellite LEO*
>
> 🔒 *Toutes les modifications sont réversibles via `airspy_spiflash -w` ou récupération DFU (jumper P5)*
>
> 🗓️ *Dernière mise à jour : 17 Février 2026 — Firmware compilé et flashé avec GCC 14.2 (Debian Trixie)*
