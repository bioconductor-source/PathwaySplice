[![Travis-CI Build Status](https://travis-ci.org/SCCC-BBC/PathwaySplice.svg?branch=master)](https://travis-ci.org/SCCC-BBC/PathwaySplice)
[![codecov](https://codecov.io/github/SCCC-BBC/PathwaySplice/coverage.svg?branch=master)](https://codecov.io/github/SCCC-BBC/PathwaySplice)
[![CRAN_Status_Badge](http://www.r-pkg.org/badges/version/PathwaySplice)]

# PathwaySplice
Pathway analysis for alternative splicing in RNA-seq datasets that accounts for different number of gene features

# Introduction

In alternative splicing ananlysis of RNASeq data, one popular approach is to first identify gene features (e.g. exons or junctions) significantly associated with splicing using methods such as DEXSeq [Anders2012] or JunctionSeq [Hartley2016], and then perform pathway analysis based on the list of genes associated with the significant gene features. 

For DEXSeq results, we use _gene features_ to refers to non-overlapping exon counting bins [Anders2012, Figure 1], while for JunctionSeq results, _gene features_ refers to non-overlapping exon or splicing junction counting bins. 

A major challenge is that without explicit adjustment, pathways analysis would be biased toward pathways that include genes with a large number of gene features, because these genes are more likely to be selected as "significant genes" in pathway analysis.  

PathwaySplice is an R package that falicitate the folowing analysis: 

1. Performing pathway analysis that explicitly adjusts for the number of exons or junctions associated with each gene; 
2. Visualizing selection bias due to different number of exons or junctions for each gene and formally tests for presence of bias using logistic regression; 
3. Supporting gene sets based on the Gene Ontology terms, as well as more broadly defined gene sets (e.g. MSigDB) or user defined gene sets; 
4. Identifing the significant genes driving pathway significance and 
5. Organizing significant pathways with an enrichment map, where pathways with large number of overlapping genes are grouped together in a network graph.


# Quick start on using PathwaySplice

After installation, the PathwaySplice package can be loaded into R using:
```{r eval=TRUE, message=FALSE, warning=FALSE, results='hide'}
library(PathwaySplice)
```

The latest version can also be installed by 
```{r eval=FALSE, message=FALSE, warning=FALSE, results='hide'}
library(devtools)
install_github("SCCC-BBC/PathwaySplice",ref = 'development')
```

The input file of PathwaySplice are p-values for multiple gene features associated with each gene. This information can be obtained from DEXSeq [Anders2012] or JunctionSeq [Hartley2016] output files. As an example, PathwaySplice includes a feature based dataset within the package, based on a RNASeq study of CD34+ cells from myelodysplastic syndrome (MDS) patients with SF3B1 mutations (Dolatshad, et al., 2015). This dataset was downloaded from GEO database (GSE63569), we selected a random subset of 5000 genes here for demonstration. 

The example dataset can be loaded directly:
```{r eval=TRUE, warning=FALSE, message=FALSE, results='markup'}
data(featureBasedData)
head (featureBasedData)
```

Next the **makeGeneTable** function can be used to convert it to a gene based table. 
```{r eval=TRUE, message=FALSE, warning=FALSE, results='markup'}
gene.based.table <- makeGeneTable(featureBasedData)
head(gene.based.table)
```
Here `geneWisePvalue` is simply the lowest feature based p-value for the gene, `numFeature` is number of features for the gene, `fdr` is false discovery rate for `genewisePvalue`, `sig.gene` indicates if a gene is significant.  

To assess selection bias, i.e. whether gene with more features are more likely to be selected as significant genes, **lrTestBias** function fits a logistic regression with `logit (sig.gene) ~ numFeature`

```{r eval=TRUE, warning=FALSE, message=FALSE, results='markup', fig.height=5, fig.width=5}
lrTestBias(gene.based.table,boxplot.width=0.3)
```

To perform pathway analysis that adjusts for the number of gene features, we use the **runPathwangSplice** function, which implements the methodology described in [Young2010]. **runPathwangSplice** returns a   [tibble](https://cran.r-project.org/web/packages/tibble/vignettes/tibble.html) dataset  with statistical significance of the pathway (`over_represented_pvalue`), as well as the significant genes that drives pathway significance (`SIGgene_ensembl` and `SIGgene_symbol`). An additional bias plot that visualizes the relationship between the proportion of significant genes and the mean number of gene features within gene bins is also generated. 

```{r eval=TRUE,warning=FALSE,message=FALSE,results='markup'}
result.adjusted <- runPathwaySplice(gene.based.table,genome='hg19',
                        id='ensGene',
                        test.cats=c('GO:BP'),
                        go.size.limit=c(5,30),method='Wallenius')
head(result.adjusted)
```

To performe pathway analysis for other user defined databases, one needs to specify the pathway database in [.gmt format](http://software.broadinstitute.org/cancer/software/gsea/wiki/index.php/Data_formats) first and then use the `gmtGene2Cat` function before calling `pathwaySplice` function. 


```{r eval=TRUE, message=FALSE, warning=FALSE,results='hide',fig.show='hide'}
dir.name <- system.file('extdata', package='PathwaySplice')
hallmark.local.pathways <- file.path(dir.name,'h.all.v6.0.symbols.gmt.txt')
hlp <- gmtGene2Cat(hallmark.local.pathways, genomeID='hg19')
result.hallmark <- runPathwaySplice(gene.based.table,genome='hg19',id='ensGene',
                gene2cat=hlp, go.size.limit=c(5,200), method='Wallenius', binsize=20)
```                
For example, the results for the MSigDB Hallmark gene sets are
```{r eval=TRUE, message=FALSE, warning=FALSE}
head(result.hallmark)
```


Lastly, to visualize pathway analysis results in an enrichment network, we use the **enrichmentMap** function:

```{r eval=TRUE, warning=FALSE,message=FALSE,results ='markup', fig.align='center', fig.height=6, fig.width=6}
output.file.dir <- file.path("~/PathwaySplice_output")
enmap <- enrichmentMap(result.adjusted,n=5,
                       output.file.dir=output.file.dir,
                       similarity.threshold=0.3, scaling.factor = 2)
```

In the enrichment map, the `size of the nodes` indicates the number of significant genes within the pathway. The `color of the nodes` indicates pathway significance, where smaller p-values correspond to dark red color. Pathways with Jaccard coefficient > `similarity.thereshold` will be connected on the network. The `thickness of the edges` corresponds to Jaccard similarity coefficient between the two pathways, scaled by `scaling.factor`. A file named "network.layout.for.cytoscape.gml" is generated in the "~/PathwaySplice_output" directory. This file can be used as an input file for cytoscape software[Shannon2003], which allows users to further maually adjust appearance of the generated network. 

# Reference
<!-- Usage: rmarkdown::render("vignettes/tutorial.Rmd", output_format="all") --> 
<!-- Usage: rmarkdown::render("vignettes/tutorial.Rmd", output_format="all",encoding="utf8")(on windows) -->