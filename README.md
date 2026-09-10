# MSc NGS Practical: Germline Variant Calling with GATK HaplotypeCaller

This practical runs entirely in **GitHub Codespaces** using the preconfigured `gvar` environment.

**Dataset:** HCC1954 Normal paired-end reads  
**Reference:** GRCh38 chromosome 17 subset  
**Pipeline:** FASTQ → QC → Alignment → Read Groups → Mark Duplicates → BQSR → HaplotypeCaller → Genotyping → Hard Filtering → Database Annotation

---

## Before starting

Open your Codespace and activate the preinstalled environment:

```bash
mamba activate gvar
```

Check available CPUs:

```bash
nproc
```

> The course Codespace currently provides 2 CPUs, ~8 GB RAM, and a 32 GB workspace.
> Do **not** run `mamba create` or reinstall the tools; they are already installed in `gvar`.

---

## Terminal 2 — Start the live file browser

Keep the NGS pipeline running in **Terminal 1**.

Open **Terminal → New Terminal** and run:

```bash
python3 -m http.server 8000 --directory ~/variantcalling
```

Leave this terminal running. Open the **PORTS** tab in Codespaces, find port `8000`, choose **Open in Browser**, and keep visibility **Private**.

Refresh the browser whenever you want to see newly generated files.

Stop the server later with:

```text
Ctrl + C
```

---

## STEP 1 — Create directories

Run in **Terminal 1**:

```bash
mkdir -p ~/variantcalling/reference/hg38
mkdir -p ~/variantcalling/reference/BQSR
mkdir -p ~/variantcalling/reference/annotation

mkdir -p ~/variantcalling/gvar/input/HCC1954

mkdir -p ~/variantcalling/gvar/output/fastqc/pre-QC
mkdir -p ~/variantcalling/gvar/output/fastqc/post-QC
mkdir -p ~/variantcalling/gvar/output/trimming
mkdir -p ~/variantcalling/gvar/output/alignment
mkdir -p ~/variantcalling/gvar/output/markduplicates
mkdir -p ~/variantcalling/gvar/output/BQSR
mkdir -p ~/variantcalling/gvar/output/variants
mkdir -p ~/variantcalling/gvar/output/annotation
```

Check:

```bash
find ~/variantcalling -maxdepth 4 -type d | sort
```

---

## STEP 2 — Environment

The environment is already installed by the Codespaces template.

```bash
mamba activate gvar
```

Optional verification:

```bash
gatk --version
samtools --version | head -2
bcftools --version | head -2
fastqc --version
```

---

## STEP 3 — Download input files

### FASTQ files

```bash
wget -c -P ~/variantcalling/gvar/input/HCC1954 https://gcu-msc.s3.amazonaws.com/somatic-subset/HCC1954_Normal_R1.fastq.gz https://gcu-msc.s3.amazonaws.com/somatic-subset/HCC1954_Normal_R2.fastq.gz
```

### GRCh38 chromosome 17 reference

```bash
wget -c -P ~/variantcalling/reference/hg38 https://gcu-msc.s3.amazonaws.com/somatic-subset/GRCh38_chr17.fa https://gcu-msc.s3.amazonaws.com/somatic-subset/GRCh38_chr17.fa.fai https://gcu-msc.s3.amazonaws.com/somatic-subset/GRCh38_chr17.dict
```

### BQSR known-sites databases

```bash
wget -c -P ~/variantcalling/reference/BQSR https://gcu-msc.s3.amazonaws.com/somatic-subset/dbsnp138_chr17.final.vcf.gz https://gcu-msc.s3.amazonaws.com/somatic-subset/dbsnp138_chr17.final.vcf.gz.tbi https://gcu-msc.s3.amazonaws.com/somatic-subset/known_indels_chr17.final.vcf.gz https://gcu-msc.s3.amazonaws.com/somatic-subset/known_indels_chr17.final.vcf.gz.tbi https://gcu-msc.s3.amazonaws.com/somatic-subset/Mills_1000G_indels_chr17.final.vcf.gz https://gcu-msc.s3.amazonaws.com/somatic-subset/Mills_1000G_indels_chr17.final.vcf.gz.tbi
```

Verify:

```bash
ls -lh ~/variantcalling/gvar/input/HCC1954
ls -lh ~/variantcalling/reference/hg38
ls -lh ~/variantcalling/reference/BQSR
df -h /workspaces
```

---

## STEP 4 — Raw-read QC

```bash
fastqc -t $(nproc) ~/variantcalling/gvar/input/HCC1954/HCC1954_Normal_R1.fastq.gz ~/variantcalling/gvar/input/HCC1954/HCC1954_Normal_R2.fastq.gz -o ~/variantcalling/gvar/output/fastqc/pre-QC
```

Check R1:

