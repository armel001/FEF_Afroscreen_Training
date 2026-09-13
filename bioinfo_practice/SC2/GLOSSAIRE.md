# Glossaire — Génomique virale

**Formation FEF · Afroscreen — Bio-informatique**

Définitions des termes techniques rencontrés dans l'analyse SARS-CoV-2 (et transposables aux autres pathogènes viraux).

---

## Formats de fichiers

**FASTQ** (`.fastq`, `.fastq.gz`)
Format des lectures brutes de séquençage. Chaque lecture occupe 4 lignes : identifiant, séquence (ACGT), séparateur, scores de qualité (un caractère par base). Le `.gz` indique une compression gzip.

**FASTA** (`.fasta`, `.fa`)
Format d'une ou plusieurs séquences. Chaque séquence a une ligne d'en-tête commençant par `>`, suivie de la séquence. Utilisé pour la référence et le consensus.

**SAM / BAM** (`.sam`, `.bam`)
Résultat de l'alignement des lectures sur une référence. SAM est lisible (texte), BAM est sa version compressée (binaire). On travaille en BAM, trié et indexé.

**BAI** (`.bai`)
Index d'un fichier BAM. Permet un accès rapide à une région précise du génome sans lire tout le fichier.

**BED / BEDPE** (`.bed`, `.bedpe`)
Formats décrivant des régions du génome par coordonnées. BED = intervalles simples ; BEDPE = paires d'intervalles (utilisé pour décrire les paires d'amorces amplicon).

**bedGraph** (`.bedgraph`)
Format tabulé donnant la couverture (nombre de lectures) le long du génome, région par région.

**GFF3** (`.gff3`)
Fichier d'annotation : décrit la position des gènes et régions codantes sur le génome. Permet de traduire une mutation en changement d'acide aminé.

