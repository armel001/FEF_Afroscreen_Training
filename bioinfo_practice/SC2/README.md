# Cas d'étude — SARS-CoV-2

**Atelier FEF · Afroscreen — Session Bio-informatique**

Analyse complète d'un échantillon SARS-CoV-2 réel, du fichier brut de séquençage au génome consensus et à la liste des variants.

---

## Une approche standard et transférable

Cette analyse suit le schéma **standard de la génomique virale par mapping sur référence** :

```
QC & nettoyage → alignement sur référence → consensus → appel de variants
```

Ce schéma n'est pas spécifique au SARS-CoV-2. Il s'applique à **tout pathogène viral séquencé selon la même logique** (lectures courtes appariées, séquençage ciblé ou sur génome de référence connu) : autres virus respiratoires, arbovirus, virus des fièvres hémorragiques, etc. Pour passer d'un pathogène à un autre, on adapte essentiellement **trois éléments** :

- la **référence** (`reference.fasta`) et son **annotation** (`reference.gff3`) propres au pathogène ;
- le **schéma d'amorces** si le séquençage est en amplicon (sinon, cette étape saute) ;
- éventuellement les **seuils** (qualité, couverture, fréquence) selon la profondeur et la nature des données.

Les outils, la structure du dossier et l'enchaînement des étapes restent, eux, identiques. Comprendre ce cas SARS-CoV-2, c'est donc acquérir une méthode réutilisable — pas une recette à usage unique.

---

## Données

| Caractéristique | Valeur |
|---|---|
| Pathogène | SARS-CoV-2 |
| Origine | Afrique du Sud (nov.–déc. 2021) | https://www.ncbi.nlm.nih.gov/sra/?term=SRR17051908
| Plateforme | Illumina, paired-end |
| Protocole | Amplicon ARTIC |
| Accession (exemple) | `SRR17051908` |
| Rechange (même lot) | `SRR17051923`, `SRR17051916`, `SRR17051953` |
| Référence | Wuhan-Hu-1 (`MN908947.3`) |

---

## Chaîne d'analyse

| Étape | Outils | Paramètres clés |
|---|---|---|
| QC | FastQC, MultiQC | — |
| Nettoyage | Cutadapt (adaptateurs + amorces), Sickle | `Q30`, `longueur 25` |
| Mapping | BWA | référence Wuhan-Hu-1 |
| Couverture | Bedtools | `couverture min 10` |
| Variants | ivar | `couverture 10`, `fréquence 0.2` |
| Consensus | ivar | `couverture 10` → N |

> Le retrait des amorces est fait au QC (Cutadapt). Une méthode alternative après alignement (Bamclipper, format BEDPE) est décrite dans l'étape `02_mapping` mais non exécutée, pour ne pas retirer les amorces deux fois.

---

## Architecture du dossier

```
SC2/
├── environment.yml       ← environnement conda unique (sc2_analyse)
│
├── data/                 ← fichiers partagés entre étapes
│   ├── reference.fasta       (Wuhan-Hu-1, MN908947.3)
│   ├── reference.gff3        (annotation des gènes)
│   └── primers.bedpe         (amorces ARTIC, format BEDPE — méthode alternative)
│
├── 01_qc/                ← QC + nettoyage
│   ├── README.md
│   └── clean/                (fastq nettoyés → entrée de 02)
│
├── 02_mapping/           ← alignement + couverture + consensus
│   ├── README.md
│   └── results/
│
├── 03_variants/          ← appel de variants
│   ├── README.md
│   └── results/
│
└── README.md             ← ce fichier
```

---

## Enchaînement des étapes

| Étape | Dossier | Entrée | Sortie |
|---|---|---|---|
| 1. QC & nettoyage | `01_qc/` | fastq bruts | fastq nettoyés |
| 2. Mapping & consensus | `02_mapping/` | fastq nettoyés | BAM + consensus |
| 3. Variants | `03_variants/` | BAM aligné | tableau `.tsv` |

La sortie de chaque étape est l'entrée de la suivante.

---

## Environnement de travail

Un seul environnement conda regroupe tous les outils de l'analyse SC2 :

```bash
cd ~/pratique/SC2
mamba env create -f environment.yml
conda activate sc2_analyse
```

---

## Glossaire

Les termes techniques (formats de fichiers, outils, concepts, paramètres) sont définis dans **[`GLOSSAIRE.md`](GLOSSAIRE.md)** 