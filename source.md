Project2 command history

# quality report
```sh
mkdir fastqc_sample
fastqc sample.fastq.gz -o fastqc_sample

mkdir fastqc_paired
fastqc sample1.fastq.gz sample2.fastq.gz -o fastqc_sample
```

- [X] `quality_report/fastqc_control`
- [X] `quality_report/fastqc_tumor`

# hg38.fa
```sh
samtools faidx hg38.fa
samtools dict hg38.fa > hg38.dict
bwa index hg38.fa
```
- [X] done

# Read Group tags
zcat sample_R1.fastq.gz | grep "^@" | head -n 1

@K00150:233:HL5YTBBXX:6:1101:1621:33457
@HWI-ST1153:198:C431LACXX:4:1101:1164:83094
- [X] ID=C431LACXX.Sample
- [X] SM=Sample # !!! use different names for Control and Tumor !!!
- [X] PL=ILLUMINA
- [X] LB=C431LACXX.Sample
- [X] PU=ST1153.198.C431LACXX
- [X] CN=FIITSTU
- [X] DT=2026-01-01T00:00:00
Combine components into RG tag:
- [X] RG="@RG\tID:$ID\tSM:$SM\tPL:$PL\tLB:$LB\tPU:$PU\tCN:$CN\tDT:$DT"

```sh
bwa mem -t 8 -R "$RG" hg38.fa sample_R1.fastq.gz sample_R2.fastq.gz | samtools view -hbS -o sample.bwa.bam # 40 mins
samtools sort -o sample.bwa.sorted.bam sample.bwa.bam
samtools index sample.bwa.sorted.bam
```
- [X] control
- [X] tumor

skipped:
```sh
  406  bowtie2 --threads 8 -x hg38_bt2_index -1 sample_R1.fastq.gz -2 sample_R2.fastq.gz --rg-id $ID --rg SM:$SM --rg PL:$PL --rg LB:$LB --rg PU:$PU --rg CN:$CN --rg DT:$DT | samtools view -hbS -o sample.bowtie2.bam
  407  samtools sort -o sample.bowtie2.sorted.bam sample.bowtie2.bam
  408  samtools index sample.bowtie2.sorted.bam
  409  samtools view sample.bwa.sorted.bam | less
  410  samtools view -H sample.bwa.sorted.bam
  411  samtools view sample.bwa.sorted.bam chr22:10000000-11000000 | less
  412  samtools view -h sample.bowtie2.sorted.bam chr22:10000000-11000000 > chr22_1Mb.sam
```

```sh
samtools stats sample.bwa.sorted.bam>sample.bwa.stats.txt
samtools flagstat sample.bwa.sorted.bam > sample.flagstat.txt; 
```
- [X] control
- [X] tumor
```sh
head sample.bwa.stats.txt 
plot-bamstats -p sample.bwa.stats_plots/ sample.bwa.stats.txt
```
- [ ] control
- [ ] tumor

skipped:
```sh
  415  samtools coverage sample.bwa.sorted.bam
  416  samtools coverage --histogram sample.bwa.sorted.bam
```

# Cv 4
```
bcftools annotate --rename-chrs <(awk '{print $1, "chr"$1}' contig_map.txt) -O z -o dbsnp.chr.vcf.gz common_all_20180418.vcf.gz
bcftools index -t dbsnp.chr.vcf.gz
```
- [X] done

skipped:
```sh
Qualimap bamqc -bam sample.bwa.sorted.bam -outdir qualimap_report
```

```
gatk MarkDuplicates -I sample.bwa.sorted.bam -O sample.bwa.dedup.bam -M sample.dup_metrics.txt --CREATE_INDEX
gatk BaseRecalibrator -R hg38.fa -I sample.bwa.dedup.bam --known-sites dbsnp.vcf -O sample.recal_data.recalib
gatk ApplyBQSR -R hg38.fa -I sample.bwa.dedup.bam --bqsr-recal- le sample.recal_data.recalib -O Sample.bam
```
- [X] control
- [X] tumor

```sh
gatk CollectAlignmentSummaryMetrics -R hg38.fa -I Sample.bam -O sample.cleaned.alignment_metrics.txt
```
or
```sh
samtools stats sample.bwa.sorted.bam > sample.bwa.stats.txt
samtools flagstat sample.bwa.sorted.bam > sample.flagstat.txt; 
```
- [ ] control
- [ ] tumor

# Variant calls
## somatic variant calls
- [X]
```
gatk HaplotypeCaller -ERC GVCF -R hg38.fa -I Tumor.bam -O Tumor.g.vcf.gz --dbsnp dbsnp.chr.vcf.gz
gatk GenotypeGVCFs -R hg38.fa -V Tumor.g.vcf.gz -O Tumor.vcf.gz
bcftools stats Tumor.vcf.gz > Tumor.vcf.stats
```

