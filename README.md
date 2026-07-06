# SpatialAnchors

Configuration centralisée des **ancres spatiales Meta (Quest)** par salle physique.  
Les jeux Unity (ex. *Le Serviteur*) récupèrent les UUID à distance **sans rebuild**.

---

## Sommaire

1. [Principe général](#1-principe-général)
2. [Fichier `anchors.json`](#2-fichier-anchorsjson)
3. [Quelle salle charger ? (`AnchorRoomKey`)](#3-quelle-salle-charger-anchorroomkey)
4. [Cascade UUID — toutes les éventualités](#4-cascade-uuid--toutes-les-éventualités)
5. [Chargement Meta Cloud](#5-chargement-meta-cloud)
6. [Rôles GM vs Client (réseau NGO)](#6-rôles-gm-vs-client-réseau-ngo)
7. [Scénario : GitHub indisponible](#7-scénario--github-indisponible)
8. [Scénario : recalibration d'urgence par le GM](#8-scénario--recalibration-durgence-par-le-gm)
9. [Ajouter une nouvelle salle](#9-ajouter-une-nouvelle-salle)
10. [Fichier local `clientLocalSettings.json`](#10-fichier-local-clientlocalsettingsjson)
11. [Mise à jour après calibration](#11-mise-à-jour-après-calibration)
12. [Dépannage](#12-dépannage)
13. [Référence technique Unity](#13-référence-technique-unity)

---

## 1. Principe général

```mermaid
flowchart LR
    A["Calibration Quest\n(salle physique)"] --> B["Sauvegarde\nMeta Cloud"]
    B --> C["UUID copié dans\nanchors.json (GitHub)"]
    C --> D["Runtime Unity\n(fetch GitHub)"]
    D --> E["Load Meta Cloud\n+ alignement joueur"]
```

| Étape | Qui | Où |
|-------|-----|-----|
| Calibration | Opérateur / GM sur Quest | Salle physique |
| UUID officiel | Mainteneur | Ce repo GitHub |
| Chargement runtime | Chaque casque | Automatique au lancement |

**Règle d'or :** 1 UUID = 1 salle physique. Partagé entre tous les jeux du même lieu.

---

## 2. Fichier `anchors.json`

### URL raw (Unity)

```
https://raw.githubusercontent.com/Genie-Culturel/SpatialAnchors/main/anchors.json
```

> Le loader Unity convertit automatiquement cette URL vers l'**API GitHub Contents** (anti-cache CDN).

### Format actuel

```json
{
  "RoomA": "cfe69293-ac2f-4707-2eb5-1f8f553ea52b",
  "RoomB": "36d5b466-659d-e873-6f57-79ccc0e3eead",
  "RoomDev": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "updatedAt": "2026-06-16"
}
```

| Champ | Description |
|-------|-------------|
| `RoomA`, `RoomB`, … | UUID de l'ancre **Meta Cloud** pour cette salle |
| `updatedAt` | Date de dernière modification (traçabilité) |

### Clés supportées

| Clé JSON | Alias legacy | Usage |
|----------|--------------|-------|
| `RoomA` | `SalleA` | Salle physique A |
| `RoomB` | `SalleB` | Salle physique B |
| `RoomDev` | — | Salle de développement / test |
| `RoomStudio`, etc. | — | Toute clé `Lettre + alphanumérique` |

> Les nouvelles salles s'ajoutent dans le JSON **sans modifier le code Unity**, tant que le nom respecte le format (`RoomStudio`, `RoomTest1`…).

---

## 3. Deux questions distinctes (ne pas confondre)

Le système répond à **deux questions différentes** :

| Question | Champ concerné | Quand |
|----------|----------------|-------|
| **Quelle salle ?** (nom de la clé JSON) | `AnchorRoomKey` | Avant le fetch GitHub |
| **Quel UUID ?** (identifiant Meta) | `AnchorUuid` + cascade | Si GitHub/Meta échoue |

> ⚠️ **`AnchorRoomKey` invalide ≠ passage immédiat au legacy UUID.**  
> Si la clé est absente ou invalide, le jeu choisit un **nom de salle de secours** (Inspector → `RoomB`), puis **tente quand même GitHub** avec ce nom.  
> Le legacy `AnchorUuid` du GM n'intervient qu'**après un échec du fetch GitHub** (voir §4).

### 3.1 — Quelle salle charger ? (`AnchorRoomKey`)

Étape **A** : choisir le **nom de la clé** dans `anchors.json`.

```mermaid
flowchart TD
    START["Étape A — Choisir la clé JSON"] --> LS{"AnchorRoomKey\nvalide dans\nclientLocalSettings ?"}
    LS -->|Oui| USE["Clé = AnchorRoomKey\n(ex: RoomA, RoomDev)"]
    LS -->|Non ou invalide| FB["Nom de salle fallback\n(Inspector Unity :\nfallbackRoomKeyIfJsonMissing)"]
    FB --> DEF["Défaut dur : RoomB"]
    USE --> GH["→ Étape B : fetch GitHub\navec ce nom de clé"]
    DEF --> GH

    style GH fill:#e8f5e9
```

> Ce schéma ne concerne **que le nom de la salle**, pas l'UUID.  
> `AnchorUuid` (legacy GM) n'est **pas** lu à cette étape.

### 3.2 — Exemple `clientLocalSettings.json` (sur le Quest)

```json
{
  "AnchorRoomKey": "RoomA",
  "AnchorUuid": "cfe69293-ac2f-4707-2eb5-1f8f553ea52b"
}
```

| Champ | Rôle | Utilisé quand |
|-------|------|---------------|
| `AnchorRoomKey` | **Nom** de la salle → clé dans `anchors.json` | Étape A (toujours en premier) |
| `AnchorUuid` | **UUID** de secours (legacy GM) | Étape B — seulement si GitHub fetch échoue |

### 3.3 — Schéma complet (étapes A + B)

```mermaid
flowchart TD
    A["A — AnchorRoomKey valide ?"]
    A -->|Oui| KEY["Clé = AnchorRoomKey"]
    A -->|Non| KEYFB["Clé = fallback Inspector\n(défaut RoomB)"]

    KEY --> B["B — Fetch GitHub\nanchors.json[clé]"]
    KEYFB --> B

    B -->|UUID reçu| META["Charger Meta Cloud"]
    B -->|GitHub KO| LEG["C — Legacy UUID"]

    LEG --> GM{"Rôle ?"}
    GM -->|GM / Host| CLS["AnchorUuid\ndu GM\n(clientLocalSettings)"]
    GM -->|Client connecté| NGO["UUID session NGO\n(partagé par le GM)"]
    GM -->|Player anchor| PP["PlayerPrefs"]

    CLS --> META
    NGO --> META
    PP --> META

    LEG -->|Legacy vide| INS["D — Fallback Inspector\n(UUID compilé par salle)"]
    INS --> META

    META -->|OK| OK["Ancre assignée ✓"]
    META -->|KO| RETRY["Retenter Legacy\npuis Inspector"]
```

---

## 4. Cascade UUID — toutes les éventualités

Ordre **strict** pour obtenir un UUID, avant même d'interroger Meta :

```mermaid
flowchart TD
    START["LoadAnchor()"] --> G["① GitHub\nanchors.json"]
    G -->|UUID trouvé| META["Charger Meta Cloud"]
    G -->|Échec fetch| L["② Legacy\nancien système"]
    L -->|UUID trouvé| META
    L -->|Échec| I["③ Fallback Inspector\n(UUID compilé dans le build)"]
    I -->|UUID trouvé| META
    I -->|Échec| ERR["Erreur : aucun UUID disponible"]

    META -->|Succès| OK["Ancre assignée ✓\nAlignement joueur"]
    META -->|Échec| RETRY["Retenter Legacy\npuis Inspector\n(autres UUID)"]
    RETRY -->|Succès| OK
    RETRY -->|Échec| FAIL["Meta : ancre introuvable"]
```

### Tableau récapitulatif

| Étape | Source | Quand ça s'active |
|-------|--------|-------------------|
| **① GitHub** | `anchors.json` (ce repo) | Toujours en premier (mode remote config) |
| **② Legacy** | Voir §6 selon rôle GM/Client | GitHub KO ou Meta KO |
| **③ Inspector** | UUID compilé dans le build Unity | GitHub + Legacy KO |

### Sources Legacy (étape ②)

```mermaid
flowchart TD
    LEG["② Legacy"] --> PA{"Mode player anchor ?"}
    PA -->|Oui| PP["PlayerPrefs\n(SelfAnchorUuid)"]
    PA -->|Non| ROLE{"Rôle réseau ?"}
    ROLE -->|Client connecté| NGO["Session NGO\n(UUID partagé par le GM)"]
    ROLE -->|GM / Host / hors ligne| CLS["clientLocalSettings.json\n(champ AnchorUuid)"]
```

> **Attention :** le legacy lit `AnchorUuid` (un seul UUID), **pas** `AnchorRoomKey`.  
> Si tu changes de salle sans recalibrer, l'UUID legacy peut être obsolète.

---

## 5. Chargement Meta Cloud

Une fois l'UUID résolu (quelle que soit la source) :

```mermaid
sequenceDiagram
    participant Q as Quest
    participant GH as GitHub
    participant M as Meta Cloud

    Q->>GH: Fetch anchors.json (clé = AnchorRoomKey)
    GH-->>Q: UUID
    Q->>M: LoadUnboundAnchorsAsync(UUID)
    alt Ancre trouvée + bonne salle physique
        M-->>Q: Ancre localisée
        Q->>Q: Bind + alignement joueur
        Q->>Q: Sauvegarde AnchorUuid en local
    else UUID inconnu de Meta
        M-->>Q: Query failed / no anchors
        Q->>Q: Cascade Legacy → Inspector
    else Mauvaise salle physique
        M-->>Q: Échec localisation
        Q->>Q: Message "êtes-vous dans la bonne salle ?"
    end
```

### Cas fréquents d'échec Meta

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `Query failed` / `no anchors` | UUID absent de Meta Cloud | Recalibrer + mettre à jour `anchors.json` |
| UUID GitHub OK mais Meta KO | Mauvaise salle physique | Se placer dans la bonne salle |
| OK en éditeur PC fetch, Meta KO | Normal — Meta non supporté sur PC | Tester sur Quest en salle |

---

## 6. Rôles GM vs Client (réseau NGO)

```mermaid
flowchart TB
    subgraph GM["GM / Host"]
        G1["Fetch GitHub\n(AnchorRoomKey local)"]
        G2["Charge Meta Cloud"]
        G3["Partage UUID\nen session NGO"]
        G1 --> G2 --> G3
    end

    subgraph CLIENT["Client connecté"]
        C1["Fetch GitHub\n(son AnchorRoomKey)"]
        C2["Charge Meta Cloud"]
        C1 --> C2
    end

    subgraph FALLBACK["Si GitHub KO"]
        FG["GM : AnchorUuid\nlocal settings"]
        FC["Client : UUID session NGO\n(du GM)"]
    end

    G3 -.->|legacy secours| FC
```

| Rôle | Mode normal (GitHub OK) | Secours (GitHub KO) |
|------|-------------------------|---------------------|
| **GM** | Fetch GitHub avec son `AnchorRoomKey` | `AnchorUuid` dans son `clientLocalSettings` |
| **Client** | Fetch GitHub avec son `AnchorRoomKey` | UUID de la **session NGO** (partagé par le GM) |

> En mode remote config, **chaque casque** tente GitHub indépendamment.  
> La session NGO sert surtout de **filet de sécurité** si GitHub est down.

---

## 7. Scénario : GitHub indisponible

```mermaid
flowchart TD
    GH["GitHub KO"] --> GM["GM"]
    GH --> CL["Clients"]

    GM --> GML["Legacy : AnchorUuid\n(clientLocalSettings GM)"]
    GML --> GMM["Meta Cloud"]
    GML -->|vide| GMI["Fallback Inspector\n(RoomA/B/Dev compilé)"]

    CL --> CLL["Legacy : UUID session NGO"]
    CLL --> CLM["Meta Cloud"]
    CLL -->|session vide| CLI["Fallback Inspector"]

    GMM --> OK["Jeu continue si ancre valide"]
    CLM --> OK
```

### RoomA / RoomB encore nécessaires ?

| Situation | Besoin de RoomA/B |
|-----------|-------------------|
| GitHub OK (normal) | **Oui** — `AnchorRoomKey` choisit l'entrée JSON |
| GitHub KO + GM a un UUID local valide | **Non** — le GM est la source de vérité |
| GitHub KO + pas d'UUID legacy | **Oui** — fallback Inspector par salle |

---

## 8. Scénario : recalibration d'urgence par le GM

Quand GitHub est down et l'ancre est perdue :

```mermaid
sequenceDiagram
    participant GM as GM (Quest)
    participant M as Meta Cloud
    participant LS as clientLocalSettings
    participant NGO as Session NGO
    participant CL as Clients

    GM->>GM: Recrée / place l'ancre
    GM->>M: SaveAnchor → Cloud
    M-->>GM: Nouvel UUID
    GM->>LS: Sauvegarde AnchorUuid
    GM->>NGO: SetAnchorUuidToRoom(UUID)
    CL->>NGO: Legacy (GitHub KO)
    NGO-->>CL: Nouvel UUID du GM
    CL->>M: Charge l'ancre
    Note over CL: Recharger (touche R)\nou reconnecter si besoin
```

**Étapes opérateur :**

1. GM recalibre l'ancre en salle
2. Nouvel UUID → `clientLocalSettings` du GM + session NGO
3. Clients rechargent l'ancre (**R** ou reconnexion)
4. Quand GitHub revient → mettre à jour `anchors.json` avec le nouvel UUID

---

## 9. Ajouter une nouvelle salle

```mermaid
flowchart LR
    A["1. Calibrer sur Quest\n(salle physique)"] --> B["2. Ajouter clé + UUID\ndans anchors.json"]
    B --> C["3. Commit GitHub"]
    C --> D["4. Sur chaque Quest :\nAnchorRoomKey = nouvelle clé"]
    D --> E["5. Relancer ou touche R"]
```

### Exemple : ajouter `RoomStudio`

**GitHub (`anchors.json`) :**
```json
"RoomStudio": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
```

**Quest (`clientLocalSettings.json`) :**
```json
"AnchorRoomKey": "RoomStudio"
```

> Pas de rebuild nécessaire si le jeu supporte déjà le remote config (LeServiteur oui).

---

## 10. Fichier local `clientLocalSettings.json`

Emplacement sur Quest : `Application.persistentDataPath/clientLocalSettings.json`

| Champ | Utilisé pour | Exemple |
|-------|--------------|---------|
| `AnchorRoomKey` | Choisir l'entrée GitHub | `"RoomA"` |
| `AnchorUuid` | Legacy GM + cache après bind | `"cfe69293-..."` |

```mermaid
flowchart LR
    RK["AnchorRoomKey"] --> GH["→ GitHub\n(quelle salle)"]
    AU["AnchorUuid"] --> LG["→ Legacy GM\n(secours)"]
    AU --> NGO["→ Session NGO\n(partage clients)"]
```

---

## 11. Mise à jour après calibration

### Procédure standard (GitHub disponible)

1. Calibrer l'ancre sur Quest (outil ou GM in-game)
2. Noter l'UUID affiché dans les logs
3. Modifier `anchors.json` — remplacer l'UUID de la salle concernée
4. Mettre à jour `updatedAt`
5. Commit + push sur `main`
6. Sur les Quest : **touche R** ou relance (pas de rebuild)

### Vérification rapide

- Overlay debug : `Source active: GitHub` (vert)
- `UUID utilisé` = celui dans `anchors.json`
- `Ancre Meta assignée : Oui` (sur Quest en salle)

---

## 12. Dépannage

| Problème | Diagnostic | Action |
|----------|------------|--------|
| Bon UUID GitHub, Meta KO | UUID mort ou mauvaise salle | Recalibrer + mettre à jour JSON |
| `Room: RoomB` alors que `AnchorRoomKey: RoomDev` | Clé invalide ou absente | Vérifier orthographe exacte de la clé |
| Client legacy ✗ rouge | Session NGO sans UUID ou différent du GM | GM reconnecte + partage UUID |
| Fetch OK, bind non | Éditeur PC | Normal — tester sur Quest |
| Ancienne UUID après changement salle | `AnchorUuid` local obsolète | Recalibrer ou vider `AnchorUuid` |

### Arbre de décision rapide

```mermaid
flowchart TD
    P["Problème ancre"] --> Q1{"UUID GitHub\naffiché correct ?"}
    Q1 -->|Non| FIX1["Vérifier AnchorRoomKey\n+ anchors.json"]
    Q1 -->|Oui| Q2{"Meta assignée ?"}
    Q2 -->|Oui| OK["OK"]
    Q2 -->|Non| Q3{"Dans la bonne\nsalle physique ?"}
    Q3 -->|Non| FIX2["Se déplacer en salle"]
    Q3 -->|Oui| FIX3["Recalibrer +\nmettre à jour UUID GitHub"]
```

---

## 13. Référence technique Unity

Implémentation de référence : projet **LeServiteur**

| Fichier | Rôle |
|---------|------|
| `AnchorRemoteConfigLoader.cs` | Fetch GitHub + parsing JSON |
| `AnchorManager.cs` | Cascade + Meta Cloud + debug overlay |
| `NetworkManagerMH_NGO.cs` | `AnchorRoomKey` / `AnchorUuid` local |

### Overlay debug (Quest / éditeur)

- Panneau **Anchor Remote Config** : cascade, UUID, source active
- Touche **R** : recharger sans rebuild
- Bouton **Ouvrir config GitHub** : lien vers ce repo

### Anti double-chargement

Si `useRemoteAnchorConfig` est actif, les clients NGO **ne relancent pas** un second `LoadAnchor()` réseau — le chargement auto au démarrage suffit.

---

## Jeux connectés

| Jeu | Statut |
|-----|--------|
| Le Serviteur (LeServiteur) | Intégré |

---

*Dernière mise à jour de la doc : juillet 2026*
