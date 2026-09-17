# SC2 · Étape 03 — Appel de variants (ivar)

Données : SARS-CoV-2 (Illumina paired-end, amplicon ARTIC)
Entrée : BAM aligné de l'étape `02_mapping`

---

## Objectif

Identifier les positions où l'échantillon diffère de la référence (les variants), et lire le tableau de résultats. On fait l'appel **deux fois** — sans puis avec annotation (GFF3) — pour comprendre ce que le GFF3 apporte.

```
BAM aligné → samtools mpileup → ivar variants → tableau .tsv
```

---

## Consensus vs variants

Même BAM, deux lectures complémentaires :

| | Consensus (étape 02) | Variants (ici) |
|---|---|---|
| Question | « Quelle séquence entière ? » | « Où et comment diffère-t-elle ? » |
| Format | FASTA | Tableau (1 ligne / mutation) |

---

## Prérequis

```bash
conda activate sc2_analyse
cd ~/bioinfo_practice/SC2/03_variants
mkdir -p results
```

Fichiers nécessaires :
- BAM aligné : `../02_mapping/results/SRR17051908.sorted.bam`
- référence : `../data/reference.fasta`
- annotation (optionnelle) : `../data/reference.gff3`

---

## Préparer l'annotation GFF3 (pour la version « avec effet »)

Le GFF3 doit porter le **même nom de chromosome** que la référence, sinon ivar ne fait pas le lien.

```bash
cd ~/bioinfo_practice/SC2/data

# nom dans la référence
head -1 reference.fasta

# nom dans le GFF3
grep -v "^#" MN908947.gff3 | head -1 | cut -f1
```

Si les noms diffèrent (ex. `MN908947.3` dans la référence, `SARS-CoV-2_Wuhan_MN-908947.3` dans le GFF3), on harmonise le GFF3 sur le nom de la référence :

```bash
sed 's/SARS-CoV-2_Wuhan_MN-908947.3/MN908947.3/g' MN908947.gff3 > reference.gff3
```

> **Piège classique** — Un même génome peut porter plusieurs noms/accessions (`MN908947.3`, `NC_045512.2`…). Si la référence et le GFF3 ne s'accordent pas exactement, l'annotation échoue silencieusement. Toujours vérifier avant.

---

## Étape 1a — Appel SANS annotation

Le principe brut : positions, bases, fréquences.

```bash
cd ~/bioinfo_practice/SC2/03_variants

samtools mpileup -aa -A -d 0 -B -Q 0 \
  --reference ../data/reference.fasta \
  ../02_mapping/results/SRR17051908.sorted.bam \
  | ivar variants \
      -p results/SRR17051908.variants_sans_gff \
      -r ../data/reference.fasta \
      -q 20 \
      -t 0.2 \
      -m 10
```

ivar prévient : *« A GFF file... has not been provided. Amino acid translation will not be done »* — les colonnes d'effet protéique resteront vides.

---

## Étape 1b — Appel AVEC annotation

Même commande, plus `-g` (le GFF3 harmonisé).

```bash
samtools mpileup -aa -A -d 0 -B -Q 0 \
  --reference ../data/reference.fasta \
  ../02_mapping/results/SRR17051908.sorted.bam \
  | ivar variants \
      -p results/SRR17051908.variants_avec_gff \
      -r ../data/reference.fasta \
      -g ../data/reference.gff3 \
      -q 20 \
      -t 0.2 \
      -m 10
```