- [X] 
```
NORMAL_SAMPLE=$(samtools samples Control.bam | cut -f1)
gatk Mutect2 -R hg38.fa -I Tumor.bam -I Control.bam -normal $NORMAL_SAMPLE -O Tumor_Control.vcf.gz
gatk FilterMutectCalls -R hg38.fa -V Tumor_Control.vcf.gz -O Tumor_Control.filtered.vcf.gz
```

- [X] stats and plots
```
bcftools stats Tumor_Control.filtered.vcf.gz > Tumor_Control.filtered.vcf.stats
plot-vcfstats -p plots/ Tumor_Control.filtered.vcf.stats
```

## Structural Variant Calling
### Manta
- [X] conﬁgure Manta SV caller
```
configManta.py --bam Tumor.bam --referenceFasta hg38.fa --runDir manta_run_dir
```
--bam # Input cleaned BAM ﬁle
--referenceFasta # Reference genome
--runDir # Output directory for Manta run

- [X] run Manta SV caller
`./manta_run_dir/runWorkflow.py -m local -j 8`

- [X] View the SV VCF ﬁle
```
less manta_run_dir/results/variants/diploidSV.vcf.gz
less manta_run_dir/results/variants/candidateSV.vcf.gz
```

### Delly
- [X] Call structural variants using Delly
```
delly call -g hg38.fa -o Tumor.delly.bcf Tumor.bam
-g# Reference genome
-o# Output BCF ﬁle
```

- [X] Input cleaned BAM ﬁle
```
bcftools view -i 'QUAL>20' Tumor.delly.bcf -Oz -o Tumor.delly.filtered.vcf.gz
```

- [x] View the Delly SV VCF ﬁle
```
zless Tumor.delly.filtered.vcf.gz
```

### Compare the results from Manta and Delly 
- [X] using hap.py
```
hap.py -r hg38.fa -o ../manta_vs_delly_germline Tumor.delly.filtered.vcf.gz manta_run_dir/results/variants/diploidSV.vcf.gz
```

## Somatic Structural Variant Calling 
### Manta
- [x] Configure Manta for somatic SV calling
```
configManta.py --normalBam Control.bam --tumorBam Tumor.bam --referenceFasta hg38.fa --runDir manta_somatic_run_dir
```
--normalBam# Input normal BAM file
--tumorBam# Input tumor BAM file
--referenceFasta# Reference genome
--runDir# Output directory for Manta run

- [x] Run Manta workflow
```
./manta_somatic_run_dir/runWorkflow.py -m local -j 8
```

- [x] View the somatic SV VCF file
```
less manta_somatic_run_dir/results/variants/somaticSV.vcf.gz
```

### Delly
- [X] Call somatic structural variants using Delly
```
delly call -g hg38.fa -o Tumor_Control.delly.bcf Tumor.bam Control.bam
```

- [X] supply a sample map file that indicates which sample is tumor and which is normal
```
TUMOR_SAMPLE=$(samtools samples Tumor.bam | cut -f1)
NORMAL_SAMPLE=$(samtools samples Control.bam | cut -f1)
echo -e "${TUMOR_SAMPLE}\ttumor\n${NORMAL_SAMPLE}\tcontrol" > sample_map.tsv
```

- [X] Now, we need to filter the somatic SV calls
```
delly filter -f somatic -o Tumor_Control.delly.filtered.bcf -s sample_map.tsv Tumor_Control.delly.bcf
```
-f # Filter for somatic variants
-o # Output filtered VCF file
-s # Sample map

# Cv 7 - annotations
- [X] online

- [X] Funcotator
```
gatk Funcotator \ 
-R hg38.fa \
-V Tumor_Control.filtered.vcf.gz \
-O Tumor_Control.annotated.maf.gz \
--data-sources-path path/to/funcotator_db \
--ref-version hg38 \
--output-file-format MAF
```

## Visualising annotations
```
R
library(maftools)
maf <- read.maf("Tumor_Control.annotated.maf.gz")
```

- [X] Generate summary statistics
```
plotmafSummary(maf = maf, addStat = 'median')
```


- [X] Visualize mutation patterns
Since we have only one sample, 
the oncoplot will show black lines 
(multi-hit)for the top mutated genes. 
If we had multiple samples, it would show a heatmap.
```
oncoplot(maf, top = 20)
maf.titv <- titv(maf = maf, plot = FALSE)
plotTiTv(res = maf.titv)
```

- [X] Visualise the position of mutations in a speciﬁc gene
```
lollipopPlot(maf = maf, gene = "ADAM21")
```

- [X] Generate a rainfall plot and comparewith TCGA
visualize mutation distribution across the genome with rainfall plot
and compare our sample's mutation burden with TCGA samples.
```
rainfallPlot(maf = maf)
tcgaCompare(maf = maf, cohortName = "MySample", logScale = TRUE)
```

- [X] Filter out top 10 mutated genes and save them to a TSV ﬁle
```
top.genes <- subsetMaf(maf, genes = getGeneSummary(maf)$Hugo_Symbol[1:10], mafObj = FALSE)
write.table(top.genes, ﬁle = "top_10_mutated_genes.tsv", sep = "\t", row.names = FALSE, quote = FALSE)
```
