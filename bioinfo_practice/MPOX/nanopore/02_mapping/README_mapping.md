# MPOX · Nanopore · Étape 02 — Alignement, variants & consensus

Données : MPXV clade Ib (métagénomique long-read — Nanopore)
Entrée : lectures filtrées de l'étape `01_qc`
Sortie : génome consensus

---

## Objectif

Aligner les lectures longues sur la référence MPOX, appeler les variants avec un modèle adapté au Nanopore, générer le consensus.

```
lectures filtrées
   → minimap2 (alignement, mode map-ont)
   → samtools (tri + index)
   → Clair3 (variants, modèle ONT, mode haploïde)
   → bcftools consensus (+ masquage des zones peu couvertes)
```

---

## Prérequis

```bash
conda activate mpox_analyse
cd ~/bioinfo_practice/MPOX/nanopore/02_mapping
mkdir -p results
```

Fichiers nécessaires :
- lectures filtrées : `../01_qc/clean/SRR32413059.clean.fastq.gz`
- référence : `../../data/reference.fasta` (NC_003310.1)
- modèle Clair3 (voir étape 0)

---

## Étape 0 — Le modèle Clair3

Clair3 exige un **modèle** correspondant à la chimie de la flowcell ET au basecaller.

Pour ces données : flowcell **R10.4.1** + **kit v14** + basecaller **SUP** → modèle **`r1041_e82_400bps_sup_v520`**.

> **Compatibilité de version — piège important.** Clair3 v2 utilise PyTorch ; les anciens modèles TensorFlow (v1) ne sont **pas compatibles**. Il faut un modèle PyTorch pour Clair3 v2. Bonne nouvelle : l'installation conda de Clair3 **embarque déjà les modèles compatibles**. On les copie depuis l'installation, sans rien télécharger :

```bash
cd ~/bioinfo_practice/MPOX/data

# copie le modèle depuis l'installation conda (chemin trouvé automatiquement)
cp -r "$(dirname "$(which run_clair3.sh)")/models/r1041_e82_400bps_sup_v520" .

# vérifier
ls r1041_e82_400bps_sup_v520/
```

> **Choisir le bon modèle** — Le nom encode la chimie : `r1041` = R10.4.1, `e82_400bps` = kit v14, `sup` = basecaller super-accurate, `v520` = version. Lister les modèles disponibles : `ls "$(dirname "$(which run_clair3.sh)")/models/"`. Pour vos données, vérifier la chimie dans les métadonnées du run. Les variantes `_with_mv` (dwell-time) exigent des tags `mv` de Dorado dans le BAM — non applicable aux données SRA standard.

---

## Étape 1 — Aligner (minimap2, mode map-ont)

minimap2 est l'aligneur de référence pour lectures longues.

```bash
cd ~/bioinfo_practice/MPOX/nanopore/02_mapping

minimap2 \
  -a \
  -x map-ont \
  --MD \
  ../../data/reference.fasta \
  ../01_qc/clean/SRR32413059.clean.fastq.gz \
  | samtools sort -o results/SRR32413059.sorted.bam

samtools index results/SRR32413059.sorted.bam
samtools flagstat results/SRR32413059.sorted.bam
```

**Décorticage** :
- `-a` — sortie au format SAM (pour piper vers samtools).
- `-x map-ont` — préréglage lectures longues ONT (tolère les erreurs, indels).
- `--MD` — ajoute le tag MD (utile pour l'appel de variants).
- `| samtools sort` — trie en BAM.

---

## Étape 2 — Mesurer la couverture

Important en Nanopore : la profondeur guide le seuil de masquage du consensus.

```bash
# profondeur moyenne
samtools depth -a results/SRR32413059.sorted.bam \
  | awk '{sum+=$3; n++} END {print "Profondeur moyenne :", sum/n}'

# fraction couverte (>=1x)
samtools depth -a results/SRR32413059.sorted.bam \
  | awk '$3>0 {c++} END {print "Positions couvertes :", c}'
```

> **Adapter le seuil au Nanopore** — Le Nanopore a souvent une profondeur plus faible que l'Illumina (moins de reads, plus longs). Si la profondeur moyenne est basse (~10×), un seuil de masquage à 10× masquerait une grande partie du génome. On l'abaisse (ex. 5×) — acceptable car la qualité par base du R10.4.1 est bonne. À ajuster selon la profondeur observée.

---

## Étape 3 — Appeler les variants (Clair3, mode haploïde)

Clair3 avec les options adaptées à un génome viral haploïde. On utilise des **chemins absolus** (Clair3 y est sensible), générés automatiquement par `$PWD` et `$(cd ... && pwd)`.

```bash
run_clair3.sh \
  --bam_fn="$PWD/results/SRR32413059.sorted.bam" \
  --ref_fn="$(cd ../../data && pwd)/reference.fasta" \
  --threads=4 \
  --platform=ont \
  --model_path="$(cd ../../data && pwd)/r1041_e82_400bps_sup_v520" \
  --output="$PWD/results/clair3_SRR32413059" \
  --include_all_ctgs \
  --haploid_precise \
  --no_phasing_for_fa
```

**Décorticage** (options clés pour un virus) :
- `--platform=ont` — données Nanopore.
- `--model_path` — le modèle correspondant à la chimie (étape 0).
- `--include_all_ctgs` — appelle sur tous les contigs de la référence.
- `--haploid_precise` — **mode haploïde** : un virus n'a qu'une copie du génome.
- `--no_phasing_for_fa` — désactive le phasing (inutile en haploïde).

Sortie clé : `results/clair3_SRR32413059/merge_output.vcf.gz`.

> **Chemins absolus** — Clair3 exige des chemins absolus pour `--bam_fn`, `--ref_fn`, `--model_path`, `--output`. `$PWD` et `$(cd DOSSIER && pwd)` les génèrent sans avoir à les taper.

---

## Étape 4 — Générer le consensus (bcftools + masquage)

```bash
# indexer le VCF de Clair3
bcftools index results/clair3_SRR32413059/merge_output.vcf.gz

# zones peu couvertes (< 4, seuil adapté au Nanopore) à masquer
samtools depth -a results/SRR32413059.sorted.bam \
  | awk '$3 < 4 {print $1"\t"$2-1"\t"$2}' > results/low_cov.bed

# consensus en masquant ces zones
bcftools consensus \
  -f ../../data/reference.fasta \
  -m results/low_cov.bed \
  results/clair3_SRR32413059/merge_output.vcf.gz \
  > results/SRR32413059.consensus.fa
```

**Décorticage** :
- `samtools depth` + `awk '$3 < 4'` — construit un BED des positions < 4× (à masquer). Seuil bas adapté à la profondeur Nanopore.
- `bcftools consensus -m` — applique les variants et masque (N) les zones peu couvertes.

---

## Étape 5 — Évaluer

```bash
# nombre de N
grep -v ">" results/SRR32413059.consensus.fa | tr -d '\n' | tr -cd 'Nn' | wc -c

# longueur totale
grep -v ">" results/SRR32413059.consensus.fa | tr -d '\n' | wc -c
```

---

## Récapitulatif

| Étape | Commande clé | Sortie |
|---|---|---|
| Modèle Clair3 | `cp -r "$(dirname ...)/models/..."` | dossier modèle |
| Aligner | `minimap2 -x map-ont \| samtools sort` | `.sorted.bam` |
| Couverture | `samtools depth` | profondeur, fraction |
| Variants | `run_clair3.sh --haploid_precise` | `merge_output.vcf.gz` |
| Consensus | `bcftools consensus -m low_cov.bed` | `.consensus.fa` |