# MPOX · Nanopore · Étape 01 — Contrôle qualité (NanoPlot)

Données : MPXV clade Ib (Nanopore, lectures longues, métagénomique shotgun)
Sortie : lectures filtrées → entrée de l'étape `02_mapping`

---

## Objectif

Évaluer la qualité des lectures longues Nanopore, puis filtrer les plus faibles.

```
fastq Nanopore
   → NanoPlot (diagnostic)
   → chopper (filtrage qualité/longueur)
   → NanoPlot (contrôle)
```

---

## Lectures longues : ce qui change

Les lectures Nanopore sont **longues** (jusqu'à des dizaines de kb) mais **moins précises** que l'Illumina (qualité typique Q10-20, erreurs surtout de type indel dans les homopolymères).

- **Outil QC** : NanoPlot (pas FastQC).
- **Seuils** : on raisonne en Q ~10 (pas Q30) et en longueur.
- **Force** : les longues lectures franchissent les régions répétées, utiles pour un génome large comme MPOX (~197 kb).

---

## Prérequis

```bash
conda activate mpox_analyse
NanoPlot --version
chopper --version
```

```bash
cd ~/bioinfo_practice/MPOX/nanopore/01_qc
mkdir -p nanoplot_avant clean nanoplot_apres
```

---

## Étape 0 — Récupérer les données Nanopore

L'échantillon Nanopore (même échantillon que l'Illumina, plateforme différente).

```bash
cd ~/bioinfo_practice/MPOX/data
fasterq-dump SRR32413059
gzip SRR32413059.fastq
ls -lh
```

> Les données Nanopore sont en lectures simples (pas de paired-end) : un seul fichier `.fastq`.

---

## Étape 1 — Diagnostic (NanoPlot)

```bash
cd ~/bioinfo_practice/MPOX/nanopore/01_qc

NanoPlot \
  --fastq ../../data/SRR32413059.fastq.gz \
  --outdir nanoplot_avant \
  --threads 4
```

**Ce que produit NanoPlot** : un rapport HTML avec la distribution des longueurs, la distribution des qualités, et le graphe longueur × qualité.

> **Lecture** — On regarde le nombre total de lectures, la longueur médiane, la qualité médiane, et le N50 des lectures (longueur telle que 50 % des bases sont dans des lectures ≥ cette taille).

---

## Étape 2 — Filtrage qualité/longueur (chopper)

chopper filtre les lectures selon qualité et longueur.

```bash
zcat ../../data/SRR32413059.fastq.gz \
  | chopper -q 10 -l 200 \
  | gzip > clean/SRR32413059.clean.fastq.gz
```

**Décorticage** :
- `chopper -q 10` — garde les lectures de qualité moyenne ≥ 10 (seuil réaliste Nanopore, pas Q30).
- `-l 200` — garde les lectures ≥ 200 bases (élimine les fragments trop courts).
- On décompresse (`zcat`), on filtre, on recompresse (`gzip`) — un enchaînement par tubes.

> **Seuils Nanopore ≠ Illumina** — Exiger Q30 sur du Nanopore éliminerait presque tout. Q10 est un seuil réaliste. À ajuster selon la qualité réelle observée dans NanoPlot.

---

## Étape 3 — Contrôle après filtrage

```bash
NanoPlot \
  --fastq clean/SRR32413059.clean.fastq.gz \
  --outdir nanoplot_apres \
  --threads 4
```

> **✅ Attendu** — Qualité médiane remontée, lectures trop courtes éliminées, distribution resserrée. On compare les rapports avant/après.

---

## Récapitulatif

| Étape | Outil | Sortie |
|---|---|---|
| Diagnostic | NanoPlot | rapport HTML (longueur × qualité) |
| Filtrage | chopper (`-q 10 -l 200`) | `.clean.fastq.gz` |
| Contrôle | NanoPlot | rapport HTML |

**Sortie clé** : `clean/SRR32413059.clean.fastq.gz` → entrée de `02_mapping`.

---

## Points de vigilance

- **NanoPlot, pas FastQC** : outil adapté aux lectures longues.
- **Seuils Nanopore** : Q10 (pas Q30), on raisonne aussi en longueur.
- **La longueur est une force** : les longues lectures franchissent les régions répétées.
- **Ne pas sur-filtrer** : en shotgun, les lectures virales sont déjà minoritaires.
