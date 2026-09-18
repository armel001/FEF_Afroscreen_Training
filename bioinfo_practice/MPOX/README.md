# Cas d'étude — MPOX (Monkeypox virus)

**Formation FEF · Afroscreen — Bio-informatique**

Analyse d'un échantillon MPOX réel, du fichier brut de séquençage au génome consensus, sur **deux plateformes** (Illumina et Nanopore) à partir du même échantillon.

---

## Contexte des données

Ces données proviennent d'un cas de mpox détecté au **Kenya** en juillet 2024 — un chauffeur routier longue distance avec un historique de voyage vers l'Ouganda, présentant les symptômes classiques de l'infection (lésions cutanées). Le prélèvement (écouvillon de lésion cutanée) a été collecté le 25 juillet 2024 et analysé au **KEMRI** (Kenya Medical Research Institute).

L'échantillon a été confirmé positif au MPXV par qPCR (gène F3L, Ct ≈ 21), puis séquencé sur **deux plateformes**, Illumina (lectures courtes) et Nanopore (lectures longues), pour reconstruire le génome viral.

Article de référence :
Langat *et al.*, *Complete genome of an mpox clade 1b virus from Kenya*, **Microbiology Resource Announcements**, 2025 — <https://doi.org/10.1128/mra.00050-25>


## Données

| Caractéristique | Valeur |
|---|---|
| Pathogène | MPXV (Monkeypox virus) |
| Origine | Kenya (juillet 2024) |
| Illumina — préparation | NEBNext + **Viral Surveillance Panel** (capture ciblée) |
| Illumina — séquenceur | NextSeq 2000, paired-end |
| Illumina — accession | `SRR30229922` |
| Nanopore — préparation | native barcoding kit v14 **Shotgun metagenomic** |
| Nanopore — séquenceur | PromethION, flowcell R10.4.1 |
| Nanopore — accession | `SRR32413059` |
| BioProject | `PRJNA1147890` |
| Référence | **NC_003310.1** |

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
├── illumina/                  
│   ├── 01_qc/                     
│   │   ├── README.md
│   │   └── clean/
│   ├── 02_mapping/                
│   └── 03_denovo/   
│
├── nanopore/             
│   ├── 01_qc/             
│   └── 02_mapping/  
│
└── README.md 
```

## Environnement de travail

Un seul environnement conda regroupe tous les outils (Illumina + Nanopore) :

```bash
cd ~/bioinfo_practice/MPOX
mamba env create -f environment.yml
conda activate mpox_analyse
```

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