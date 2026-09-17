# Guide d'installation — WSL & Conda

**Formation FEF · Afroscreen — Bio-informatique**

- **Windows** : installez d'abord **WSL** (partie 1), puis **Conda** (partie 2).
- **macOS / Linux** : passez directement à **Conda** (partie 2).


# Partie 1 — Installation de WSL (Windows uniquement)


## Pourquoi WSL ?

Les outils de bio-informatique sont conçus pour **Linux**. Sur Windows, on installe **WSL** (*Windows Subsystem for Linux*), qui fait tourner un vrai **Ubuntu (Linux)** à l'intérieur de Windows — sans supprimer Windows, sans machine virtuelle compliquée.

Une fois WSL installé, vous aurez un terminal Linux prêt pour la formation.


## Avant de commencer — vérifier votre version de Windows

WSL nécessite **Windows 10 (version 2004 ou plus récente)** ou **Windows 11**.

Pour vérifier :
1. Appuyez sur les touches **Windows + R**.
2. Tapez `winver` et validez.
3. Une fenêtre affiche votre version.

> Si votre Windows est plus ancien, faites d'abord les **mises à jour Windows** (Paramètres → Windows Update → Rechercher des mises à jour), puis revenez à ce guide.

---

## Installation (la méthode simple)

C'est la seule commande à retenir. Elle installe tout ce qu'il faut, y compris Ubuntu.

### Étape 1 — Ouvrir PowerShell en administrateur

1. Cliquez sur le menu **Démarrer**.
2. Tapez **PowerShell**.
3. **Clic droit** sur « Windows PowerShell » → **Exécuter en tant qu'administrateur**.
4. Une fenêtre bleue s'ouvre. Si Windows demande une autorisation, cliquez **Oui**.

### Étape 2 — Lancer l'installation

Dans la fenêtre bleue, tapez cette commande puis **Entrée** :

```powershell
wsl --install
```

Cette commande active automatiquement tout ce qui est nécessaire et installe **Ubuntu** par défaut.

### Étape 3 — Redémarrer

Quand l'installation le demande, **redémarrez votre ordinateur**. C'est indispensable.

### Étape 4 — Configurer Ubuntu (au redémarrage)

Après le redémarrage, une fenêtre **Ubuntu** s'ouvre automatiquement et finit l'installation (cela prend quelques minutes la première fois). Elle vous demande de créer :

- un **nom d'utilisateur** (en minuscules, sans espace — ex. `armel`) ;
- un **mot de passe**.