```bash
unzip -p ~/variantcalling/gvar/output/fastqc/pre-QC/HCC1954_Normal_R1_fastqc.zip '*/summary.txt'
```

Check R2:

```bash
unzip -p ~/variantcalling/gvar/output/fastqc/pre-QC/HCC1954_Normal_R2_fastqc.zip '*/summary.txt'
```

Open the `.html` reports through the Terminal 2 web server.

For this teaching dataset, trimming may be skipped when read quality and adapter content are acceptable.

---

## STEP 5 — Alignment

### Build BWA index

```bash
bwa index ~/variantcalling/reference/hg38/GRCh38_chr17.fa
```

Verify:

```bash
ls -lh ~/variantcalling/reference/hg38
```

### Align paired-end reads and sort to BAM

Read groups are deliberately added in STEP 6 using Picard.

```bash
bwa mem -t $(nproc) ~/variantcalling/reference/hg38/GRCh38_chr17.fa ~/variantcalling/gvar/input/HCC1954/HCC1954_Normal_R1.fastq.gz ~/variantcalling/gvar/input/HCC1954/HCC1954_Normal_R2.fastq.gz | samtools sort -@ $(nproc) -o ~/variantcalling/gvar/output/alignment/HCC1954_Normal.sorted.bam
```

Index:

```bash
samtools index -@ $(nproc) ~/variantcalling/gvar/output/alignment/HCC1954_Normal.sorted.bam
```

Alignment statistics:

```bash
samtools flagstat -@ $(nproc) ~/variantcalling/gvar/output/alignment/HCC1954_Normal.sorted.bam
```

---

## STEP 6 — Add read groups and mark duplicates

### Add read groups

```bash
picard AddOrReplaceReadGroups I=~/variantcalling/gvar/output/alignment/HCC1954_Normal.sorted.bam O=~/variantcalling/gvar/output/alignment/HCC1954_Normal.RG.sorted.bam RGID=HCC1954_NORMAL RGLB=HCC1954_NORMAL RGPL=ILLUMINA RGPU=HCC1954_NORMAL RGSM=HCC1954_NORMAL SORT_ORDER=coordinate CREATE_INDEX=true
```

Verify:

```bash
samtools view -H ~/variantcalling/gvar/output/alignment/HCC1954_Normal.RG.sorted.bam | grep '^@RG'
```

### Mark duplicates

```bash
gatk MarkDuplicates -I ~/variantcalling/gvar/output/alignment/HCC1954_Normal.RG.sorted.bam -O ~/variantcalling/gvar/output/markduplicates/HCC1954_Normal.markdup.bam -M ~/variantcalling/gvar/output/markduplicates/HCC1954_Normal.markdup.metrics.txt --CREATE_INDEX true
```

Inspect metrics:

```bash
cat ~/variantcalling/gvar/output/markduplicates/HCC1954_Normal.markdup.metrics.txt
```

Compare BAM statistics:

```bash
samtools flagstat -@ $(nproc) ~/variantcalling/gvar/output/markduplicates/HCC1954_Normal.markdup.bam
```

`MarkDuplicates` flags duplicates; it does not remove them with the settings used here.

---

## STEP 7 — Base Quality Score Recalibration (BQSR)

### Build recalibration model

```bash
gatk BaseRecalibrator -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -I ~/variantcalling/gvar/output/markduplicates/HCC1954_Normal.markdup.bam --known-sites ~/variantcalling/reference/BQSR/dbsnp138_chr17.final.vcf.gz --known-sites ~/variantcalling/reference/BQSR/known_indels_chr17.final.vcf.gz --known-sites ~/variantcalling/reference/BQSR/Mills_1000G_indels_chr17.final.vcf.gz -O ~/variantcalling/gvar/output/BQSR/HCC1954_Normal.recal.table
```

Inspect:

```bash
head -40 ~/variantcalling/gvar/output/BQSR/HCC1954_Normal.recal.table
```

### Apply BQSR

```bash
gatk ApplyBQSR -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -I ~/variantcalling/gvar/output/markduplicates/HCC1954_Normal.markdup.bam --bqsr-recal-file ~/variantcalling/gvar/output/BQSR/HCC1954_Normal.recal.table -O ~/variantcalling/gvar/output/BQSR/HCC1954_Normal.recal.bam
```

Index:

```bash
samtools index -@ $(nproc) ~/variantcalling/gvar/output/BQSR/HCC1954_Normal.recal.bam
```

Check integrity:

```bash
samtools quickcheck -v ~/variantcalling/gvar/output/BQSR/HCC1954_Normal.recal.bam
```

No output means no problem was detected.

---

## STEP 8 — Germline variant calling

### HaplotypeCaller in GVCF mode

