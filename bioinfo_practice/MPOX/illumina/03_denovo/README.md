# MPOX · Illumina · Étape 03 — Assemblage de novo (SPAdes)

Données : MPXV (Illumina paired-end, capture ciblée — Viral Surveillance Panel)
Entrée : lectures nettoyées de l'étape `01_qc`

---

## Objectif

Reconstruire le génome MPOX **sans référence**, en assemblant directement les lectures, puis situer et évaluer l'assemblage.

```
lectures nettoyées
   → (vérification : proportion de lectures virales)
   → SPAdes (assemblage → contigs)
   → BLASTn (situer les contigs vs référence)
   → QUAST (évaluer la qualité)
```

> **Note sur ces données** — Les lectures déposées dans le SRA pour cet échantillon sont déjà la fraction nettoyée : dans l'étude d'origine, la majorité des lectures brutes (hôte humain, basse qualité, duplicats) ont été retirées, ne conservant que quelques dizaines de milliers de lectures majoritairement virales. On peut donc assembler directement, après une vérification rapide.

> **Lien avec le choix de référence** — Le de novo est aussi la voie à suivre **quand on ne connaît pas d'avance quel génome de référence utiliser** (par exemple, quel clade de MPOX). On assemble sans a priori, puis on identifie le résultat (BLASTn, Nextclade). C'est seulement une fois le clade connu qu'on peut choisir la bonne référence pour un mapping.

---

## Prérequis

```bash
conda activate mpox_analyse
cd ~/bioinfo_practice/MPOX/illumina/03_denovo
mkdir -p results
```

Fichiers nécessaires :
- lectures nettoyées : `../01_qc/clean/SRR30229922_{1,2}.clean.fastq.gz`
- référence : `../../data/reference.fasta` (pour BLASTn)

---

## Étape 1 — Vérifier la proportion de lectures virales

