# DEqMS
DEqMS is a tool for quantitative proteomic analysis, developped by Yafeng Zhu @ Karolinska Institutet. Manuscript in submission.

## Installation
git clone https://github.com/yafeng/DEqMS

or click green button (clone or download) choose Download ZIP, and unzip it.

## Introduction
DEqMS works on top of Limma. However, Limma assumes same prior variance for all genes, the function `spectra.count.eBayes` in DEqMS package  is able to correct the biase of prior variance estimate for genes identified with different number of PSMs/peptides. It works in a similar way to the intensity-based hierarchical Bayes method (Maureen A. Sartor et al BMC Bioinformatics 2006). Intead of locally weighted regression (Maureen et al 2006) between prior variance and intensity, DEqMS use `nls` function with an explicit fomula (Var ~ const+A/(spectra.count)) to fit prior variance against PSM/peptide count.

Outputs of `spectra.count.eBayes`:

object is augmented form of "fit" object from `eBayes` in Limma, with the additions being:

`sca.t`     - Spectra Count Adjusted posterior t-value

`sca.p`     - Spectra Count Adjusted posterior p-value

`sca.dfprior` - estimated prior degrees of freedom

`sca.priorvar`- estimated prior variance

`sca.postvar` - estimated posterior variance

`nls.model` - fitted non-linear model

## Usage
### 1. load R packages
```{r}
source("DEqMS.R")
library(matrixStats)
library(plyr)
library(limma)
```

### 2. Read input data and generate count table.
The first two columns in input table should be peptide sequence and protein/gene names, intensity values for different samples start from 3rd columns.
```{r}
dat.psm = read.table("input_table.txt",stringsAsFactors = FALSE,sep="\t",header=T,comment.char = "",row.names = NULL)
dat.psm = na.omit(dat.psm)   # remove rows with missing values
dat.psm = dat.psm[!dat.psm$Protein.Group.Accessions=="",]  # remove rows with missing protein ID

dat.psm.log = dat.psm
dat.psm.log[,3:12] =  log2(dat.psm[,3:12])  # log2 transformation

count.table = as.data.frame(table(dat.psm$Protein.Group.Accessions)) # generate count table
rownames(count.table)=count.table$Var1
```
### 3. Generate sample annotation table.
```{r}
cond = c("ctrl","miR372","miR191","ctrl","miR519",
"miR519","miR372","miR191","ctrl","miR372")

sampleTable <- data.frame(
row.names = colnames(dat.psm)[3:12],
cond = as.factor(cond)
)
```

### 4. Summarization and Normalization
Before summarizing PSMs/peptides into protein abundance, we calculate a relative abundance for each PSMs/peptides.
You can substract PSMs/peptides log2 intensity by the mean log2 intensities of control samples, or do median sweeping as shown  in Gina et al JPR 2017.

use control samples to calculate relative ratios. `group_col` is the column number you want to group by, set 2 if genes/proteins are in second column. `ref_col`  is the column where control samples are.
```{r}
dat.gene = median.summary(dat.psm.log,group_col = 2, ref_col=c(3,6,11))
dat.gene.nm = equal.median.normalization(dat.gene)

#or you can use median sweeping method (Gina et al JPR 2017)
#median.sweep does equal.median.normalization for you automatically

data.gene.nm = median.sweep(dat.psm.log,group_col = 2)

#or you can use Factor Analysis for Robust Microarray Summarization (FARMS)
#see (Hochreiter S et al Bioinformatic 2007, Zhang B et al MCP 2017)
#input is psm raw intensity, not log transformed values

dat.gene = farms.summary(dat.psm,group_col = 2)
dat.gene.nm = equal.median.normalization(dat.gene)

```

### 5. Differential expression analysis
```{r}
gene.matrix = as.matrix(dat.gene.nm)
design = model.matrix(~cond,sampleTable)

fit1 <- eBayes(lmFit(gene.matrix,design))
fit1$count <- count.table[names(fit1$sigma),]$Freq  # add PSM/peptide count values
fit2 = spectra.count.eBayes(fit1,coef_col=3) # two arguements, a fit object from eBayes() output, and the column number of coefficients
```

### 6. Output the results
```{r}
sca.results = output_result(fit2,coef_col=3)
write.table(sca.results, "DEqMS.analysis.out.txt", quote=F,sep="\t",row.names = F)
```

## Package vignette
more functioanlities are in HTML vignette.  [Go to HTML vignette](https://yafeng.github.io/DEqMS/index.html)




