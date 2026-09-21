# Formation sur la Surveillance Génomique et l'Approche One Health

**FEF · Afroscreen — Module Introduction à la Bio-informatique**

Supports pratiques de la formation en **analyse de données génomiques des pathogènes** de l'atelier **AFROSCREEN FEF**, module bio-informatique.

Cette formation apprend à analyser des données de séquençage viral réelles, du fichier brut au génome consensus et à l'identification des variants — **à la main d'abord**, pour comprendre chaque étape, puis avec des **pipelines automatisés**.

---

## À qui s'adresse cette formation

Biologistes de laboratoire, bio-informaticiens débutants, médecins et personnels de santé publique impliqués dans la surveillance génomique des épidémies. Aucun prérequis avancé : on part des bases (ligne de commande, installation des outils) pour aller vers l'analyse autonome.

---

## Ce qu'on apprend

- Installer et gérer un environnement bio-informatique (Conda) sur Windows, macOS ou Linux
- Se déplacer et manipuler des fichiers en ligne de commande
- Contrôler la qualité et nettoyer des lectures de séquençage
- Aligner sur une référence, appeler les variants, reconstruire un génome consensus
- Assembler un génome sans référence (de novo)
- Comparer les plateformes Illumina et Nanopore
- Utiliser des pipelines automatisés (nf-core/viralrecon, GeVarLi et autres)

---

## Structure du dépôt

```
bioinfo_practice/
│
├── SC2/                     Cas d'étude 1 — SARS-CoV-2 (amplicon ARTIC, Illumina)
│   ├── 01_qc/                   contrôle qualité & nettoyage
│   ├── 02_mapping/              alignement & consensus
│   ├── 03_variants/             appel de variants
│   ├── data/                    référence & fichiers partagés
│   ├── environment.yml          environnement conda du cas
│   ├── GLOSSAIRE.md             définitions techniques (tous les cas)
│   └── README.md
│
├── MPOX/                    Cas d'étude 2 — Monkeypox virus (shotgun, 2 plateformes)
│   ├── illumina/                voie lectures courtes
│   │   ├── 01_qc/                   contrôle qualité
│   │   ├── 02_mapping/              alignement & consensus
│   │   └── 03_denovo/               assemblage de novo
│   ├── nanopore/                voie lectures longues
│   │   ├── 01_qc/                   contrôle qualité
│   │   └── 02_mapping/              alignement & consensus
│   ├── data/                    référence & fichiers partagés
│   ├── environment.yml
│   └── README.md
│
└── pipelines/               Session pipelines automatisés
    ├── README.md                installation des pipelines

presentations/               Supports de cours (théorie & introductions)
├── introduction_linux.pdf         introduction à la ligne de commande Linux
└── ...                          autres présentations
```

---

## Les deux cas d'étude

### SARS-CoV-2 — préparation amplicon

Un échantillon positif SC2 séquencé en Illumina par protocole ARTIC en Afrique du Sud. On suit la chaîne complète : QC (Cutadapt, Sickle) → alignement (BWA) → variants et consensus (iVar). Illustre le traitement des données **amplicon** (avec retrait des amorces).

### Monkeypox virus — préparation shotgun, deux plateformes

Le premier cas de mpox détecté au Kenya (2024), séquencé sur **Illumina ET Nanopore** à partir du même échantillon. Illustre la **métagénomique shotgun**, la comparaison **mapping vs de novo**, et la comparaison **Illumina vs Nanopore**.

Chaque cas est autonome et progresse par étapes numérotées. Chaque étape a son `README.md` avec les commandes, leur explication, et l'interprétation des résultats.

---

## Prérequis & installation

### Système d'exploitation

La formation fonctionne sur **Linux**, **macOS** et **Windows**. Les outils de bio-informatique étant conçus pour des systèmes de type UNIX, les utilisateurs **Windows doivent installer WSL** (Windows Subsystem for Linux), qui fait tourner un vrai Linux (Ubuntu) à l'intérieur de Windows.

### Utilisateurs Windows — installer WSL 

1. Ouvrir **PowerShell en administrateur** (menu Démarrer → taper « PowerShell » → clic droit → *Exécuter en tant qu'administrateur*).
2. Lancer :

```powershell
wsl --install
```

3. **Redémarrer** l'ordinateur, puis créer un nom d'utilisateur et un mot de passe Linux au premier lancement d'Ubuntu.

Guide officiel Microsoft (recommandé) : <https://learn.microsoft.com/en-us/windows/wsl/install>

> **En cas de blocage** — Les causes les plus fréquentes sont la virtualisation désactivée dans le BIOS/UEFI ou un Windows non à jour. Sur un ordinateur d'institution, des droits administrateur peuvent être nécessaires. Le guide Microsoft ci-dessus couvre le dépannage.

### Conda (tous les systèmes)

Tous les outils s'installent via **Conda** (distribution **Miniforge**). Le module d'installation de la formation détaille la procédure pas à pas pour chaque OS. Une fois Conda en place, chaque cas d'étude fournit son `environment.yml` pour installer les outils en une commande.

> La procédure complète d'installation (Conda + environnements) est détaillée dans le dossier `presentations/` et dans les README de chaque cas.

---

## Comment démarrer

1. **Cloner le dépôt** :

```bash
git clone https://github.com/armel001/FEF_Afroscreen_Training.git
mv FEF_Afroscreen_Training/bioinfo_practice/ .
cd  bioinfo_practice
```

2. **Choisir un cas** (SC2 ou MPOX) et lire son `README.md`.

3. **Créer l'environnement conda** du cas, puis suivre les étapes dans l'ordre :

```bash
cd SC2
mamba env create -f environment.yml
conda activate sc2_analyse
```

4. **Suivre chaque étape** (`01_qc/`, `02_mapping/`, …) en lisant son README.

> Les termes techniques sont définis dans **[`bioinfo_practice/SC2/GLOSSAIRE.md`](bioinfo_practice/SC2/GLOSSAIRE.md)**.

---

## Note sur les données

Les fichiers de séquençage (`.fastq.gz`), alignements (`.bam`), résultats et modèles volumineux **ne sont pas versionnés** dans ce dépôt (voir `.gitignore`). Les participants les **téléchargent** en suivant les instructions de chaque cas (les accessions SRA sont indiquées dans les README). Seuls les fichiers légers et nécessaires (références, README, configurations) sont inclus.

---

## Pipelines

Après l'analyse manuelle, la session `pipelines/` introduit les workflows automatisés :

- **GeVarLi** — pipeline Snakemake du projet AFROSCREEN (TransVIHMI / IRD) — <https://forge.ird.fr/transvihmi/nfernandez/GeVarLi>
- **nf-core/viralrecon** — pipeline Nextflow de référence internationale — <https://nf-co.re/viralrecon>

L'objectif : constater que les pipelines reproduisent l'analyse manuelle, et apprendre à les utiliser en autonomie.

---

## Facilitateurs

Thibaut Armel Chérif Gnimadi, Mohamed Kane & Esther Konou

---

## Crédits

Formation **FEF · AFROSCREEN** — module introduction bio-informatique.
Projet AFROSCREEN : <https://www.afroscreen.org/>
