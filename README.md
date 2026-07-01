# SpatialAnchors

Configuration centralisée des ancres spatiales Meta par salle, consommée à distance par les jeux Unity (Quest).

## Fichier `anchors.json`

| Champ | Description |
|---|---|
| `RoomA` | UUID de l'ancre cloud Meta — Room A |
| `RoomB` | UUID de l'ancre cloud Meta — Room B |
| `updatedAt` | Date de dernière mise à jour (format `YYYY-MM-DD`) |

Exemple :

```json
{
  "RoomA": "00000000-0000-0000-0000-000000000001",
  "RoomB": "36d5b466-659d-e873-6f57-79ccc0e3eead",
  "updatedAt": "2026-06-16"
}
```

## URL pour Unity (raw)

```
https://raw.githubusercontent.com/Genie-Culturel/SpatialAnchors/main/anchors.json
```

Les jeux Unity peuvent charger ce fichier au démarrage pour récupérer les UUID des ancres sans rebuild.

## Mise à jour après recalibration

1. Calibrer la nouvelle ancre avec l'outil Quest
2. Remplacer l'UUID correspondant dans `anchors.json`
3. Mettre à jour le champ `updatedAt`
4. Commit sur GitHub — pas de rebuild des jeux nécessaire
