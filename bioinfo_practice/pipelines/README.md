# Session pipelines

**Formation FEF · Afroscreen — Bio-informatique**

Objectif : constater que les pipelines reproduisent — et enchaînent automatiquement — les étapes qu'on a exécutées une par une.

Deux pipelines, un par cas d'étude :

| Cas | Pipeline | Moteur | Documentation |
|---|---|---|---|
| **SARS-CoV-2** (amplicon) | **GeVarLi** | Snakemake | <https://transvihmi.pages.ird.fr/nfernandez/GeVarLi/en> |
| **MPOX** (shotgun) | **nf-core/viralrecon** | Nextflow | <https://nf-co.re/viralrecon> |

> **A savoir** — L'analyse manuelle et les pipelines ne s'opposent pas : comprendre les gestes à la main permet de **savoir ce que le pipeline fait**, de l'utiliser en confiance, et de diagnostiquer quand il échoue.

---

# Pipeline 1 — GeVarLi (SARS-CoV-2)

**GeVarLi** = **Ge**nome assembly, **Var**iant calling, **Li**neage assignation.
Pipeline Snakemake du projet AFROSCREEN (TransVIHMI / IRD), pour lectures courtes Illumina.

- Dépôt : <https://forge.ird.fr/transvihmi/nfernandez/GeVarLi>
- Documentation : <https://transvihmi.pages.ird.fr/nfernandez/GeVarLi/en>

## 1.1 — Installation

L'installation complète est décrite dans la documentation officielle :
**<https://transvihmi.pages.ird.fr/nfernandez/GeVarLi/en/pages/installations/>**

En résumé (se référer à la doc pour les détails et le dépannage) :

```bash
# 1. Récupérer le pipeline
git clone --depth 1 https://forge.ird.fr/transvihmi/nfernandez/GeVarLi.git ~/GeVarLi/
cd ~/GeVarLi/

# 2. Rendre le script de lancement exécutable
chmod +x Run_GeVarLi.sh
```

GeVarLi crée lui-même ses environnements Conda au premier lancement (via Snakemake). Prérequis : **Conda** (Miniforge) installé et fonctionnel.

> **Premier lancement long** — La création des environnements Conda prend du temps (téléchargements), une seule fois. Les lancements suivants sont rapides.

## 1.2 — Placer nos données

On réutilise **notre échantillon SC2** analysé à la main. GeVarLi attend les fichiers dans `resources/reads/`, nommés `{SAMPLE}_R1.fastq.gz` / `{SAMPLE}_R2.fastq.gz`.

```bash
# copier et renommer (_1/_2 → _R1/_R2)
cp ~/bioinfo_practice/SC2/data/SRR17051908_1.fastq.gz ~/GeVarLi/resources/reads/SRR17051908_R1.fastq.gz
cp ~/bioinfo_practice/SC2/data/SRR17051908_2.fastq.gz ~/GeVarLi/resources/reads/SRR17051908_R2.fastq.gz

# amorces ARTIC (format BEDPE) pour le retrait par Bamclipper
cp ~/bioinfo_practice/SC2/data/primers.bedpe ~/GeVarLi/resources/primers/bedpe/
```

## 1.3 — Configurer

Les paramètres se règlent dans `configuration/config.yaml` :

```bash
nano ~/GeVarLi/configuration/config.yaml
```

Points à vérifier (valeurs par défaut = celles de notre analyse manuelle) :
- **référence** : SARS-CoV-2 Wuhan MN908947.3 (incluse) ;
- **aligneur** : bwa / minimap2 / bowtie2 ;
- **amorces** : protocole amplicon + fichier BEDPE ;
- **seuils** : qualité Q30, couverture min 10, fréquence variant 0.2.

## 1.4 — Exécuter

Tout se lance avec un seul script :

```bash
cd ~/GeVarLi/
./Run_GeVarLi.sh
```

Le pipeline enchaîne : QC → nettoyage → retrait amorces → alignement → couverture → variants → consensus → **lignée (Nextclade / Pangolin)**.

## 1.5 — Résultats

