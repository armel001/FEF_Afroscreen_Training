# SC2 · Étape 01 — Contrôle qualité & nettoyage

Données : SARS-CoV-2 (Illumina paired-end, amplicon ARTIC)
Sortie : lectures nettoyées → entrée de l'étape `02_mapping`

---

## Objectif

Évaluer la qualité des lectures brutes, les nettoyer et vérifier l'amélioration.

```
fastq bruts
   → FastQC
   → Cutadapt (adaptateurs Illumina)
   → Cutadapt (amorces amplicon, hard-clipping)
   → Sickle-trim (nettoyage qualité, Q30)
   → FastQC (contrôle)
   → MultiQC (synthèse)
```
---

## Prérequis

```bash
conda activate sc2_analyse
fastqc --version
cutadapt --version
sickle --version
multiqc --version
```

```bash
cd ~/pratique/SC2/01_qc
mkdir -p fastqc_avant clean fastqc_apres
```

---

## Étape 0 — Récupérer les données brutes

Un seul échantillon, placé dans `../data/` .

```bash
cd ~/pratique/SC2/data
fasterq-dump --split-files SRR17051908
gzip SRR17051908_1.fastq SRR17051908_2.fastq
ls -lh
```
ref : https://www.ncbi.nlm.nih.gov/sra/?term=SRR17051908  
---

## Étape 1 — Diagnostic (FastQC)

```bash
cd ~/pratique/SC2/01_qc
fastqc ../data/SRR17051908_1.fastq.gz ../data/SRR17051908_2.fastq.gz -o fastqc_avant/
```

> **Lecture amplicon** — Duplication et séquences surreprésentées en rouge sont NORMALES (PCR ciblée). On juge sur *Per base sequence quality* et *Adapter Content*.

---

## Étape 2 — Retrait des adaptateurs Illumina (Cutadapt)

Cutadapt retire les séquences d'adaptateurs en 3' des lectures. Ces données ont été préparées avec un kit **Nextera** (visible dans les métadonnées SRA), donc on utilise l'adaptateur Nextera.

```bash
cutadapt \
  -a CTGTCTCTTATACACATCT \
  -A CTGTCTCTTATACACATCT \
  -o clean/SRR17051908_1.noadapt.fastq.gz \
  -p clean/SRR17051908_2.noadapt.fastq.gz \
  ../data/SRR17051908_1.fastq.gz \
  ../data/SRR17051908_2.fastq.gz
```
Ref : https://support-docs.illumina.com/SHARE/AdapterSequences/Content/Nextera_Illumina-Sequences.htm

**Décorticage** :
- `-a` / `-A` — séquence d'adaptateur à retirer en 3' de R1 / R2. `CTGTCTCTTATACACATCT` est l'adaptateur Nextera (Transposase). Pour un kit TruSeq, on utiliserait `AGATCGGAAGAGC`.
- `-o` / `-p` — fichiers de sortie R1 / R2.

> **Vérifier le kit** — Le type d'adaptateur dépend de la préparation de librairie (Nextera vs TruSeq). L'information se trouve dans les métadonnées du séquençage (page SRA, champ « Design / Library »).

---

## Étape 3 — Retrait des amorces amplicon (Cutadapt, hard-clipping)

**Attention.** Si vous ne connaissez pas les coordonnées exactes, on retire de force les N premiers nucléotides (N = longueur des amorces ARTIC, ~22-30 nt selon le schéma).

```bash
cutadapt \
  -u 30 -U 30 \
  -o clean/SRR17051908_1.noprimer.fastq.gz \
  -p clean/SRR17051908_2.noprimer.fastq.gz \
  clean/SRR17051908_1.noadapt.fastq.gz \
  clean/SRR17051908_2.noadapt.fastq.gz
```

**Décorticage** :
- `-u 30` — coupe 30 nt au début de R1. `-U 30` — idem pour R2.
- La valeur 30 = longueur approximative des amorces ARTIC.

---

## Étape 4 — Nettoyage qualité (Sickle, Q30)

Sickle rogne les extrémités de faible qualité par fenêtre glissante

```bash
sickle pe \
  -f clean/SRR17051908_1.noprimer.fastq.gz \
  -r clean/SRR17051908_2.noprimer.fastq.gz \
  -t sanger \
  -q 30 \
  -l 25 \
  -g \
  -o clean/SRR17051908_1.clean.fastq.gz \
  -p clean/SRR17051908_2.clean.fastq.gz \
  -s clean/SRR17051908.singles.fastq.gz
```

**Décorticage** :
- `pe` — mode paired-end.
- `-f` / `-r` — R1 / R2 en entrée.
- `-t sanger` — encodage qualité (Illumina récent = sanger).
- `-q 30` — **seuil de qualité Q30**
- `-l 25` — **longueur minimale 25** 
- `-g` — sortie compressée (gzip).
- `-o` / `-p` — sorties nettoyées R1 / R2.
- `-s` — lectures « orphelines »

---

## Étape 5 — Contrôle après nettoyage

```bash
fastqc clean/SRR17051908_1.clean.fastq.gz clean/SRR17051908_2.clean.fastq.gz -o fastqc_apres/
```

---

## Étape 6 — Synthèse (MultiQC)
 
```bash
cd ~/pratique/SC2/01_qc
multiqc . -d -dd 1 -o multiqc_report/
```
 
**Décorticage** :
- `-d` — préfixe chaque échantillon par le nom de son dossier (`fastqc_avant` / `fastqc_apres`).
- `-dd 1` — ne garde qu'un seul niveau de dossier comme préfixe.
> **Pourquoi `-d -dd 1` ?** Sans ces options, MultiQC identifie les échantillons par leur nom de fichier seul. Comme les rapports avant et après portent le même identifiant (`SRR17051908_1`), MultiQC les confond et n'en garde qu'un. Avec `-d -dd 1`, on obtient bien **4 entrées distinctes** (avant/après × R1/R2), comparables sur une seule page.
 
MultiQC agrège FastQC (avant/après) en distinguant les deux étapes.
 
---
 
## Récapitulatif
 
| Étape | Outil | Sortie |
|---|---|---|
| Diagnostic | FastQC | rapports HTML |
| Adaptateurs | Cutadapt (`-a`/`-A`) | `.noadapt` |
| Amorces (hard) | Cutadapt (`-u`/`-U`) | `.noprimer` |
| Qualité | Sickle (`-q 30 -l 25`) | `.clean` |
| Contrôle | FastQC | rapports HTML |
| Synthèse | MultiQC | rapport agrégé |
 
**Sortie clé** : `clean/SRR17051908_{1,2}.clean.fastq.gz` → entrée de `02_mapping`.
 