> **Important** — Quand vous tapez le mot de passe, **rien ne s'affiche à l'écran** (pas d'étoiles, pas de points). C'est normal, c'est une sécurité Linux. Tapez votre mot de passe et appuyez sur Entrée. Notez-le, il vous servira.

---

## Vérifier que tout fonctionne

### Test 1 — Ubuntu s'ouvre

Menu Démarrer → tapez **Ubuntu** → l'application s'ouvre sur un terminal.

### Test 2 — La version de WSL

Ouvrez **PowerShell** (pas besoin d'administrateur cette fois) et tapez :

```powershell
wsl --list --verbose
```

Vous devez voir **Ubuntu** avec, dans la colonne VERSION, le chiffre **2**.

### Test 3 — Une commande Linux

Dans le terminal **Ubuntu**, tapez :

```bash
pwd
```

Si une ligne comme `/home/votre_nom` s'affiche, **tout fonctionne** — vous êtes prêt(e) pour la formation.


## En cas de problème

### « wsl --install » ne fonctionne pas / erreur

La cause la plus fréquente est la **virtualisation désactivée** dans le BIOS/UEFI de l'ordinateur, ou un **Windows pas à jour**.

- **Mettre Windows à jour** : Paramètres → Windows Update → Rechercher des mises à jour. Puis réessayer.
- **Activer la virtualisation** : elle se trouve dans les réglages du BIOS/UEFI (à l'allumage de l'ordinateur, souvent touche `F2`, `F10`, `Suppr` ou `Échap` selon la marque), sous un nom comme *Virtualization Technology*, *Intel VT-x*, *AMD-V* ou *SVM Mode*. Activez-la, enregistrez, redémarrez.

### « Ordinateur d'institution » sans droits administrateur

Si vous ne pouvez pas exécuter PowerShell en administrateur (ordinateur d'un hôpital, d'un labo…), contactez votre **service informatique** pour qu'il installe WSL, ou **prévenez un encadrant** dès votre arrivée à la formation : un rattrapage est prévu, mais il prend du temps.

### Guide officiel Microsoft

Pour tout problème, la référence complète (en anglais, avec dépannage détaillé) :
<https://learn.microsoft.com/en-us/windows/wsl/install>


## Checklist finale WSL

Avant la formation, vérifiez que :

- [ ] `winver` indique Windows 10 (2004+) ou Windows 11
- [ ] `wsl --install` a été exécuté en PowerShell administrateur
- [ ] L'ordinateur a redémarré
- [ ] Ubuntu s'ouvre depuis le menu Démarrer
- [ ] `wsl --list --verbose` affiche Ubuntu en VERSION **2**
- [ ] La commande `pwd` fonctionne dans Ubuntu

Si toutes les cases sont cochées, votre WSL est prêt. Passez à l'installation de Conda ci-dessous.

---

# Partie 2 — Installation de Conda (tous les systèmes)

Cette partie concerne **tout le monde** : Windows (dans le terminal Ubuntu/WSL), macOS et Linux.

**Conda** est l'outil qui installe proprement les logiciels de bio-informatique et gère leurs dépendances. On installe **Miniforge**, une version légère et gratuite de Conda, préconfigurée pour la science.

> **Où taper ces commandes ?**
> - **Windows** : dans le terminal **Ubuntu** (celui de WSL), pas dans PowerShell.
> - **macOS** : dans le **Terminal** (Spotlight ⌘+Espace → « Terminal »).
> - **Linux** : dans votre **terminal** (Ctrl+Alt+T).


## Étape 1 — Télécharger l'installateur Miniforge

Choisissez la commande correspondant à votre système.

**Windows (WSL/Ubuntu) et Linux** :

```bash
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh" -O miniforge.sh
```

**macOS — puce Apple (M1/M2/M3/M4)** :

```bash
curl -L "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh" -o miniforge.sh
```

**macOS — puce Intel** :

```bash
curl -L "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-x86_64.sh" -o miniforge.sh
```

> **Quelle puce ai-je sur mon Mac ?** Menu  (en haut à gauche) → « À propos de ce Mac ». Si vous lisez « Puce Apple », prenez la version arm64. Si vous lisez « Processeur Intel », prenez la version Intel (x86_64).


## Étape 2 — Lancer l'installation

```bash
bash miniforge.sh
```

Déroulez l'installation :
- appuyez sur **Entrée** pour lire la licence, puis tapez `yes` pour l'accepter ;
- acceptez l'emplacement proposé par défaut (appuyez sur Entrée) ;
- à la question finale (« initialize Miniforge3? »), tapez **`yes`**.


## Étape 3 — Activer Conda

Fermez complètement le terminal, puis **rouvrez-le**. Vous devez maintenant voir `(base)` au début de la ligne — cela signifie que Conda est actif.

## Étape 4 — Vérifier

```bash
conda --version
mamba --version
```

Deux numéros de version doivent s'afficher (par exemple `conda 24.11.0` et `2.0.5`). Les numéros exacts peuvent différer — l'important est qu'**aucune erreur** n'apparaisse.

> **Si « command not found »** — Fermez complètement le terminal et rouvrez-le. Si le problème persiste, tapez :
> ```bash
> source ~/miniforge3/etc/profile.d/conda.sh
> conda init
> ```
> puis rouvrez le terminal.


## Étape 5 — Configurer les canaux

Les « canaux » sont les dépôts d'où Conda télécharge les logiciels. On les configure une seule fois :

```bash
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
```

---

## Checklist finale Conda

- [ ] L'installateur Miniforge a été téléchargé (bon système : Linux / macOS arm64 / macOS Intel)
- [ ] `bash miniforge.sh` s'est terminé sans erreur
- [ ] Le terminal affiche `(base)` au début de la ligne
- [ ] `conda --version` et `mamba --version` affichent un numéro
- [ ] Les canaux `conda-forge` et `bioconda` sont configurés


