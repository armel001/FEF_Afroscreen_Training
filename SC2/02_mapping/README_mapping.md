# SC2 · Étape 02 — Alignement & consensus

Données : SARS-CoV-2 (Illumina paired-end, amplicon ARTIC)
Entrée : lectures nettoyées de l'étape `01_qc`
Sortie : génome consensus → base de l'étape `03_variants`

---

## Objectif

Aligner les lectures nettoyées sur la référence, mesurer la couverture, produire le consensus. (Les amorces ont déjà été retirées au QC.)

```
lectures nettoyées
   → BWA (alignement)
   → samtools (tri + index)
   → Bedtools (couverture)
   → ivar consensus
```

---

## Prérequis

```bash
conda activate sc2_analyse
cd ~/pratique/SC2/02_mapping
mkdir -p results
```

Fichiers nécessaires :
- lectures nettoyées : `../01_qc/clean/SRR17051908_{1,2}.clean.fastq.gz`
- référence : `../data/reference.fasta` (Wuhan-WIV04-2019, MN996528.1)
- amorces : `../data/primers.bedpe`

---

## Étape 1 — Indexer la référence

```bash
bwa index ../data/reference.fasta
samtools faidx ../data/reference.fasta
```

`bwa index` crée les fichiers auxiliaires (`.amb`, `.ann`, `.bwt`, `.pac`, `.sa`) à côté de la référence. On n'y touche pas.

---

## Étape 2 — Aligner (BWA-mem)

```bash
bwa mem \
  ../data/reference.fasta \
  ../01_qc/clean/SRR17051908_1.clean.fastq.gz \
  ../01_qc/clean/SRR17051908_2.clean.fastq.gz \
  | samtools sort -o results/SRR17051908.sorted.bam

samtools index results/SRR17051908.sorted.bam
samtools flagstat results/SRR17051908.sorted.bam
```

**Décorticage** :
- `bwa mem référence R1 R2` — aligne les paires de lectures sur la référence.
- `| samtools sort` — trie les alignements par position (indispensable pour la suite).
- `samtools index` — crée l'index `.bai`.
- `samtools flagstat` — statistiques : combien de lectures alignées.

---

## Étape 3 — Note sur le retrait des amorces

Les amorces ARTIC ont **déjà été retirées à l'étape 01** (QC), par `cutadapt -u 30 -U 30` : les 30 premières bases de chaque lecture, correspondant aux amorces, ont été coupées. Le FastQC après nettoyage l'a confirmé (le biais de composition en début de read a disparu). **On ne les retire donc pas une seconde fois ici.**

> **Méthode alternative (non exécutée ici) — Bamclipper**
> Une approche plus précise consiste à retirer les amorces *après* l'alignement, avec **Bamclipper**, à partir d'un fichier **BEDPE** décrivant la position exacte de chaque paire d'amorces (schéma ARTIC V4.1) :
> ```bash
> bamclipper.sh -b results/SRR17051908.sorted.bam -p ../data/primers.bedpe -n 4
> ```
> Bamclipper fait du *soft-clipping* : il masque les amorces aux vraies positions génomiques, au lieu de couper un nombre fixe de bases. C'est la méthode de référence quand on dispose du BEDPE. Comme les amorces sont déjà retirées au QC dans ce TP, on ne l'exécute pas, mais il est utile de connaître cette voie.

Le BAM utilisé pour la suite est donc directement `results/SRR17051908.sorted.bam`.

---

## Étape 4 — Couverture (Bedtools)

```bash
bedtools genomecov \
  -ibam results/SRR17051908.sorted.bam \
  -bga \
  > results/SRR17051908.coverage.bedgraph

# zones faiblement couvertes (< 10) = futures zones N
awk '$4 < 10' results/SRR17051908.coverage.bedgraph | head -20
```

**Décorticage** :
- `genomecov -bga` — couverture par position, y compris les zones à zéro.
- `awk '$4 < 10'` — repère les zones sous 10× (deviendront des N dans le consensus).

---

## Étape 5 — Générer le consensus (ivar)

```bash
samtools mpileup -aa -A -d 0 -Q 0 results/SRR17051908.sorted.bam \
  | ivar consensus -p results/SRR17051908.consensus -q 20 -t 0.6 -m 10 -n N
```

**Décorticage** :
- `-q 20` — qualité minimale des bases considérées.
- `-t 0.6` — une base est retenue si elle représente ≥ 60 % des lectures.
- `-m 10` — profondeur minimale 10 ; en dessous → N.
- `-n N` — caractère pour les positions non couvertes.

Consensus final : `results/SRR17051908.consensus.fa`.

---

## Étape 6 — Évaluer le consensus

```bash
# nombre de N
grep -v ">" results/SRR17051908.consensus.fa | tr -d '\n' | tr -cd 'Nn' | wc -c

# profondeur moyenne
samtools depth -a results/SRR17051908.sorted.bam \
  | awk '{sum+=$3; n++} END {print "Profondeur moyenne :", sum/n}'
```

> **Lecture** — Beaucoup de N = couverture insuffisante. Un bon consensus SC2 a peu de N.

---

## Récapitulatif

| Étape | Commande clé | Sortie |
|---|---|---|
| Indexer | `bwa index` + `samtools faidx` | index |
| Aligner | `bwa mem \| samtools sort` | `.sorted.bam` |
| Amorces | déjà retirées au QC (voir note étape 3) | — |
| Couverture | `bedtools genomecov -bga` | `.bedgraph` |
| Consensus | `samtools mpileup \| ivar consensus` | `.consensus.fa` |

**Sortie clé** : `results/SRR17051908.sorted.bam` → entrée de `03_variants`.