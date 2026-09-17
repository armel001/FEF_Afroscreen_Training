# MPOX · Illumina · Étape 02 — Alignement, variants & consensus

Données : MPXV clade Ib (Illumina paired-end, métagénomique shotgun)
Entrée : lectures nettoyées de l'étape `01_qc`
Sortie : génome consensus

---

## Objectif

Aligner les lectures nettoyées sur la référence MPOX, appeler les variants et générer le consensus.

```
lectures nettoyées
   → Bowtie2 (alignement)
   → samtools (tri + index)
   → bcftools (variants)
   → bcftools consensus (+ masquage des zones peu couvertes)
```

---

## Prérequis

```bash
conda activate mpox_analyse
cd ~/bioinfo_practice/MPOX/illumina/02_mapping
mkdir -p results
```

Fichiers nécessaires :
- lectures nettoyées : `../01_qc/clean/SRR30229922_{1,2}.clean.fastq.gz`
- référence : `../../data/reference.fasta` (NC_003310.1)

---

## Étape 0 — Récupérer la référence (NC_003310.1)

Le génome de référence clade I (Zaire-96-I-16, ~197 kb).

```bash
cd ~/bioinfo_practice/MPOX/data
# télécharger reference.fasta = NC_003310.1
# (via datasets, efetch, ou lien NCBI direct selon l'outil disponible)
head -1 reference.fasta
```

---

## Étape 1 — Indexer la référence (Bowtie2)

Bowtie2 construit son propre index.

```bash
cd ~/bioinfo_practice/MPOX/illumina/02_mapping

# index bowtie2 (préfixe "mpxv_ref")
bowtie2-build ../../data/reference.fasta results/mpxv_ref

# index samtools (pour la suite)
samtools faidx ../../data/reference.fasta
```

---

## Étape 2 — Aligner (Bowtie2)

On aligne toutes les lectures nettoyées sur la référence MPOX. Les lectures non virales (hôte humain…) ne s'alignent pas et sont écartées naturellement.

```bash
bowtie2 \
  -x results/mpxv_ref \
  -1 ../01_qc/clean/SRR30229922_1.clean.fastq.gz \
  -2 ../01_qc/clean/SRR30229922_2.clean.fastq.gz \
  --threads 4 \
  | samtools sort -o results/SRR30229922.sorted.bam

samtools index results/SRR30229922.sorted.bam
samtools flagstat results/SRR30229922.sorted.bam
```

**Décorticage** :
- `-x results/mpxv_ref` — préfixe de l'index bowtie2.
- `-1` / `-2` — lectures paired-end R1 / R2.
- `--threads 4` — parallélisation.
- `| samtools sort` — trie directement en BAM.

> **Lecture shotgun** — Le taux de lectures alignées peut être faible : en shotgun, seule une fraction des lectures est virale (le reste = hôte, autres organismes). `samtools flagstat` révèle cette proportion. Un faible % aligné n'est pas un échec — c'est la nature du shotgun.

---

## Étape 3 — Couverture (Bedtools)

```bash
bedtools genomecov \
  -ibam results/SRR30229922.sorted.bam \
  -bga \
  > results/SRR30229922.coverage.bedgraph

# zones faiblement couvertes (< 10) = futures zones N
awk '$4 < 10' results/SRR30229922.coverage.bedgraph | head -20

# profondeur moyenne
samtools depth -a results/SRR30229922.sorted.bam \
  | awk '{sum+=$3; n++} END {print "Profondeur moyenne :", sum/n}'
```

> **Lecture** — Sur un génome de ~197 kb en shotgun, la couverture peut être partielle si la charge virale de l'échantillon est faible. Les zones < 10 deviendront des N dans le consensus.

---

## Étape 4 — Appeler les variants (BCFtools)

```bash
bcftools mpileup \
  -f ../../data/reference.fasta \
  --max-depth 0 \
  --annotate FORMAT/AD,FORMAT/DP \
  results/SRR30229922.sorted.bam \
  | bcftools call \
      --ploidy 1 \
      --multiallelic-caller \
      --variants-only \
      -Oz -o results/SRR30229922.vcf.gz

bcftools index results/SRR30229922.vcf.gz
```

