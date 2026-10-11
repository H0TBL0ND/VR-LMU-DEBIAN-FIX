# Le Mans Ultimate en VR sous Linux (WiVRn + xrizer + Proton) — les vrais blocages et leurs solutions

English: [README.md](README.md)

Le Mans Ultimate (LMU, appid 2399420) peut se lancer en VR sous Linux avec un casque autonome, **sans SteamVR**, grâce à WiVRn (runtime OpenXR),
xrizer (OpenVR → OpenXR) et un Proton-GE modifié. En l'état, le jeu se ferme au bout de quelques secondes. Nous avons rencontré
**quatre blocages successifs**, chacun cachant le suivant. Ce document donne les symptômes, le diagnostic et la solution de chacun.

> **Statut.** Testé une fois, sur une seule machine : le jeu démarre et tourne en VR (confirmé par l'utilisateur, casque sur la tête).
> Non testé : longues sessions, performances, jeu en ligne. **La configuration qui marche lance le jeu *sans Easy Anti-Cheat*** (blocage 2) :
> le multijoueur en ligne ne devrait donc pas fonctionner avec.

## Machine de test

Debian 13 (GNOME, session Wayland, Steam du paquet Debian) · Radeon RX 7700 XT (RADV) · Pico 4 via WiVRn 26.9 (Flatpak) · xrizer 0.6 (inclus dans le Flatpak WiVRn) ·
`GE-Proton10-34-LMU-hid_fixes-vr-bcrypt` (JacKeTUs) · Mesa hôte 25.0.7 → **26.1.6 (trixie-backports)** ; runtime Flatpak de WiVRn : 26.2.2.

## Les quatre blocages

| # | Symptôme | Cause | Solution |
|---|---|---|---|
| 1 | Avec la `openvr_api.dll` du jeu (OpenComposite) : fenêtre « call openxr fail » ; `opencomposite.log` : `XR_ERROR_FUNCTION_UNSUPPORTED … xrCreateXDevListMNDX` | OpenComposite demande une fonction d'extension Monado que le chemin runtime via `wineopenxr` de Proton ne fournit apparemment pas (couche fautive non identifiée) | Remplacer la DLL par celle de Valve et laisser xrizer gérer OpenVR (connu : [xrizer#301](https://github.com/Supreeeme/xrizer/issues/301)) |
| 2 | Après remplacement de la DLL : plantage ~2 s après le lancement, avant tout appel VR : `c0000005` dans `load_dll` de `ntdll` (pointeur nul) | Constaté uniquement avec le lanceur EAC (`start_protected_game.exe`) | Lancer directement `Le Mans Ultimate.exe` (EAC désactivé, voir ci-dessous) |
| 3 | xrizer « panicked » à `compositor.rs:1067` (échec de swapchain) ; WiVRn : `vk_create_image_from_native size mismatch, exported 10616832 but requires 13107200` | **Versions de Mesa différentes** entre le runtime Flatpak de WiVRn (26.2) et le Mesa de l'hôte utilisé par le jeu (25.0) — cf. [WiVRn#957](https://github.com/WiVRn/WiVRn/issues/957) | Rapprocher le Mesa de l'hôte de celui du Flatpak (Debian : `trixie-backports`, 26.1.6) |
| 4 | Le jeu s'arrête sur `VRUI.cpp : D3DX11CreateTextureFromMemory(… vrCursorFile …) FATAL!!!` (dans `UserData/Log/trace*.txt` du jeu) | Le jeu charge un **curseur Windows (`.cur`)** comme texture, ce que `d3dx` de Wine ne sait pas faire ; ce Proton contient un contournement dans sa **`d3dx11_43` intégrée**. Or le préfixe avait la `d3dx11_43.dll` **native de Microsoft** (probablement posée par le script DirectX de Steam — non vérifié), qui prend le dessus | Remettre la `d3dx11_43.dll` intégrée du Proton dans le préfixe |

Contexte : [Proton#7512](https://github.com/ValveSoftware/Proton/issues/7512) (LMU), qui documente le problème du `.cur` (mdias, srounce).

## Procédure qui a fonctionné

Chemins de l'installation Steam Debian (`~/.steam/steam`). Fermer Steam avant de toucher à ses fichiers. **Une mise à jour du jeu ou la « vérification des fichiers » annulera très probablement les étapes 4 et 5 (non testé).**

1. **Mesa** : aligner le Mesa de l'hôte sur celui du Flatpak WiVRn (`flatpak list --runtime | grep Mesa` ; `vulkaninfo --summary | grep driverInfo`). Sous Debian 13 : activer `trixie-backports`
   et mettre à jour ensemble les paquets Mesa 64 et 32 bits (simulation d'abord avec `apt-get -s` ; rien n'a été supprimé). Les commandes exactes sont dans [README.md](README.md#procedure-that-worked).
   Retour arrière : réinstaller `25.0.7-2+deb13u1`. Se déconnecter/reconnecter ou redémarrer ensuite (nous ne l'avons pas fait pendant notre test).
2. **Proton** : `GE-Proton10-34-LMU-hid_fixes-vr-bcrypt` ([JacKeTUs/proton-ge-custom](https://github.com/JacKeTUs/proton-ge-custom/releases)) dans `compatibilitytools.d/`, à sélectionner pour LMU.
   « Proton EasyAntiCheat Runtime » (1826330) était aussi installé ; nous n'avons pas testé sans.
3. **Runtime OpenVR = xrizer** : `~/.config/openvr/openvrpaths.vrpath` doit lister le dossier xrizer du Flatpak WiVRn et `~/.config/openxr/1/active_runtime.json` pointer sur le `openxr_wivrn.json` de WiVRn.
   Chez nous le vrpath contient le chemin **avec hash** (`/var/lib/flatpak/app/io.github.wivrn.wivrn/x86_64/stable/<commit>/files/xrizer`), qui change à chaque mise à jour du Flatpak :
   à revérifier après une mise à jour de WiVRn (l'alias `…/current/active/files/xrizer` devrait l'éviter, non testé dans ce fichier).
4. **`openvr_api.dll` de Valve** dans le dossier du jeu (garder l'originale en `openvr_api.dll.opencomposite`) : version 64 bits de [ValveSoftware/openvr](https://github.com/ValveSoftware/openvr/tree/master/bin/win64) (2.15.6 chez nous).
5. **Sans EAC (désactive le jeu en ligne)** : `mv start_protected_game.exe start_protected_game.exe.orig` puis `cp "Le Mans Ultimate.exe" start_protected_game.exe`. Pour annuler : remettre le `.orig`.
6. **`d3dx11_43` du préfixe** : remplacer `…/compatdata/2399420/pfx/drive_c/windows/system32/d3dx11_43.dll` (celle de Microsoft, 276 832 octets chez nous) par
   `…/compatibilitytools.d/GE-Proton10-34-LMU-hid_fixes-vr-bcrypt/files/lib/wine/x86_64-windows/d3dx11_43.dll` (307 949 octets), après sauvegarde.
   Vérification avec `PROTON_LOG=1` : le log doit dire `d3dx11_43.dll … builtin` (il disait `native`).
7. **Options de lancement Steam** (utilisées chez nous, sans test pour savoir lesquelles sont indispensables) :
   `PRESSURE_VESSEL_IMPORT_OPENXR_1_RUNTIMES=1 PRESSURE_VESSEL_FILESYSTEMS_RW=/var/lib/flatpak/app/io.github.wivrn.wivrn %command% +VR`
   (ajouter `PROTON_LOG=1 PROTON_LOG_DIR=$HOME/lmu-logs` pour les logs ; éviter `WINEDEBUG=+module` : logs de plusieurs Go et risque de boucle d'erreurs qui remplit le disque).
8. Lancer le serveur WiVRn, connecter le casque, puis lancer LMU avec le bouton « Jouer » normal de Steam (pas « avec SteamVR/OpenXR »).

## Versions et empreintes exactes

Build Steam de LMU `25661166` · noyau 6.12.107+deb13-amd64 · WiVRn 26.9 (commit Flatpak `61d58e18…5aea`) · `GE-Proton10-34-LMU-hid_fixes-vr-bcrypt` ·
`openvr_api.dll` de Valve 2.15.6 sha256 `bab8ac6e…cb4a` · DLL d'origine (OpenComposite) `21a8b364…cf5a` · lanceur EAC d'origine `e86f518b…d95c` ·
`d3dx11_43.dll` du préfixe avant (Microsoft) `981e4262…bd59`, après (Proton) `373f2989…9551`. Les empreintes complètes sont dans [README.md](README.md#exact-versions-and-checksums-of-our-working-setup).
Voir aussi la [fiche LMU de la base VR on Linux](https://db.vronlinux.org/games/2399420.html).

## Questions ouvertes

- Le blocage 2 est-il vraiment « l'EAC » ? Constaté avec deux DLL de remplacement différentes, jamais avec l'originale, et disparu en remplaçant le lanceur EAC ; mécanisme non identifié.
- La DLL OpenComposite d'origine marcherait-elle si `xrCreateXDevListMNDX` était transmise (blocage 1) ? Non testé.
- Mesa : écart de version majeure ou version exacte ? 26.1.6 (hôte) contre 26.2.2 (Flatpak) a suffi.
- Quelles variables `PRESSURE_VESSEL_*` restent nécessaires avec xrizer ?

## Remerciements

Le gros du travail est celui d'autres personnes : JacKeTUs (Proton pour LMU), mdias et srounce (analyse et contournement du `.cur`), xytovl et l'équipe WiVRn (explication de l'écart Mesa),
ImSapphire (DLL de Valve + xrizer), Supreeeme (xrizer). Ce document décrit seulement l'assemblage sur une configuration. Diagnostic mené avec Claude Code (Anthropic) et le propriétaire de la machine.
