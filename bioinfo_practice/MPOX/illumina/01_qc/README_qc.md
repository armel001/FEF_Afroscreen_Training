# MPOX · Étape 01 — Contrôle qualité & nettoyage (Illumina)

Données : MPXV clade Ib (Illumina paired-end, métagénomique shotgun)
Sortie : lectures nettoyées → entrée de l'étape `02_mapping`

---

## Objectif

Évaluer la qualité des lectures brutes, les nettoyer et vérifier l'amélioration. En shotgun, **pas d'amorces à retirer** : le nettoyage se fait en une passe avec fastp.

```
fastq bruts
   → FastQC (diagnostic)
   → fastp (adaptateurs + qualité)
   → FastQC (contrôle)
   → MultiQC (synthèse)
```

---

## Prérequis

```bash
conda activate mpox_analyse
fastqc --version
fastp --version
multiqc --version
```

```bash
cd ~/bioinfo_practice/MPOX/illumina/01_qc
mkdir -p fastqc_avant clean fastqc_apres
```

---

## Étape 0 — Récupérer les données brutes

L'échantillon Illumina, placé dans `../../data/`.

```bash
cd ~/bioinfo_practice/MPOX/data
fasterq-dump --split-files SRR30229922
gzip SRR30229922_1.fastq SRR30229922_2.fastq
ls -lh
```

---

## Étape 1 — Diagnostic (FastQC)

```bash
cd ~/bioinfo_practice/MPOX/illumina/01_qc
fastqc ../../data/SRR30229922_1.fastq.gz ../../data/SRR30229922_2.fastq.gz -o fastqc_avant/
```

> **Lecture shotgun** — Contrairement à l'amplicon, on ne s'attend pas à une forte duplication. En revanche, le contenu des séquences peut être hétérogène (plusieurs organismes présents : virus, hôte humain…). C'est normal.

> **Biais de composition en début de read** — En shotgun / RNA-Seq, les premières bases (~10-12) montrent souvent une composition instable (FastQC le signale en rouge). C'est la signature de l'**amorçage aléatoire** (random priming) utilisé en préparation de librairie : les hexamères d'amorçage ne sont pas parfaitement aléatoires. **On ne coupe pas ces bases** — contrairement aux amorces d'un protocole amplicon, ce biais est bénin et fait partie des données.

---

## Étape 2 — Nettoyage (fastp)

fastp fait tout en une passe : détection et retrait des adaptateurs, rognage qualité, filtrage des lectures courtes.

```bash
fastp \
  -i ../../data/SRR30229922_1.fastq.gz \
  -I ../../data/SRR30229922_2.fastq.gz \
  -o clean/SRR30229922_1.clean.fastq.gz \
  -O clean/SRR30229922_2.clean.fastq.gz \
  --detect_adapter_for_pe \
  --cut_front \
  --cut_tail \
  --cut_mean_quality 20 \
  --qualified_quality_phred 20 \
  --length_required 30 \
  --json clean/SRR30229922.fastp.json \
  --html clean/SRR30229922.fastp.html
```

**Décorticage** :
- `-i` / `-I` / `-o` / `-O` — entrées et sorties R1 / R2.
- `--detect_adapter_for_pe` — détecte et retire automatiquement les adaptateurs (paired-end).
- `--cut_front` / `--cut_tail` — rogne les bases de faible qualité en début et fin de lecture.
- `--cut_mean_quality 20` — seuil de qualité pour le rognage.
- `--qualified_quality_phred 20` — base « bonne » si Q ≥ 20.
- `--length_required 30` — jette les lectures < 30 bases après rognage (seuil bas car les inserts de ce jeu sont courts, ~39 bp).
- `--json` / `--html` — rapports (le JSON est lu par MultiQC).

> **Détection auto des adaptateurs** — En shotgun, on ne connaît pas toujours le kit. `--detect_adapter_for_pe` laisse fastp identifier l'adaptateur tout seul, ce qui est robuste.

> **Adapter le seuil de longueur à l'insert** — fastp rapporte l'« insert size peak » (taille du fragment ADN réel). Si les inserts sont courts (ex. ~39 bp), beaucoup de lectures deviennent courtes après rognage. Un seuil `--length_required` trop haut les jetterait et ferait perdre du signal viral. On abaisse alors le seuil (ici 30) pour conserver ces lectures. À vérifier dans le rapport fastp.

---

## Étape 3 — Contrôle après nettoyage

```bash
fastqc clean/SRR30229922_1.clean.fastq.gz clean/SRR30229922_2.clean.fastq.gz -o fastqc_apres/
```

> **✅ Attendu** — Qualité stabilisée, adaptateurs retirés.

---

## Étape 4 — Synthèse (MultiQC)

```bash
cd ~/bioinfo_practice/MPOX/illumina/01_qc
multiqc . -d -dd 1 -o multiqc_report/
```

**Décorticage** :
- `-d -dd 1` — préfixe chaque échantillon par son dossier (`fastqc_avant` / `fastqc_apres`) pour distinguer avant/après. Sans ça, MultiQC confond les rapports de même nom.

---

## Récapitulatif

| Étape | Outil | Sortie |
|---|---|---|
| Diagnostic | FastQC | rapports HTML |
| Nettoyage | fastp | `.clean` |
| Contrôle | FastQC | rapports HTML |
| Synthèse | MultiQC | rapport agrégé |

**Sortie clé** : `clean/SRR30229922_{1,2}.clean.fastq.gz` → entrée de `02_mapping`.

---

## Points de vigilance

- **fastp fait tout** : adaptateurs + qualité en une commande (pas d'étape d'amorces en shotgun).
- **Détection auto des adaptateurs** : robuste quand le kit est inconnu.
- **Contenu métagénomique** : les lectures ne sont pas toutes virales ; le tri se fait au mapping.
- **Ne pas sur-nettoyer** : en shotgun, les lectures virales sont déjà minoritaires ; garder de la marge.