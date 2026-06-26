# distribution/ — Dépôt public des modsets TesseraSynth

Contenu **public et statique** servi par **GitHub Pages** depuis `The-Genium007/distribution`.
C'est la source de vérité que le **launcher** consomme (modèle *manifest-first*). Voir
[ADR 0006](../docs/architecture/0006-distribution-and-signing.md).

> ⚠️ **Ne pas éditer les manifestes à la main.** `modsets/<canal>/latest.json` et son
> `latest.json.sig` sont **générés et signés** par l'outil `tools/release` (`tessera-release`).
> Toute édition manuelle invaliderait la signature.

## URLs canoniques (figées)

| Ressource | URL |
|-----------|-----|
| Manifeste | `https://the-genium007.github.io/distribution/modsets/<canal>/latest.json` |
| Signature | `https://the-genium007.github.io/distribution/modsets/<canal>/latest.json.sig` |
| Asset zip | `https://github.com/The-Genium007/distribution/releases/download/modset-v<version>/<id>.zip` |

Canaux : `stable` (défaut), `playtest`, `dev`.

## Signature

- Ed25519 **détachée** : `latest.json.sig` = base64 d'une signature 64 octets sur les
  **octets bruts exacts** de `latest.json`.
- Clé publique (épinglée aussi dans le launcher) : voir [`pubkey.ed25519`](pubkey.ed25519).
- Le launcher **vérifie la signature avant de parser** quoi que ce soit.

## Structure

```
distribution/
  index.html            ← page d'accueil Pages
  pubkey.ed25519        ← clé publique de signature (copie de référence)
  .nojekyll             ← sert tous les fichiers tels quels (pas de Jekyll)
  modsets/
    stable/   latest.json + latest.json.sig
    playtest/ latest.json + latest.json.sig
    dev/      latest.json + latest.json.sig
```

Les `.zip` ne vivent pas ici : ils sont uploadés en **GitHub Releases** (tags `modset-v*`).

## Publication

Ce dossier est mirroré vers `The-Genium007/distribution` via
[`scripts/distribution-subtree-push.sh`](../scripts/distribution-subtree-push.sh). Le push
déclenche le redéploiement Pages. Les Releases sont créées par `tessera-release publish`.