**VCF** (`.vcf`, `.vcf.gz`)
Format standard listant les variants (positions où l'échantillon diffère de la référence). Produit par certains variant callers.

**TSV** (`.tsv`)
Fichier tabulé (valeurs séparées par tabulations). Le tableau de variants produit par ivar est un TSV.

---

## Concepts de séquençage

**Lecture (read)**
Une séquence individuelle produite par le séquenceur, correspondant à un fragment d'ADN/ARN.

**Paired-end (appariées)**
Mode de séquençage où chaque fragment est lu des deux extrémités : lecture avant (R1, `_1`) et lecture arrière (R2, `_2`). Les deux fichiers vont par paires.

**Lectures courtes / longues**
Courtes (Illumina, ~100-300 bp) : précises mais fragmentées. Longues (Nanopore, jusqu'à des dizaines de kb) : franchissent les régions répétées, mais plus d'erreurs.

**Amplicon**
Fragment de génome amplifié par PCR ciblée avant séquençage. En amplicon, on ne séquence que les régions ciblées par les amorces (ex. schéma ARTIC pour SARS-CoV-2).

**Shotgun / métagénomique**
Séquençage de tout l'ADN/ARN présent dans l'échantillon, sans amplification ciblée. Comprend le pathogène mais aussi l'hôte et d'autres organismes.

**Adaptateur**
Courte séquence synthétique ajoutée aux fragments pendant la préparation de la librairie, nécessaire au séquençage. Doit être retirée avant l'analyse (Nextera, TruSeq… selon le kit).

**Amorce (primer)**
Courte séquence synthétique qui délimite un amplicon en PCR ciblée. Ne fait pas partie du génome viral réel → doit être retirée pour ne pas fausser l'analyse.

**Profondeur / couverture (depth / coverage)**
Nombre de lectures qui couvrent une position donnée du génome. Une profondeur élevée = plus de confiance dans la base appelée. « Couverture 1000× » = 1000 lectures en moyenne par position.

---

## Qualité

**Score Phred (Q)**
Mesure de la fiabilité d'une base. Q20 = 1 erreur sur 100 (99 % fiable) ; Q30 = 1 sur 1000 (99,9 %). Plus Q est élevé, plus la base est fiable.

**Encodage (sanger)**
Manière dont les scores de qualité sont codés dans le FASTQ. Les données Illumina récentes utilisent l'encodage « sanger » (Phred+33).

**Trimming (rognage)**
Suppression des portions de faible qualité (extrémités) ou indésirables (adaptateurs, amorces) des lectures.

**Read orphelin (single)**
En paired-end, lecture dont le partenaire a été éliminé au filtrage qualité. Sa paire est « cassée ».

---

## Alignement & analyse

**Référence (genome de référence)**
Génome connu servant de modèle pour comparer l'échantillon. Pour SARS-CoV-2 : Wuhan-Hu-1 (MN908947.3).

**Alignement / mapping**
Positionnement de chaque lecture sur la référence, pour déterminer d'où elle provient dans le génome.

**Indexation**
Préparation de la référence (bwa index) ou d'un BAM (samtools index) pour un accès rapide. Étape préalable obligatoire.

**Properly paired (correctement appariées)**
Se dit de paires R1/R2 qui s'alignent à la bonne distance et dans la bonne orientation — signe d'un bon alignement.

**Soft-clipping / hard-clipping**
Deux façons de retirer les amorces. Soft-clipping : les bases sont masquées (ignorées) mais conservées dans le fichier. Hard-clipping : les bases sont supprimées. Le soft-clipping est réversible et plus précis.

**Consensus**
Séquence finale de l'échantillon, reconstruite position par position à partir des lectures alignées. C'est le « génome » de l'échantillon.

**Variant / mutation**
Position où l'échantillon diffère de la référence : substitution (une base change), délétion (`-`, bases manquantes) ou insertion (`+`, bases ajoutées).

**N (position masquée)**
Base indéterminée dans le consensus, notée `N`. Signale une position où la couverture était insuffisante (sous le seuil, ex. < 10×) pour décider de la base.

**Fréquence allélique (ALT_FREQ)**
Proportion des lectures portant le variant à une position. Proche de 1 = variant majoritaire (fixé) ; 0,2-0,5 = variant minoritaire (population mixte, ou à confirmer).

**Mutation synonyme / non-synonyme**
Synonyme : le changement de base ne modifie pas l'acide aminé (protéine inchangée). Non-synonyme : l'acide aminé change (REF_AA ≠ ALT_AA).

**Acide aminé (AA)**
Unité de base des protéines. Une mutation non-synonyme change l'acide aminé à une position donnée de la protéine (ex. N501Y = asparagine → tyrosine en position 501).

---

## Outils

**conda / mamba**
Gestionnaires d'environnements et de logiciels scientifiques. mamba est une version plus rapide de conda. Permettent d'installer les outils et leurs dépendances de façon reproductible.

**FastQC**
Diagnostic qualité d'un fichier de lectures : produit un rapport visuel (qualité par base, adaptateurs, longueurs…).

**MultiQC**
Agrège plusieurs rapports (FastQC, cutadapt…) en une seule page de synthèse.

**Cutadapt**
Retire les séquences indésirables des lectures : adaptateurs, amorces, bases en excès.

**Sickle**
Rogne les extrémités de faible qualité des lectures par fenêtre glissante.

**BWA**
Aligneur de lectures courtes sur une référence (`bwa mem`).

**minimap2**
Aligneur polyvalent, adapté aux lectures longues (mode `map-ont` pour Nanopore).

**SAMtools**
Boîte à outils pour manipuler les alignements : tri, index, statistiques (flagstat), profondeur (depth), empilement (mpileup).

**BCFtools**
Outils pour l'appel de variants et la manipulation des fichiers VCF.

**Bedtools**
Outils pour manipuler les intervalles génomiques ; calcule la couverture (genomecov).

**Bamclipper**
Retire les amorces d'un BAM par soft-clipping, à partir d'un fichier BEDPE (méthode alternative).

**ivar**
Outil dédié à l'analyse d'amplicons viraux : retrait d'amorces, appel de variants, génération de consensus.

---

## Paramètres fréquents

| Paramètre | Sens |
|---|---|
| `-q 20` / `-q 30` | seuil de qualité minimale (Phred) |
| `-l 25` | longueur minimale d'une lecture après rognage |
| `-m 10` | profondeur (couverture) minimale pour appeler une base ou un variant |
| `-t 0.2` | fréquence minimale pour rapporter un variant (20 %) |
| `-t 0.6` | fréquence minimale pour fixer une base au consensus (60 %) |
| `-t sanger` | type d'encodage de qualité |

---

## Abréviations

| Sigle | Signification |
|---|---|
| NGS | Next-Generation Sequencing (séquençage à haut débit) |
| QC | Quality Control (contrôle qualité) |
| PCR | Polymerase Chain Reaction (amplification) |
| RT-PCR | Reverse Transcription PCR (pour les virus à ARN) |
| ARTIC | consortium fournissant les schémas d'amorces amplicon viraux |
| RBD | Receptor Binding Domain (domaine de liaison au récepteur, dans le Spike) |
| NTD | N-Terminal Domain (domaine N-terminal du Spike) |
| UTR | Untranslated Region (région non traduite du génome) |
| SRA / ENA | bases de données publiques de lectures de séquençage |
| bp | base pair (paire de bases), unité de longueur |
