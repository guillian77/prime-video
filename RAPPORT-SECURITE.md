# Rapport de sécurité — Prime Video (Android TV) 6.23.23 patché « Skip ads »

Date : 08/11/2026 · Analyste : Hermes · Environnement : conteneur Unraid (x64), JDK 21, build-tools 34.0.0

## Artefacts

| Fichier | SHA-256 | Taille |
|---|---|---|
| `prime-video-6.23.23.apkm` (original officiel, APKMirror) | `1cd5be2e9cd29080a0dc5060995006b3170a4f82b9c7e67d25eb8c79ae67d7a8` | 17 845 803 o |
| `prime-video-6.23.23-patched.apk` (in-place, même package) | `5d2ab8ff6b90a219907736be40521b80cd79fa08ab86bfd594254d2855d13dd8` | 38 292 793 o |
| `prime-video-6.23.23-patched-clone.apk` (clone `.mod`) | `c233e532a9bc812cdc391282b21db3e00fa00cd1a342f7f56313927c02251892` | 38 292 793 o |
| `libpvhook.so` extrait | `d98ba4c6a0ac0a710d284cafb9ca273f2702464ce5f4da9226636c498fd81857` | 83 624 o |

Chaîne : APKMirror (bundle armv7a, 26 splits) → Morphe Desktop 2.1.1 + patches ATV 1.50.3 (dépôt `ajstrick81/morphe-androidtv-patches`).

## 1. Provenance de l'APK source — OK

- `apksigner verify` : v1 + v2 + v3 valides, SourceStamp Play présent.
- Certificat : `CN=Amazon Services LLC` — SHA-256 `2f19adeb284eb36f7f07786152b9a1d14b21653203ad0b04ebbf9c73ab6d7625`, RSA 4096.
- Correspond exactement à l'empreinte publiée par APKMirror pour cette version → aucun APK retouché en amont.

## 2. Signature de l'APK patché — conforme à l'attendu, avec conséquence

- `CN=Morphe`, auto-signé, RSA 4096, SHA-256 `9d9f4cbc207df06a3cfb6b727ecd5adcfe6c95f4c624cf1eae10f4f706824c15`.
- Schémas v1 + v2 + v3 valides ; `zipalign -c 4` OK.
- Conséquence : la signature Amazon est remplacée → installation impossible par-dessus l'app d'origine (il faut désinstaller l'existant) et plus aucune mise à jour Play Store.
- Risque résiduel : la clé « Morphe » est publique et partagée par tous les patchs Morphe ; quiconque la possède peut produire un APK de même identité. Sans incidence pratique ici (paquet déjà détourné d'Amazon), mais à savoir.

## 3. Manifeste — 2 différences seulement

Comparaison `aapt2 dump xmltree` (base.apk officielle vs APK patché) :

- `versionCode` : 606023231 → **616023231** (incrément volontaire du patch « Disable auto-updates »).
- `android:extractNativeLibs` : `false` → `true` (nécessaire pour charger `libpvhook.so` depuis un APK fusionné).
- Suppression des métadonnées de splits Play (`com.android.vending.splits*`, `com.android.stamp.*`) et de `res/xml/splits0.xml` : normal pour un APK universel.

Aucune permission ajoutée. Aucun composant ajouté. Aucun `exported`, `debuggable`, `allowBackup`, `usesCleartextTraffic` ni `networkSecurityConfig` modifié (le patch « Override certificate pinning » n'a **pas** été appliqué).

## 4. Code Dalvik — écart minimal et ciblé

- `classes2.dex` et `classes4.dex` : **identiques octet pour octet**.
- `classes.dex` / `classes3.dex` : modifiés uniquement par extraction de 10 classes (SSAI media3, `BasicNetwork` Volley, `MetricsTransporter`, `PrimeVideoApplication`…).
- `classes5.dex` (46 Ko) : les 10 classes patchées ; le code injecté appelle `SkipAdsPatch.skipAllMedia3AdGroups(...)`.
- `classes6.dex` (180 Ko) : bundle d'extensions Morphe qui embarque **toutes** les apps (HBO, Peacock, Twitch, ESPN…). Code mort pour Prime Video → poids inutile, pas de risque d'exécution.

APIs sensibles recherchées dans le code ajouté — **toutes absentes** : `Runtime.exec`, `ProcessBuilder`, `DexClassLoader`, `SmsManager`, `TelephonyManager`, `MediaRecorder`, `Camera`, `getDeviceId`, `getLastKnownLocation`, `ContactsContract`, `AccessibilityService`, `setComponentEnabledSetting`, `java.net.Socket`.

## 5. Bibliothèque native `libpvhook.so` — propre

- ELF 32 bits ARM (armeabi-v7a), PIE, 83 Ko.
- Bibliothèques liées : `liblog`, `libm`, `libdl`, `libc` uniquement.
- Symboles importés : memory (`memcpy/memmove/memcmp/memset/memmem`), `malloc/free/realloc`, `mprotect`, pthread, `dl_iterate_phdr`, `__android_log_print`, `sleep/usleep`. **Aucun** `socket`, `connect`, `getaddrinfo`, `dlopen`, `fopen`, `exec*`, `ptrace`, `fork`.
- Exports cohérents avec la doc : `JNI_OnLoad`, proxies `memcpy/memmove`, `pvfilter::strip_remote_items`, `pvfilter::blank_complete_media_urls`, `pvgot::hook_import`.
- Les 6 autres `.so` sont **identiques octet pour octet** à ceux du bundle officiel.

## 6. Ressources et fichiers

- 219 entrées identiques, ressources seulement reconstruites (renommage des qualificateurs `-v4/-v21`, ARSC régénéré).
- Aucun exécutable/script ajouté. Seuls ajouts : `classes5.dex`, `classes6.dex`, `META-INF/MORPHE.SF/.RSA` (signature).
- ABI unique : `armeabi-v7a` (compatible Shield ; pas de 64 bits utile ici).

## 7. Endpoints réseau présents dans le code ajouté

Analyse des chaînes des dex 5/6 (aucune requête émise depuis le code Prime Video) :

- Listes de blocage publicitaires pour d'autres apps (peacocktv, amazon-adsystem, tubitv, twitch, youbora…).
- `https://site.api.espn.com/...` (extension ESPN, non utilisée ici).
- `https://morphe.invalid/b.ts?i=` — sentinelle : TLD `.invalid` non routable, sert de fausse URL de segment.
- Aucun endpoint de télémétrie ou de collecte vers un tiers.

## 8. Limites de l'audit

- Pas d'analyse dynamique (pas de Shield/émulateur connecté) : comportement runtime non instrumenté.
- Pas de scan multi-antivirus : pas de clé API VirusTotal. Les SHA-256 ci-dessus sont fournis pour vérification externe.
- Audit statique uniquement ; le code source des patches (Kotlin + C++) a été consulté et concorde avec le binaire produit.

## Verdict

APK conforme à ce qu'annonce le patch : suppression des pubs côté app, aucune capacité nouvelle (réseau, exécution, permissions), aucune modification du manifeste au-delà du versionCode et du chargement natif. Le risque principal n'est pas le code mais l'usage : violation des CGU Amazon et remplacement de la signature officielle.
