# somatico
## Descrição
Primeiro Pipeline de análise somática

## Antes de começar

### Link para os arquivos BAM
```bash
https://github.com/renatopuga/somatico
```

### Nossas amostras
WP190 (tumor) e WP191 (normal)

WP017 (tumor) e WP018 (normal)

## Ferramentas utilizadas 

### **GATK** 
> 4.2.2.0

### **samtools**
> Version 1.19.2

## Baixando as referências chr13 e chr19

**Download chr13**
```bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr13.fa.gz
```
**Download chr19**
```bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr19.fa.gz
```

## Concatenando os arquivos .fa.gz
> O zcat lê arquivos compactados .gz e zip
```bash
zcat chr13.fa.gz chr19.fa.gz | sed -e "s/chr//g" > hg19.fa
```

## Gerando o index do arquivo hg19.fa
````bash
samtools faidx hg19.fa
````

## Conferindo o índice que foi gerado
```bash
cat hg19.fa.fai
```
## Output esperado
13 | 115169878 | 4         | 50 | 51

19 | 59128983  | 117473284 | 50 | 51

## Criando o hg19.dict
````bash
./gatk-4.2.2.0/gatk CreateSequenceDictionary -R hg19.fa -O hg19.dict
````

## Criando o hg19 intervals
````bash
./gatk-4.2.2.0/gatk ScatterIntervalsByNs \
-R hg19.fa \
-O hg19.interval_list \
-OT ACGT
````

## Conferindo o ID das amostras normais
````bash
samtools view -H tumor_wp191.bam | grep RG | cut -f5 | sed -e "s/SM://g"
samtools view -H tumor_wp018.bam | grep RG | cut -f6 | sed -e "s/SM://g"
````

## Output esperado
WP191
WP018

## Chamando as variantes com Mutect2
### Amostras WP190 (tumor) e WP191 (normal) 
```bash
./gatk-4.2.2.0/gatk Mutect2 \
-R hg19.fa \
-I tumor_wp190.bam \
-I tumor_wp191.bam \
-normal WP191 \
--germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
-O wp190-somatic.vcf.gz \
-L hg19.interval_list
````

### Amostras WP017 (tumor ) e WP018 (normal)
````bash
./gatk-4.2.2.0/gatk Mutect2 \
-R hg19.fa \
-I tumor_wp017.bam \
-I tumor_wp018.bam \
-normal WP018 \
--germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
-O wp017-somatic.vcf.gz \
-L hg19.interval_list
````

## Calculando a contaminação Tumor WP190 e WP017
````bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
-I tumor_wp190.bam \
-V af-only-gnomad-chr13-chr19.vcf.gz \
-L hg19.interval_list \
-O tumor_wp190.table
````

````bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
-I tumor_wp017.bam \
-V af-only-gnomad-chr13-chr19.vcf.gz \
-L hg19.interval_list \
-O tumor_wp017.table
````


## Normais WP191 e WP018
````bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
-I tumor_wp191.bam \
-V af-only-gnomad-chr13-chr19.vcf.gz \
-L hg19.interval_list \
-O tumor_wp191.table
````

````bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
-I tumor_wp018.bam \
-V af-only-gnomad-chr13-chr19.vcf.gz \
-L hg19.interval_list \
-O tumor_wp018.table
````

## Calculate contamination
````bash
./gatk-4.2.2.0/gatk CalculateContamination \
-I tumor_wp190.table \
-matched tumor_wp191.table \
-O contamination_wp190-wp191.table
````
````bash
./gatk-4.2.2.0/gatk CalculateContamination \
-I tumor_wp017.table \
-matched tumor_wp018.table \
-O contamination_wp017-wp018.table
````

## FilterMutectCalls
````bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
-R hg19.fa \
-V wp190-somatic.vcf.gz \
--contamination-table contamination_wp190-wp191.table \
-O wp190-wp191-filtered.vcf.gz
````
````bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
-R hg19.fa \
-V wp017-somatic.vcf.gz \
--contamination-table contamination_wp017-wp018.table \
-O wp017-wp018-filtered.vcf.gz
````

## Conferindo o número de variantes encontradas
```bash
zcat wp190-wp191-filtered.vcf.gz | grep -v "\#" | wc -l 
zcat wp017-wp018-filtered.vcf.gz | grep -v "\#" | wc -l
```

## Conferindo com PASS
```bash
zcat wp190-wp191-filtered.vcf.gz | grep -v "\#" | grep PASS | wc -l 
zcat wp017-wp018-filtered.vcf.gz | grep -v "\#" | grep PASS | wc -l
```