```bash
gatk HaplotypeCaller -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -I ~/variantcalling/gvar/output/BQSR/HCC1954_Normal.recal.bam -O ~/variantcalling/gvar/output/variants/HCC1954_Normal.g.vcf.gz -ERC GVCF --native-pair-hmm-threads $(nproc)
```

### Genotype the GVCF

```bash
gatk GenotypeGVCFs -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -V ~/variantcalling/gvar/output/variants/HCC1954_Normal.g.vcf.gz -O ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.vcf.gz
```

Count all variants:

```bash
bcftools view -H ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.vcf.gz | wc -l
```

Count SNPs:

```bash
bcftools view -v snps -H ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.vcf.gz | wc -l
```

Count indels:

```bash
bcftools view -v indels -H ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.vcf.gz | wc -l
```

Statistics:

```bash
bcftools stats ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.vcf.gz
```

---

## STEP 9 — Separate SNPs and indels

### SNPs

```bash
gatk SelectVariants -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -V ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.vcf.gz --select-type-to-include SNP -O ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.snps.vcf.gz
```

### Indels

```bash
gatk SelectVariants -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -V ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.vcf.gz --select-type-to-include INDEL -O ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.indels.vcf.gz
```

---

## STEP 10 — Hard-filter SNPs

```bash
gatk VariantFiltration -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -V ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.snps.vcf.gz --filter-expression "QD < 2.0" --filter-name "QD2" --filter-expression "QUAL < 30.0" --filter-name "QUAL30" --filter-expression "SOR > 3.0" --filter-name "SOR3" --filter-expression "FS > 60.0" --filter-name "FS60" --filter-expression "MQ < 40.0" --filter-name "MQ40" --filter-expression "MQRankSum < -12.5" --filter-name "MQRankSum-12.5" --filter-expression "ReadPosRankSum < -8.0" --filter-name "ReadPosRankSum-8" -O ~/variantcalling/gvar/output/variants/HCC1954_Normal.filtered.snps.vcf.gz
```

---

## STEP 11 — Hard-filter indels

```bash
gatk VariantFiltration -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -V ~/variantcalling/gvar/output/variants/HCC1954_Normal.raw.indels.vcf.gz --filter-expression "QD < 2.0" --filter-name "QD2" --filter-expression "QUAL < 30.0" --filter-name "QUAL30" --filter-expression "FS > 200.0" --filter-name "FS200" --filter-expression "ReadPosRankSum < -20.0" --filter-name "ReadPosRankSum-20" -O ~/variantcalling/gvar/output/variants/HCC1954_Normal.filtered.indels.vcf.gz
```

> These are hard-filtering thresholds for this teaching workflow.

---

## STEP 12 — Extract PASS variants

### PASS SNPs

```bash
gatk SelectVariants -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -V ~/variantcalling/gvar/output/variants/HCC1954_Normal.filtered.snps.vcf.gz --exclude-filtered -O ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.snps.vcf.gz
```

### PASS indels

```bash
gatk SelectVariants -R ~/variantcalling/reference/hg38/GRCh38_chr17.fa -V ~/variantcalling/gvar/output/variants/HCC1954_Normal.filtered.indels.vcf.gz --exclude-filtered -O ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.indels.vcf.gz
```

Count PASS SNPs:

```bash
bcftools view -H ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.snps.vcf.gz | wc -l
```

Count PASS indels:

```bash
bcftools view -H ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.indels.vcf.gz | wc -l
```

---

## STEP 13 — Combine final PASS SNPs + indels

```bash
bcftools concat -a -Ou ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.snps.vcf.gz ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.indels.vcf.gz | bcftools sort -Oz -o ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.vcf.gz
```

Index:

```bash
bcftools index -t ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.vcf.gz
```

Count:

```bash
bcftools view -H ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.vcf.gz | wc -l
```

---

## STEP 14 — Database annotation: dbSNP

Use the chromosome 17 dbSNP VCF already downloaded for BQSR.

```bash
bcftools annotate -a ~/variantcalling/reference/BQSR/dbsnp138_chr17.final.vcf.gz -c ID -Oz -o ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.vcf.gz ~/variantcalling/gvar/output/variants/HCC1954_Normal.PASS.vcf.gz
```

Index:

```bash
bcftools index -t ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.vcf.gz
```

View:

```bash
bcftools query -f '%CHROM	%POS	%ID	%REF	%ALT	%QUAL	%FILTER[	%GT	%DP	%GQ]
' ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.vcf.gz | head -30
```

Count variants with a database ID:

```bash
bcftools query -f '%ID
' ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.vcf.gz | grep -vc '^\.$'
```

---

## STEP 15 — Clinical database annotation: ClinVar

ClinVar is an NCBI database relating human genetic variants to clinical phenotypes and classifications.

### Download GRCh38 ClinVar