Avant d'assembler, on s'assure que les lectures sont majoritairement du MPOX (et non de l'hôte). Le taux d'alignement mesuré à l'étape 02 (`samtools flagstat`) répond à cette question.

```bash
cd ~/bioinfo_practice/MPOX/illumina/03_denovo
```

> **Lecture** — Si le `flagstat` de l'étape 02 montrait un taux d'alignement élevé sur la référence MPOX, les lectures sont bien majoritairement virales : on peut assembler directement. Si le taux était très faible, l'assemblage risquerait d'être dominé par des séquences non virales.

> **Important — pour vos propres données**
> Les lectures de ce jeu public sont déjà nettoyées (fraction virale conservée). **Mais vos propres échantillons cliniques bruts ne le seront pas** : ils contiennent une majorité de lectures humaines. Avant d'assembler vos données, vous **devez** retirer les lectures de l'hôte, typiquement avec **Kraken2** (base humaine) :
> ```bash
> kraken2 --db kraken2_human_db --paired \
>   --unclassified-out reads_unclass#.fastq \
>   R1.clean.fastq.gz R2.clean.fastq.gz
> ```
> puis assembler les lectures « unclassified » (non humaines). Sans ce dé-hôtage, l'assemblage sera dominé par le génome humain.


## Étape 2 — Assemblage de novo (SPAdes)

SPAdes assemble directement les lectures nettoyées en contigs.

```bash
spades.py \
  --isolate \
  -1 ../01_qc/clean/SRR30229922_1.clean.fastq.gz \
  -2 ../01_qc/clean/SRR30229922_2.clean.fastq.gz \
  -t 4 \
  -o results/spades_SRR30229922
```

**Décorticage** :
- `-1` / `-2` — lectures paired-end nettoyées.
- `--isolate` — mode SPAdes adapté ADN pur ou presque.
- `-t 4` — nombre de cœurs (threads) utilisés en parallèle.
- `-o` — dossier de sortie.

Le résultat clé : `results/spades_SRR30229922/contigs.fasta`.

---

## Étape 3 — Inspecter les contigs

```bash
# nombre de contigs
grep -c ">" results/spades_SRR30229922/contigs.fasta

# les plus longs (SPAdes les trie par longueur)
grep ">" results/spades_SRR30229922/contigs.fasta | head -10
```

## Étape 4 — Situer les contigs (BLASTn)

On vérifie que les contigs correspondent bien à MPOX et où ils tombent sur la référence.

```bash
# base BLAST à partir de la référence
makeblastdb -in ../../data/reference.fasta -dbtype nucl -out results/mpxv_blastdb

# aligner les contigs sur la référence
blastn \
  -query results/spades_SRR30229922/contigs.fasta \
  -db results/mpxv_blastdb \
  -outfmt "6 qseqid sseqid pident length evalue bitscore" \
  -out results/SRR30229922.blastn.tsv

# meilleurs hits
column -t results/SRR30229922.blastn.tsv | head -20
```

**Décorticage** :
- `makeblastdb` — construit une base BLAST à partir de la référence.
- `blastn` — aligne chaque contig sur la référence.
- `-outfmt "6 ..."` — sortie tabulée : identité (`pident`), longueur, e-value, score.

> **Lecture** — Une identité élevée (%) confirme que les contigs sont bien du MPXV. Les contigs qui ne matchent pas = contamination éventuelle.

---

## Étape 5 — Évaluer l'assemblage

**Option A — QUAST** (Linux / WSL2 uniquement). QUAST produit un rapport complet, mais peut ne tourner sur macOS.

```bash
quast.py \
  results/spades_SRR30229922/contigs.fasta \
  -r ../../data/reference.fasta \
  -o results/quast_SRR30229922
```

**Option B — Commandes de base** (tous OS). Si QUAST n'est pas installé, on obtient les mêmes métriques essentielles à la main :

```bash
# nombre de contigs
grep -c ">" results/spades_SRR30229922/contigs.fasta

# longueur totale de l'assemblage
grep -v ">" results/spades_SRR30229922/contigs.fasta | tr -d '\n' | wc -c

# N50 (contiguïté) : longueur telle que 50 % de l'assemblage est dans des contigs >= cette taille

grep ">" results/spades_SRR30229922/contigs.fasta \
  | sed 's/.*length_\([0-9]*\)_.*/\1/' \
  | sort -rn \
  | awk '{a[NR]=$1; sum+=$1} END {half=sum/2; run=0; for(i=1;i<=NR;i++){run+=a[i]; if(run>=half){print "N50 :", a[i]; break}}}'
```

**Métriques clés** :
- **N50** — longueur telle que 50 % de l'assemblage est dans des contigs ≥ cette taille. Plus grand = mieux assemblé.
- **# contigs** — moins il y en a, mieux c'est (à couverture égale).
- **Longueur totale** — doit approcher ~197 kb pour un génome complet.
- **Identité vs référence** — donnée par le BLASTn de l'étape 4.

> **Lecture** — Ces métriques disent si l'assemblage est complet et contigu, ou fragmenté. Un assemblage à couverture modérée est souvent fragmenté (plusieurs contigs) tout en couvrant la quasi-totalité du génome.

---

## Récapitulatif

| Étape | Commande clé | Sortie |
|---|---|---|
| Vérifier | `samtools flagstat` (étape 02) | proportion de lectures virales |
| Assemblage | `spades.py --isolate` | `contigs.fasta` |
| Inspection | `grep -c ">"` | nombre de contigs |
| Situer | `blastn` vs référence | `.blastn.tsv` |
| Évaluer | `quast.py` (Linux/WSL2) ou commandes de base | N50, longueur, # contigs |

---

## Points de vigilance

- **De novo ≠ mapping** : complémentaires. Le de novo révèle ce que la référence ne contient pas, et sert quand on ne connaît pas d'avance la bonne référence.
- **N50 = contiguïté** : la métrique reine de l'assemblage.
- **Contigs multiples = fragmentation** : normal si couverture faible ; on situe avec BLASTn.
- **BLASTn valide l'identité** : confirme que les contigs sont bien viraux.
- **Évaluer objectivement** : ne pas juger un assemblage « à l'œil ». QUAST (Linux/WSL2) ou les commandes de base donnent les métriques.