# Cas d'étude — MPOX (Monkeypox virus)

**Formation FEF · Afroscreen — Bio-informatique**

Analyse d'un échantillon MPOX réel, du fichier brut de séquençage au génome consensus, sur **deux plateformes** (Illumina et Nanopore) à partir du même échantillon.

---

## Contexte des données

Ces données proviennent du **premier cas de mpox détecté au Kenya**, en juillet 2024. Le patient, un chauffeur routier longue distance ayant voyagé de Kampala (Ouganda) vers Mombasa (Kenya), a présenté les symptômes classiques de l'infection (lésions cutanées). Le prélèvement (matériel de lésion cutanée) a été transmis au **KEMRI** (Kenya Medical Research Institute) pour analyse.

L'équipe a appliqué le **séquençage métagénomique non ciblé** (mNGS) pour reconstruire le génome viral directement à partir de l'échantillon clinique, sur deux plateformes (Illumina et Nanopore).

Article de référence : Chomba et al., *Genomic sequence analysis of the first mpox virus detected in Kenya*, bioRxiv, 2024 — <https://www.biorxiv.org/content/10.1101/2024.08.20.608891v1>

---

## Métagénomique shotgun

MPOX est ici séquencé en **métagénomique shotgun** : on séquence tout l'ADN de l'échantillon (virus + hôte humain + autres), sans amplification ciblée. Conséquences pour l'analyse :

- **Pas d'étape de retrait d'amorces** (il n'y en a pas).
- **Beaucoup de lectures non virales** (surtout de l'hôte humain) : le taux d'alignement sur la référence virale est plus faible — c'est normal, la référence sert aussi à trier les lectures virales.
- Le génome MPOX est un **ADN double brin de ~197 kb**.

---

## Données

| Caractéristique | Valeur |
|---|---|
| Pathogène | MPXV (Monkeypox virus) |
| Origine | Kenya (cas index, juillet 2024) |
| Préparation | Métagénomique shotgun (enrichie) |
| Illumina (paired-end) | `SRR30229922` |
| Nanopore | `SRR32413059` (PromethION, R10.4.1 + kit v14) |
| BioProject | `PRJNA1147890` |
| Référence | **NC_003310.1** (Zaire-96-I-16, ~197 kb) |

> Le même échantillon a été séquencé sur les deux plateformes : idéal pour comparer Illumina et Nanopore sur des données identiques.

---

## Chaîne d'analyse

### Voie Illumina (lectures courtes)

| Étape | Outils |
|---|---|
| QC & nettoyage | FastQC, fastp, MultiQC |
| Mapping | Bowtie2 |
| Variants & consensus | BCFtools |
| De novo (alternative) | SPAdes, QUAST, BLASTn |

### Voie Nanopore (lectures longues)

| Étape | Outils |
|---|---|
| QC & nettoyage | NanoPlot, chopper |
| Mapping | minimap2 |
| Variants & consensus | Clair3, BCFtools |

---

## Architecture du dossier

```
MPOX/
├── environment.yml            ← environnement conda unique (mpox_analyse)
│
├── data/                      ← fichiers partagés (les deux plateformes)
│   └── reference.fasta            (NC_003310.1)
│
├── illumina/                  ← voie lectures courtes
│   ├── 01_qc/                     (fastp)
│   │   ├── README.md
│   │   └── clean/
│   ├── 02_mapping/                (bowtie2 → bcftools → consensus)
│   └── 03_denovo/                 (Kraken2 → SPAdes → BLASTn → QUAST)
│
├── nanopore/                  ← voie lectures longues
│   ├── 01_qc/                     (NanoPlot + chopper)
│   └── 02_mapping/                (minimap2 → Clair3 → bcftools)
│
└── README.md                  ← ce fichier
```

Les fichiers partagés (référence, environnement) sont à la racine `MPOX/`. Depuis une étape (ex. `illumina/01_qc/`), on y accède par `../../data/`.

---

## Environnement de travail

Un seul environnement conda regroupe tous les outils (Illumina + Nanopore) :

```bash
cd ~/bioinfo_practice/MPOX
mamba env create -f environment.yml
conda activate mpox_analyse
```

---

## Télécharger les données

Toutes les données se placent dans `data/` (partagé entre les deux plateformes).

```bash
cd ~/bioinfo_practice/MPOX/data
```

**1. Lectures Illumina** (paired-end → 2 fichiers) :

```bash
fasterq-dump --split-files SRR30229922
gzip SRR30229922_1.fastq SRR30229922_2.fastq
```

**2. Lectures Nanopore** (single → 1 fichier) :

```bash
fasterq-dump SRR32413059
gzip SRR32413059.fastq
```

**3. Référence** (NC_003310.1, ~197 kb) :

```bash
# via datasets, efetch, ou lien NCBI direct selon l'outil disponible
# à enregistrer sous : reference.fasta
```

**4. Modèle Clair3** (pour la voie Nanopore — R10.4.1 + kit v14) :

Le modèle est déjà embarqué dans l'installation conda de Clair3. On le copie (pas de téléchargement) :

```bash
cp -r "$(dirname "$(which run_clair3.sh)")/models/r1041_e82_400bps_sup_v520" .
```

Vérifier que tout est en place :

```bash
ls -lh
```

Attendu : `SRR30229922_{1,2}.fastq.gz`, `SRR32413059.fastq.gz`, `reference.fasta`, et le dossier du modèle Clair3.

---

## Glossaire

Les termes techniques sont définis dans le **[`GLOSSAIRE.md`](../SC2/GLOSSAIRE.md)** (générique, valable pour tous les pathogènes).