```bash
wget -c -P ~/variantcalling/reference/annotation https://ftp.ncbi.nlm.nih.gov/pub/clinvar/vcf_GRCh38/clinvar.vcf.gz https://ftp.ncbi.nlm.nih.gov/pub/clinvar/vcf_GRCh38/clinvar.vcf.gz.tbi
```

### Extract chromosome 17

```bash
bcftools view -r 17 ~/variantcalling/reference/annotation/clinvar.vcf.gz -Oz -o ~/variantcalling/reference/annotation/clinvar.GRCh38.17.vcf.gz
```

Index:

```bash
bcftools index -t ~/variantcalling/reference/annotation/clinvar.GRCh38.17.vcf.gz
```

### Rename `17` to `chr17`

```bash
printf "17	chr17
" > ~/variantcalling/reference/annotation/clinvar.rename.txt
```

```bash
bcftools annotate --rename-chrs ~/variantcalling/reference/annotation/clinvar.rename.txt -Oz -o ~/variantcalling/reference/annotation/clinvar.GRCh38.chr17.vcf.gz ~/variantcalling/reference/annotation/clinvar.GRCh38.17.vcf.gz
```

Index:

```bash
bcftools index -t ~/variantcalling/reference/annotation/clinvar.GRCh38.chr17.vcf.gz
```

Check:

```bash
bcftools view -H ~/variantcalling/reference/annotation/clinvar.GRCh38.chr17.vcf.gz | head
```

The first column should be `chr17`.

### Annotate final variants with ClinVar

```bash
bcftools annotate -a ~/variantcalling/reference/annotation/clinvar.GRCh38.chr17.vcf.gz -c INFO/CLNSIG,INFO/CLNDN,INFO/CLNREVSTAT,INFO/GENEINFO -Oz -o ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.clinvar.vcf.gz ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.vcf.gz
```

Index:

```bash
bcftools index -t ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.clinvar.vcf.gz
```

---

## STEP 16 — View annotated variants

Show variants with ClinVar clinical-significance annotations:

```bash
bcftools view -i 'INFO/CLNSIG!="."' ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.clinvar.vcf.gz
```

Create a compact TSV table:

```bash
bcftools query -i 'INFO/CLNSIG!="."' -f '%CHROM	%POS	%ID	%REF	%ALT	%INFO/GENEINFO	%INFO/CLNSIG	%INFO/CLNDN	%INFO/CLNREVSTAT
' ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.clinvar.vcf.gz > ~/variantcalling/gvar/output/annotation/HCC1954_Normal.ClinVar.tsv
```

View:

```bash
column -t -s $'	' ~/variantcalling/gvar/output/annotation/HCC1954_Normal.ClinVar.tsv | head -30
```

Show variants containing a `Pathogenic` ClinVar classification:

```bash
bcftools query -i 'INFO/CLNSIG~"Pathogenic"' -f '%CHROM	%POS	%ID	%REF	%ALT	%INFO/GENEINFO	%INFO/CLNSIG	%INFO/CLNDN	%INFO/CLNREVSTAT
' ~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.clinvar.vcf.gz
```

> Not every called variant is present in ClinVar. Missing ClinVar annotation does **not** mean the variant is benign.

---

## STEP 17 — Final output checks

```bash
find ~/variantcalling/gvar/output -type f -printf '%p	%k KB
' | sort
```

```bash
df -h /workspaces
```

Final annotated VCF:

```text
~/variantcalling/gvar/output/annotation/HCC1954_Normal.PASS.dbsnp.clinvar.vcf.gz
```

Final ClinVar table:

```text
~/variantcalling/gvar/output/annotation/HCC1954_Normal.ClinVar.tsv
```

---

## Pipeline summary

```text
FASTQ R1 + R2
      ↓
FastQC
      ↓
BWA-MEM
      ↓
coordinate-sorted BAM
      ↓
Read Groups
      ↓
MarkDuplicates
      ↓
BQSR
      ↓
HaplotypeCaller (GVCF)
      ↓
GenotypeGVCFs
      ↓
raw VCF
      ↓
SNP / INDEL separation
      ↓
hard filtering
      ↓
PASS variants
      ↓
dbSNP annotation
      ↓
ClinVar annotation
      ↓
final annotated VCF + TSV
```

## Important teaching notes

- FastQC sequence duplication and Picard duplicate marking are different analyses.
- `MarkDuplicates` marks likely duplicate reads; it does not remove them in this workflow.
- BQSR changes base-quality scores, not genomic alignment coordinates.
- GVCF mode records variant and non-variant confidence information for later genotyping.
- dbSNP membership is not evidence that a variant is pathogenic.
- ClinVar classifications must be interpreted with review status, phenotype, inheritance, allele frequency, and other evidence.
- This is a **teaching pipeline using a chromosome 17 subset**, not a complete production-scale clinical germline workflow.
