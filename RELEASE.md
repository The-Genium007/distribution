# Publier un modset (release)

Procédure pour produire et publier un manifeste signé + ses zips, consommables par le launcher.
Outil : `tessera-release` (crate `tools/release`). Contrat : [ADR 0006](../docs/architecture/0006-distribution-and-signing.md).

## Publication via CI (recommandé)

Le workflow **`.github/workflows/modset-release.yml`** (`workflow_dispatch`, bouton manuel) automatise
tout. Trois actions :

- **`package-dev`** → assemble le core (netcode épinglé `netcode-v*` + mods redscript in-repo +
  toolchain) et publie sur **dev** avec une version `X.Y.Z-devN` auto-incrémentée.
- **`promote`** (`from` = dev ou playtest) → **re-manifeste les octets exacts** du canal source vers
  le suivant (dev→playtest retire `-dev` ; playtest→stable). Ce qui passe en stable est
  byte-pour-byte ce qui a été validé.
- **`hotfix`** (cible stable) → repackage stable avec un `Z` incrémenté.

**Versionnage par canal** : dev `X.Y.Z-devN` · playtest/stable `X.Y.Z` (bump `Z`). Chaque canal suit
sa propre version (lue dans son `latest.json`).

**Composition** : les packages du modset core sont déclarés dans
[`modset.packages.toml`](modset.packages.toml) — ajouter un mod interne = ajouter une entrée.

**Secrets requis (setup opérateur, une fois)** : `TESSERA_SIGNING_KEY` (seed de signature) et
`DISTRIBUTION_TOKEN` (push + Releases sur `The-Genium007/distribution`).

> ⚠️ **`netcode-v*`** : le workflow **télécharge** l'overlay netcode par version. La *publication* de
> `netcode-v<version>` (le zip `cyberverse-client.zip` = port Cyberverse) provient du **build du
> netcode**, pas de `client-plugin.yml` (qui compile le plugin minimal `tessera-client`). Tant que
> le build Cyberverse n'est pas un workflow, publier `netcode-v*` **une fois à la main** (upload du
> `cyberverse-client.zip` existant), puis épingler cette version.

La procédure **manuelle** ci-dessous reste le **fallback** (ou pour comprendre ce que fait le CI).

## Pré-requis (une fois)

1. **Dépôt public** `The-Genium007/distribution` créé, **GitHub Pages activé** sur la branche
   `main` (racine). Vérifier que `https://the-genium007.github.io/distribution/pubkey.ed25519`
   répond après le premier mirror.
2. **Clé privée de signature** dans ton gestionnaire de mots de passe (seed base64 32 octets).
   Elle n'est **jamais** dans le dépôt. La clé publique correspondante est épinglée dans le
   launcher et copiée dans [`pubkey.ed25519`](pubkey.ed25519).
3. `gh` (GitHub CLI) authentifié pour créer les Releases.

> **Rotation de clé** (si fuite) — générer une nouvelle paire, mettre à jour `pubkey.ed25519`,
> faire ré-épingler la publique côté launcher, re-signer les manifestes :
> ```bash
> openssl genpkey -algorithm ed25519 -out new.pem
> openssl pkey -in new.pem -outform DER | tail -c 32 | base64   # seed (SECRET)
> openssl pkey -in new.pem -pubout -outform DER | tail -c 32 | base64   # pubkey
> ```

## Étapes

### 1. Préparer le staging
Place le contenu de chaque package dans un dossier overlay (chemins **relatifs**, enracinés
à la racine du jeu) et écris un `release.toml` (voir [`release.example.toml`](release.example.toml)).

### 2. Construire + signer
```bash
export TESSERA_SIGNING_KEY='<seed base64 32 octets>'   # depuis ton gestionnaire de mots de passe
cargo run -p tessera-release -- build \
  --descriptor path/to/release.toml \
  --out target/release-artifacts \
  --dist distribution
```
Produit `distribution/modsets/<canal>/latest.json` + `latest.json.sig` (vrais sha256, signés)
et les zips dans `target/release-artifacts/`.

### 3. Vérifier (avant publication)
```bash
cargo run -p tessera-release -- verify \
  --manifest distribution/modsets/<canal>/latest.json \
  --pubkey @distribution/pubkey.ed25519
```
Attendu : `✔ signature VALIDE`.

### 4. Uploader les zips en Release
Le tag DOIT être `modset-v<modsetVersion>` et les assets `<id>.zip` (URLs du manifeste).
```bash
gh release create modset-v<version> target/release-artifacts/*.zip \
  --repo The-Genium007/distribution \
  --title "modset v<version>" --notes "..."
```

### 5. Publier le manifeste (mirror Pages)
```bash
git add distribution/ && git commit -m "release(modset): v<version>"
./scripts/distribution-subtree-push.sh
```
Le push met à jour Pages. ⚠️ Cache CDN Pages (~minutes) : le nouveau `latest.json` peut
mettre quelques minutes à être servi.

## Ordre de publication recommandé
`dev` (intégration launcher) → `playtest` (testeurs) → `stable` (public). Tester l'install
de bout en bout sur `dev` avant de promouvoir.

## Invariants de sécurité (ne jamais enfreindre)
- `sha256` **réels** uniquement — jamais vide ni `REPLACE_*` (l'outil le garantit).
- Octets signés == octets publiés (l'outil sérialise une seule fois).
- Clé privée **hors dépôt**.
- Zips : chemins relatifs, pas de `..`/absolu/symlink (l'outil refuse à la construction).