**Décorticage** :
- `-t 0.2` — fréquence minimale 20 % pour rapporter un variant.
- `-m 10` — couverture minimale 10.
- `-q 20` — qualité minimale des bases.
- `-g` — annotation GFF3 (remplit l'effet protéique).

---

## Étape 2 — Comparer les deux sorties

```bash
wc -l results/SRR17051908.variants_sans_gff.tsv results/SRR17051908.variants_avec_gff.tsv
```

> **Point clé** — Les deux fichiers ont le **même nombre de variants**. Le GFF3 n'ajoute aucun variant : il **remplit** les colonnes d'effet (gène, acide aminé) qui étaient vides. Le nombre de mutations détectées ne dépend pas de l'annotation.

Comparaison du contenu :

```bash
# sans GFF : colonnes d'effet à NA
grep -v "^REGION" results/SRR17051908.variants_sans_gff.tsv | head -3 | cut -f1,2,3,4,11,15,17,18

# avec GFF : colonnes d'effet remplies (si région codante)
grep -v "^REGION" results/SRR17051908.variants_avec_gff.tsv | head -3 | cut -f1,2,3,4,11,15,17,18
```

> **Nuance** — Une mutation dans une région **non traduite (UTR)** reste `NA` même avec le GFF3 : il n'y a pas d'acide aminé à annoter. Le GFF3 n'annote que les régions codantes.

---

## Étape 3 — Lire les colonnes

| Colonne | Signification |
|---|---|
| `POS` | Position sur le génome |
| `REF` / `ALT` | Base de référence / variant (`-` = délétion, `+` = insertion) |
| `ALT_FREQ` | Fréquence du variant (0 à 1) |
| `TOTAL_DP` | Profondeur totale à cette position |
| `PASS` | TRUE si le variant passe le test statistique |
| `GFF_FEATURE` | Gène touché (si GFF3 et région codante) |
| `REF_AA` / `ALT_AA` | Acide aminé de référence / muté |
| `POS_AA` | Position de l'acide aminé dans la protéine |

### Trois questions par variant

1. **Fiable ?** → `PASS` = TRUE, `TOTAL_DP` élevé.
2. **Majoritaire ou minoritaire ?** → `ALT_FREQ` proche de 1 = fixé ; 0.2-0.5 = minoritaire.
3. **Change la protéine ?** → `REF_AA` ≠ `ALT_AA` = mutation non-synonyme.

---

## Étape 4 — Explorer les variants du gène Spike

Le gène S (Spike) s'étend des positions ~21563 à 25384. On isole ses variants pour les examiner :

```bash
awk -F'\t' 'NR==1 || ($2>=21563 && $2<=25384)' results/SRR17051908.variants_avec_gff.tsv \
  | cut -f1,2,3,4,11,15,17,18,20 \
  | column -t
```

Pour nommer une mutation d'acide aminé, on combine `REF_AA` + `POS_AA` + `ALT_AA` (ex. une lysine en position 501 devenue asparagine se note K501N).

> **Indels non annotés** — Les délétions (`ALT` commençant par `-`) et insertions (`+`) ont `GFF_FEATURE = NA` même avec le GFF3 : ivar n'annote l'effet protéique que des substitutions.

---

## Lien avec la couverture (étape 02)

Le rapport de couverture de l'étape 02 (bedtools) et le tableau de variants se lisent ensemble : une zone à faible couverture ne peut pas produire de variant fiable (pas assez de lectures pour décider). Croiser les deux permet de distinguer « pas de mutation ici » de « zone où l'on est aveugle ».

En amplicon, une accumulation de mutations peut faire échouer une amorce (mismatch amorce/template) et créer un trou de couverture localisé — un point à garder en tête quand une région attendue comme variable ressort vide.

---

## Récapitulatif

| Étape | Commande clé | Sortie |
|---|---|---|
| Harmoniser GFF3 | `sed ... > reference.gff3` | GFF3 aligné sur la référence |
| Appel sans GFF | `ivar variants` (sans `-g`) | `.variants_sans_gff.tsv` |
| Appel avec GFF | `ivar variants -g` | `.variants_avec_gff.tsv` |
| Comparer | `wc -l`, `cut` | même nb de variants, effet en plus |
| Explorer Spike | `awk` sur 21563-25384 | variants du gène S |

---

## Points de vigilance

- **Le GFF3 n'ajoute pas de variants** : il annote l'effet des substitutions codantes.
- **Noms de chromosome identiques** entre référence et GFF3, sinon échec silencieux.
- **UTR et indels non annotés** : normal, pas une erreur.
- **Profondeur = confiance** : un variant à faible profondeur est peu fiable.