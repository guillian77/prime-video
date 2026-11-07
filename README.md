# Prime Video (Android TV) — sans publicité sur NVIDIA Shield

APK Prime Video **Android TV 6.23.23** (`com.amazon.amazonvideo.livingroom`) patché avec le patch communautaire
**« Skip ads »** du dépôt [ajstrick81/morphe-androidtv-patches](https://github.com/ajstrick81/morphe-androidtv-patches),
appliqué via **Morphe Desktop 2.1.1** avec le bundle de patches **1.50.3**.

Supprime les pré-rolls et mid-rolls (films + séries) côté application, sans DNS, sans VPN, sans root.

> ⚠️ **Version figée : 6.23.23 (moteur v15.5).** Ne pas mettre à jour vers 6.24.x : Amazon a déplacé le
> pipeline publicitaire dans le moteur natif (v16) et le patch n'a plus d'effet. Le patch `Disable auto-updates`
> est inclus : le Play Store ne remplacera pas l'app.

## Contenu de la release

| Fichier | Usage |
|---|---|
| `prime-video-6.23.23-patched.apk` | Installation en remplacement de l'app d'origine (même package) |
| `prime-video-6.23.23-patched-clone.apk` | Installation **côte à côte** (package `.mod`), si l'app d'origine est un système non désinstallable |
| `SHA256SUMS` | Empreintes de vérification |
| `RAPPORT-SECURITE.md` | Audit statique complet (signature, manifeste, dex, lib native, réseau) |

Installation **au choix** : commencez par l'APK standard. Si l'installation échoue avec
`INSTALL_FAILED_UPDATE_INCOMPATIBLE` ou `signatures do not match`, utilisez la variante clone.

## Installation sur la Shield

### Option A — ADB (la plus fiable)

Sur la Shield : **Paramètres → Système → Informations →** appuyer 7× sur *Build* pour activer les options
développeur, puis **Débogage réseau** (même réseau que le PC).

```bash
adb connect 192.168.1.xx:5555
adb install -r prime-video-6.23.23-patched.apk
```

Si l'app d'origine est un système non désinstallable :

```bash
# désinstallation pour l'utilisateur courant uniquement
adb shell pm uninstall --user 0 com.amazon.amazonvideo.livingroom

# ou, sans toucher au système, installer la variante clone
adb install prime-video-6.23.23-patched-clone.apk
```

### Option B — Downloader (sans PC)

Installer *Downloader* (AFTVnews) sur la Shield, puis ouvrir l'URL de l'asset de la release
(page *Releases* de ce dépôt). Nécessite un dépôt **public** pour un téléchargement sans authentification.

### Option C — Send Files to TV

Télécharger l'APK sur un PC/téléphone, puis le pousser sur la Shield avec *Send Files to TV* (même réseau),
et installer via un gestionnaire de fichiers avec « sources inconnues » autorisé.

## Après installation

- Se reconnecter au compte Amazon (les identifiants ne sont pas transférés d'une signature à l'autre).
- La signature ayant changé (Amazon → Morphe), **l'ancienne app doit être désinstallée** : aucune mise à jour
  automatique ne sera possible, c'est voulu.
- En cas de gel bref sur un très gros bloc publicitaire (régions à forte densité de pubs) :
  **Retour → Reprendre** relance la lecture sans pub.

## Vérification d'intégrité

```bash
sha256sum -c SHA256SUMS
```

## Sécurité — résumé de l'audit

- APK source officiel (signature Amazon Services LLC, empreinte `2f19adeb…` vérifiée) ; seul le re-signage change.
- Manifeste : seuls `versionCode` et `extractNativeLibs` diffèrent. **Aucune** permission ni composant ajouté.
- Code ajouté : 10 classes patchées + extensions Morphe ; aucune API sensible
  (`Runtime.exec`, `DexClassLoader`, SMS, localisation, contacts, caméra, socket brut).
- `libpvhook.so` (83 Ko) : ne lie que `libc/libm/libdl/liblog`, aucun accès réseau ni système de fichiers.
- Limites : audit **statique** uniquement, pas de scan multi-antivirus (empreintes fournies pour vérification externe).

Détail complet : [`RAPPORT-SECURITE.md`](RAPPORT-SECURITE.md).

## Avertissement

APK modifié, non distribué par Amazon. L'usage sort des conditions d'utilisation de Prime Video et peut cesser
de fonctionner à tout moment. Aucun contournement d'abonnement : l'abonnement Prime reste nécessaire.
À usage personnel.