Les sorties sont rangées dans le dossier de résultats (voir la doc : <https://transvihmi.pages.ird.fr/nfernandez/GeVarLi/en/pages/results/>) : rapports QC, consensus, tableau de variants, et **assignation de lignée** — l'étape qu'on n'avait pas faite à la main.

---
---

# Pipeline 2 — nf-core/viralrecon 

Pipeline **Nextflow** de référence internationale (communauté nf-core), pour reconstruction de génomes viraux. Gère Illumina et Nanopore, amplicon et métagénomique.

- Site : <https://nf-co.re/viralrecon>
- Documentation d'usage : <https://nf-co.re/viralrecon/docs/usage>

## 2.1 — Installation

viralrecon a besoin de **Nextflow** et d'un gestionnaire d'outils (**Conda**, ou Docker/Singularity). La façon la plus simple : installer Nextflow **via conda**, dans son propre environnement. Cela installe aussi les outils `nf-core`.

```bash
# créer un environnement dédié avec Nextflow + nf-core
mamba create -n nf-core -c bioconda -c conda-forge nextflow nf-core
conda activate nf-core

# vérifier
nextflow -version
```

Documentation officielle : <https://nf-co.re/docs/usage/getting_started/installation>

> **Le pipeline se télécharge tout seul** — pas besoin de le cloner. `nextflow run nf-core/viralrecon` récupère automatiquement le pipeline au premier lancement.

> **Le profil (`-profile`) choisit comment les outils sont fournis** :
> - `conda` — installe les outils via conda (pas de logiciel supplémentaire à installer ; le plus simple ici).
> - `docker` / `singularity` — conteneurs (recommandé par nf-core pour une reproductibilité maximale, mais nécessite d'installer Docker ou Singularity).
> Dans cette formation, on utilise `-profile conda`.

### Tester l'installation avant d'utiliser ses données

nf-core fournit un jeu de test minimal. Toujours le lancer d'abord pour vérifier que tout fonctionne :

```bash
nextflow run nf-core/viralrecon -profile test,conda --outdir test_results
```

Si le test se termine sans erreur, l'installation est bonne.

## 2.2 — Ressources & mémoire

Les pipelines Nextflow lancent **plusieurs étapes en parallèle**, chacune demandant sa part de mémoire (RAM) et de cœurs. C'est puissant, mais exigeant pour la machine.

**Ce qu'il faut savoir :**
- Certaines étapes (assemblage SPAdes, alignement, Kraken2 si activé) sont **gourmandes en RAM**. Sur une machine modeste, elles peuvent échouer faute de mémoire.
- viralrecon définit des besoins par défaut pensés pour des serveurs. Sur un ordinateur portable, il faut souvent **plafonner** les ressources pour éviter que Nextflow ne demande plus que disponible.

**Plafonner les ressources** (adapter à votre machine) — deux options :

Directement dans la commande :
```bash
nextflow run nf-core/viralrecon ... \
  --max_memory '6.GB' \
  --max_cpus 4
```

Ou via un fichier de configuration `custom.config` passé avec `-c` :
```
process {
  resourceLimits = [ cpus: 4, memory: 6.GB ]
}
```

**En cas d'échec « out of memory » ou processus tué :**
- Baisser `--max_memory` et `--max_cpus` pour rester sous la limite réelle de la machine.
- Sauter les étapes lourdes non essentielles : `--skip_assembly` (ne fait que le mapping), `--skip_kraken2` (pas de dé-hôtage).
- Relancer avec `-resume` : Nextflow reprend là où ça s'est arrêté, sans tout refaire.

---

## 2.3 — Préparer la feuille d'échantillons (samplesheet)

viralrecon lit un fichier **`samplesheet.csv`** qui liste les échantillons. Pour notre MPOX Illumina (paired-end) :

```bash
mkdir -p ~/viralrecon_mpox && cd ~/viralrecon_mpox

cat > samplesheet.csv << 'CSV'
sample,fastq_1,fastq_2
MPOX_KEN,/home/USER/bioinfo_practice/MPOX/data/SRR30229922_1.fastq.gz,/home/USER/bioinfo_practice/MPOX/data/SRR30229922_2.fastq.gz
CSV
```

> Remplacer `/home/USER/` par votre chemin réel (`echo $HOME` pour le connaître). Les chemins doivent être **absolus**.

> **Astuce** — Un script `fastq_dir_to_samplesheet.py` fourni par nf-core génère automatiquement le samplesheet à partir d'un dossier de fastq (voir la doc usage).

## 2.4 — Exécuter (Illumina, métagénomique)

Notre MPOX est du **shotgun métagénomique**. La commande :

```bash
cd ~/viralrecon_mpox

nextflow run nf-core/viralrecon \
  --input samplesheet.csv \
  --outdir results_mpox \
  --platform illumina \
  --protocol metagenomic \
  --fasta /home/USER/bioinfo_practice/MPOX/data/reference.fasta \
  --skip_kraken2 \
  -profile conda
```

**Décorticage des options** :
- `--input` — la feuille d'échantillons (samplesheet.csv).
- `--outdir` — dossier de sortie des résultats.
- `--platform illumina` — données Illumina (lectures courtes).
- `--protocol metagenomic` — shotgun (pas amplicon) → variant calling par BCFTools.
- `--fasta` — notre référence MPOX (NC_003310.1). *(Pour SC2, on utiliserait plutôt `--genome 'MN908947.3'` qui est intégré.)*
- `--skip_kraken2` — saute le dé-hôtage (nos données sont déjà filtrées ; à retirer pour vos propres échantillons cliniques).
- `-profile conda` — installe les outils via conda. Alternatives : `docker` ou `singularity` si installés.

> **Options utiles** (voir la doc pour la liste complète) :
> - `--skip_assembly` — ne fait que le mapping (pas le de novo), plus rapide.
> - `--kraken2_db <chemin>` — fournir une base Kraken2 pour le dé-hôtage.
> - `-resume` — reprend un run interrompu sans tout refaire.

## 2.5 — Exécuter (Nanopore) — pour aller plus loin

viralrecon en mode Nanopore est optimisé pour l'**amplicon ARTIC**. Nos données MPOX Nanopore étant du **shotgun**, ce mode ne s'applique pas directement. À titre indicatif, la forme d'une commande Nanopore amplicon :

```bash
nextflow run nf-core/viralrecon \
  --input samplesheet.csv \
  --outdir results \
  --platform nanopore \
  --genome 'MN908947.3' \
  --primer_set artic \
  --primer_set_version 3 \
  --fastq_dir fastq_pass/ \
  -profile conda
```

## 2.6 — Résultats

viralrecon produit un rapport **MultiQC** global, le consensus, le tableau de variants, et l'assignation de lignée (Pangolin). Tout est rangé dans `--outdir`.


# Points de vigilance

- **Nommage des fichiers** : GeVarLi attend `_R1`/`_R2` ; viralrecon lit un samplesheet avec chemins **absolus**.
- **Premier lancement long** : création des environnements / téléchargement des conteneurs (une fois).
- **Profil viralrecon** : `-profile conda` dans cette formation (ou docker/singularity si installés).
- **Dé-hôtage** : `--skip_kraken2` seulement parce que nos données sont déjà filtrées ; à activer pour des échantillons cliniques.
- **`-resume`** (viralrecon) : reprend là où ça s'est arrêté, très utile.
- **Comprendre avant d'automatiser** : le pipeline n'est pas une boîte noire quand on a fait les gestes à la main.