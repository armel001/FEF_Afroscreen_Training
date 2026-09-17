# MPOX · Illumina · Étape 03 — Assemblage de novo (SPAdes)

Données : MPXV clade Ib (Illumina paired-end, métagénomique shotgun)
Entrée : lectures nettoyées de l'étape `01_qc`

---

## Objectif

Reconstruire le génome MPOX **sans référence**, en assemblant directement les lectures, puis situer et évaluer l'assemblage.

```
lectures nettoyées
   → (vérification : données déjà virales)
   → SPAdes (assemblage → contigs)
   → BLASTn (situer les contigs vs référence)
   → QUAST (évaluer la qualité)
```

> Ces données ont été dé-hôtées avant dépôt (taille réduite, source « VIRAL RNA »). On ne refait donc pas de retrait de l'hôte : on vérifie d'abord que les lectures sont bien majoritairement virales, puis on assemble directement.

---

## Mapping vs de novo

| | Mapping (étape 02) | De novo (ici) |
|---|---|---|
| Principe | aligner sur une référence connue | reconstruire sans a priori |
| Question | « en quoi diffère-t-il de la référence ? » | « quelle est la séquence, indépendamment ? » |
| Force | rapide, précis si référence proche | détecte insertions, réarrangements, grandes délétions |
| Sortie | consensus aligné | contigs (fragments assemblés) |

Les deux sont complémentaires : le mapping ancre sur le connu, le de novo révèle ce que la référence ne contient pas.

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

## Étape 1 — Vérifier que les données sont bien virales

Avant d'assembler, on s'assure que les lectures sont majoritairement du MPOX (et non de l'hôte). On aligne rapidement sur la référence et on regarde le taux d'alignement.

```bash
cd ~/bioinfo_practice/MPOX/illumina/03_denovo

# le taux d'alignement sur MPOX a déjà été mesuré à l'étape 02 (samtools flagstat)
# un taux élevé confirme que les données sont dé-hôtées et exploitables pour l'assemblage
```

> **Lecture** — Si le `flagstat` de l'étape 02 montrait un taux d'alignement élevé sur la référence MPOX, les données sont bien virales : on peut assembler directement. Si le taux était très faible, l'assemblage risquerait d'être dominé par des séquences non virales.

> **⚠️ Important — pour vos propres données**
> Ce jeu public a été **dé-hôté avant dépôt** (taille réduite, source « VIRAL RNA »), on saute donc cette étape. **Mais vos propres échantillons cliniques ne le seront pas** : ils contiennent une majorité de lectures humaines. Avant d'assembler vos données, vous **devez** retirer les lectures de l'hôte, typiquement avec **Kraken2** (base humaine) :
> ```bash
> kraken2 --db kraken2_human_db --paired \
>   --unclassified-out reads_unclass#.fastq \
>   R1.clean.fastq.gz R2.clean.fastq.gz
> ```
> puis assembler les lectures « unclassified » (non humaines). Sans ce dé-hôtage, l'assemblage sera dominé par le génome humain.
>
> **Enrichissement ≠ dé-hôtage.** Une méthode d'enrichissement en laboratoire (panel de capture Twist, amplicon, etc.) augmente la proportion de lectures virales, mais laisse toujours une fraction de lectures humaines (capture non spécifique, ADN de fond). L'enrichissement **réduit** le besoin de dé-hôtage sans l'**éliminer** : un dé-hôtage bioinformatique reste recommandé, pour la qualité de l'assemblage comme pour la confidentialité des données du patient. Vérifiez toujours le taux de lectures humaines résiduelles.

---

## Étape 2 — Assemblage de novo (SPAdes)

SPAdes assemble directement les lectures nettoyées en contigs.

```bash
spades.py \
  --rnaviral \
  -1 ../01_qc/clean/SRR30229922_1.clean.fastq.gz \
  -2 ../01_qc/clean/SRR30229922_2.clean.fastq.gz \
  -t 4 \
  -o results/spades_SRR30229922
```

**Décorticage** :
- `-1` / `-2` — lectures paired-end nettoyées.
- `--rnaviral` — mode SPAdes adapté aux génomes viraux (recommandé pour ce type de données).
- `-t 4` — threads.
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

> **Lecture** — Un bon assemblage MPOX donnerait idéalement peu de contigs longs (le génome fait ~197 kb). Beaucoup de petits contigs = assemblage fragmenté (couverture faible, contamination).

---

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

**Option A — QUAST** (Linux / WSL2 uniquement). QUAST produit un rapport complet, mais n'est pas disponible sur macOS.

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

> **Lecture** — Ces métriques disent si l'assemblage est complet et contigu, ou fragmenté. Un assemblage shotgun à couverture modérée est souvent fragmenté (plusieurs contigs) tout en couvrant la quasi-totalité du génome.

---

## Récapitulatif

| Étape | Commande clé | Sortie |
|---|---|---|
| Vérifier | `samtools flagstat` (étape 02) | taux d'alignement viral |
| Assemblage | `spades.py --rnaviral` | `contigs.fasta` |
| Inspection | `grep -c ">"` | nombre de contigs |
| Situer | `blastn` vs référence | `.blastn.tsv` |
| Évaluer | `quast.py` (Linux/WSL2) ou commandes de base | N50, longueur, # contigs |

---

## Points de vigilance

- **Données déjà dé-hôtées** : pas de Kraken2 nécessaire ici ; on vérifie et on assemble directement.
- **De novo ≠ mapping** : complémentaires. Le de novo révèle ce que la référence ne contient pas.
- **N50 = contiguïté** : la métrique reine de l'assemblage.
- **Contigs multiples = fragmentation** : normal si couverture faible ; on situe avec BLASTn.
- **BLASTn valide l'identité** : confirme que les contigs sont bien viraux.
- **Évaluer objectivement** : ne pas juger un assemblage « à l'œil ». QUAST (Linux/WSL2) ou les commandes de base (N50, longueur, # contigs) donnent les métriques.