**Décorticage** :
- `bcftools mpileup -f` — empile les lectures sur la référence.
- `--max-depth 0` — pas de limite de profondeur.
- `--annotate FORMAT/AD,FORMAT/DP` — ajoute profondeur allélique (AD) et totale (DP).
- `bcftools call --ploidy 1` — **virus = haploïde** (une seule copie du génome).
- `--multiallelic-caller` — modèle d'appel recommandé.
- `--variants-only` — ne garde que les positions variantes.
- `-Oz` — sortie compressée (VCF.gz).

> **Ploïdie 1** — Un virus n'a qu'une copie de son génome. Le mode haploïde évite d'appeler de faux hétérozygotes.

---

## Étape 5 — Filtrer les variants

```bash
bcftools view -e 'INFO/DP<10 || QUAL<20' results/SRR30229922.vcf.gz \
  -Oz -o results/SRR30229922.filtered.vcf.gz
bcftools index results/SRR30229922.filtered.vcf.gz

# compter les variants retenus
bcftools view -H results/SRR30229922.filtered.vcf.gz | wc -l
```

**Décorticage** :
- `-e 'INFO/DP<10 || QUAL<20'` — exclut les variants peu profonds (< 10) ou peu fiables (qualité < 20).

---

## Étape 6 — Générer le consensus (BCFtools + masquage)

Les zones peu couvertes doivent être masquées (N) avant de générer le consensus.

```bash
# 1. zones à couverture < 10 (à masquer)
bedtools genomecov -ibam results/SRR30229922.sorted.bam -bga \
  | awk '$4 < 10' > results/low_cov.bed

# 2. consensus en masquant ces zones
bcftools consensus \
  -f ../../data/reference.fasta \
  -m results/low_cov.bed \
  results/SRR30229922.filtered.vcf.gz \
  > results/SRR30229922.consensus.fa
```

**Décorticage** :
- `bedtools genomecov` + `awk` — liste les régions sous 10× de couverture.
- `bcftools consensus -m` — applique les variants et masque (N) les régions peu couvertes.

> **Pourquoi masquer ?** En shotgun, la couverture est inégale. Sans masquage, les zones non couvertes hériteraient de la séquence de référence — créant un faux consensus. Le masquage en N indique honnêtement « ici, on ne sait pas ».

---

## Étape 7 — Évaluer le consensus

```bash
# nombre de N
grep -v ">" results/SRR30229922.consensus.fa | tr -d '\n' | tr -cd 'Nn' | wc -c

# longueur totale
grep -v ">" results/SRR30229922.consensus.fa | tr -d '\n' | wc -c
```

> **Lecture** — Sur ~197 kb, le nombre de N indique la complétude du génome reconstruit. Beaucoup de N = charge virale faible ou couverture insuffisante.

---

## Récapitulatif

| Étape | Commande clé | Sortie |
|---|---|---|
| Indexer | `bowtie2-build` + `samtools faidx` | index |
| Aligner | `bowtie2 \| samtools sort` | `.sorted.bam` |
| Couverture | `bedtools genomecov` | `.bedgraph` |
| Variants | `bcftools mpileup \| bcftools call --ploidy 1` | `.vcf.gz` |
| Filtrer | `bcftools view -e ...` | `.filtered.vcf.gz` |
| Consensus | `bedtools` + `bcftools consensus -m` | `.consensus.fa` |

**Sortie clé** : `results/SRR30229922.consensus.fa`.

---

## Points de vigilance

- **Bowtie2 pour l'alignement**, BCFtools pour variants et consensus (voie shotgun).
- **Ploïdie 1** : un virus est haploïde.
- **Faible % aligné = normal en shotgun** : la plupart des lectures ne sont pas virales.
- **Masquage obligatoire** : sans masquer les zones peu couvertes, le consensus hérite faussement de la référence.
- **Couverture partielle possible** : selon la charge virale, le génome peut ne pas être couvert entièrement.