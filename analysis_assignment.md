STAT 540 - 2024 Analysis Assignment
================
YOUR NAME

  - **Reminder:** When answering the questions, it is not sufficient to
    provide only code. Add explanatory text to interpret your results
    throughout.

  - **Total:** 30 POINTS

### Background

In this assignment, we will explore the complexity of Acute Myeloid
Leukemia (AML), an aggressive type of blood cancer, in multiomics data
integration. One of the biggest outstanding challenges in AML study is
that a wide variety of cancer-causing mutations has been frequently
spotted in genes involved in a hematopoietic stem cell (HSC)
differentiation process. Such mutations can dramatically alter
developmental/differentiation trajectories, increase heterogeneity among
cancer cells, and obfuscate downstream analysis.

We will use a comprehensive data set,
[“BeatAML2,”](https://biodev.github.io/BeatAML2/) which contains the
raw and processed gene expression data used in [Bottomly *et
al.*](https://www.sciencedirect.com/science/article/pii/S1535610822003129)
as well as other clinical, genetic, and drug response variables. We will
investigate how multiple types of high-dimensional sequencing results
can be combined in real-world settings. Here, we will focus on data
wrangling, exploratory data analysis, and differential gene expression
analysis, but many other scientific questions can be explored,
especially linking genetic and drug response variables.

#### Load libraries

Note: only a few libraries are provided for the initial steps; add
additional libraries as needed for the rest of the analyses.

``` r
library(tidyverse)
library(readxl)
library(edgeR)
library(limma)
```

We may use some handy functions over and over again.

``` r
## Avoid overwriting
if.needed <- function(.file, .code) {
    if(!all(file.exists(unlist(.file)))){ .code }
    stopifnot(all(file.exists(unlist(.file))))
}

## A quick function to concatenate two strings
`%&%` <- function(a,b) paste0(a,b)

## Feel free to define your own housekeeping functions
```

#### A primer for data management

Some helper code to read the data in is provided.

``` r
url <- "https://github.com/biodev/beataml2.0_data/raw/main/beataml_waves1to4_counts_dbgap.txt"
data.file <- "data/beataml_raw_counts.txt"
dir.create("data", recursive=T, showWarnings=F)

## Download data if needed; avoid overwriting
if.needed(data.file, download.file(url, destfile = data.file))

## Download clinical information if needed
url <- "https://github.com/biodev/beataml2.0_data/raw/main/beataml_wv1to4_clinical.xlsx"
clin.file <- "data/beataml_clinical.xlsx"
if.needed(clin.file, download.file(url, destfile = clin.file))
```

You might build your own `SummarizedExperiment` object, but we do not
need to make one for this project. We will preprocess the raw data files
and feed them into a `DGElist` object, which will be repeatedly used. We
can import the count matrix stored in “data/beataml\_raw\_counts.txt”
file, using a general purpose utility function, such as `read.table` or
`fread` in `data.tabble` package.

``` r
count.df <-
    data.table::fread(data.file, header = T, sep = "\t") %>%
    as.data.frame()
```

Note: `read.table` needed additional tuning since some fields contain
rather unconventional characters. Instead of using `read.table`
directly, here we read the data using `fread` and transformed the
`data.table` type into `data.frame`.

This study also comes with clinical information in Microsoft Excel
format. A function, `read_xlsx`, gives us a convenient and programmatic
way to read the necessary information. If you want to live up to
reproducible data science, it is highly encouraged to avoid
arbitrary/manual operations in your analysis pipeline.

``` r
.cols <- c(
    "dbgap_rnaseq_sample",           # unique sample ID
    "cohort",                        #
    "currentStage",                  #
    "consensus_sex",                 # demographic
    "vitalStatus",                   # survival information
    "overallSurvival",               #
    "plateletCount",                 # some lab tests
    "wbcCount",                      #
    "%.Basophils.in.PB",             # cell type fractions
    "%.Blasts.in.BM",                #
    "%.Blasts.in.PB",                #
    "%.Eosinophils.in.PB",           #
    "%.Immature.Granulocytes.in.PB", #
    "%.Lymphocytes.in.PB",           #
    "%.Monocytes.in.PB",             #
    "%.Neutrophils.in.PB",           #
    "%.Nucleated.RBCs.in.PB")

## Read data from .xlsx file;
## this will be a Tibble object
meta.df <-
    read_xlsx(clin.file, sheet="summary") %>%
    dplyr::select(all_of(.cols))

colnames(meta.df) <- gsub("[%][.]", "", colnames(meta.df))
```

In practice, many columns, which are supposed to be quantitative, may
come with corrupted, non-numeric values. Looking through every bit of
such columns will be laborious and may cause another type of unwanted
errors/formats. Let’s deal with such painful steps programmatically. We
will deal with it in a `tidyverse` (or `dplyr`) way or `data.table` way.
First, we can clean up the annotation data by applying the
`take.numeric` function “`dplyr::across`” all the columns except for the
categorical ones.

``` r
take.numeric <- function(x) {
    str_extract(x, pattern="\\d+") %>% as.numeric()
}

meta.df <- meta.df %>%
    mutate(across(
        -c(dbgap_rnaseq_sample, cohort, currentStage, consensus_sex, vitalStatus),    # excluding these columns
        ~. %>% take.numeric  # apply str_extract and as.numeric to all the other columns
    ))
```

*Side note:* You may `lapply` the `take.numeric` function within the
following `data.table` list.

``` r
.dt <- as.data.table(meta.df)

non.num.cols <- c("dbgap_rnaseq_sample",
                  "cohort",
                  "currentStage",
                  "consensus_sex",
                  "vitalStatus")

meta.clean.dt <-
    cbind(.dt[, .SD, .SDcols = non.num.cols],
          .dt[,
              lapply(.SD, take.numeric),
              .SDcols = ! non.num.cols])
```

### Question 1: Getting familiar with the data (5 POINTS)

#### A. Investigate the data matrix (1 pt).

How many rows and columns do you see in the data (`count.df`), and what
do they correspond to (genes/features or samples/individuals)?

  - Number of rows:

<!-- end list -->

``` r
# YOUR CODE HERE
```

  - Number of columns:

<!-- end list -->

``` r
# YOUR CODE HERE
```

#### B. What are the features (1 pt)?

One of the columns, `biotype`, indicates the biological type of each row
(feature). Can you count how many types exist and how many of each type
we have? If possible, sort the rows in the descending order of
frequency.

  - HINT: You can summarize these as a table.

<!-- end list -->

``` r
# YOUR CODE HERE
```

#### D. Match multiple data types (1 pt).

Subset the rows in the metadata `meta.df` and the columns in the count
data `count.df` to include only those samples for which we have gene
counts for downstream analysis. HINT: Keep in mind that not every sample
in the data is present in the metadata table, and vice versa. This is
one of the most frequently error-prone steps in data analysis.

  - Create a new data matrix `matched.count.df` by only retaining
    samples with matched clinical information (0.5 pt). Also, report the
    size of the resulting data matrix.

<!-- end list -->

``` r
# YOUR CODE HERE
```

  - Also create annotation information `matched.meta.dt` restricting on
    the matched samples (0.5 pt).

<!-- end list -->

``` r
# YOUR CODE HERE
```

#### E. Add additional information (cell type scores) (1 pt)

Tumour microenvironments shaped by cellular environments help understand
the disease development and progression mechanisms. However, cell type
fraction information is frequently missed in many samples. This study
provides another data file of inferred cell type scores to compensate
for the missing cell type fractions.

``` r
## Download inferred cell type scores
url <- "https://github.com/biodev/beataml2.0_data/raw/main/beataml2_manuscript_vg_cts.xlsx"
cts.file <- "data/beataml_celltypes.xlsx"
if.needed(cts.file, download.file(url, destfile = cts.file))
cts.df <- read_xlsx(cts.file)
```

  - Update the meta data (`matched.meta.df`) by adding cell type scores
    (variables ending with “-like”), matching by `dbgap_rnaseq_sample`
    column. Show the top five rows after the update (0.5 pt).

<!-- end list -->

``` r
# YOUR CODE HERE
```

  - Show that there is a strong correspondence between the observe and
    estimated cell type fractions by displaying a correlation heatmap.
    Provide your own interpretations (0.5 pt). HINT: Deal with missing
    values with `use="pairwise.complete.obs"` option when computing
    `cor` matrix; use `pheatmap` for visualization.

<!-- end list -->

``` r
# YOUR CODE HERE
```

ADD YOUR INTERPRETATION.

#### F. Validate data preprocessing steps (1 pt).

How can you check whether we have successfully matched
`dbgap_rnaseq_sample` in `meta.matched.dt` with the `colnames` of
`matched.count.dt`?

``` r
# YOUR CODE HERE
```

Add your comment.

#### G. Build a wrapper class object (1 pt).

Finally, build a `DGEList` that will be used across this project. Hint:
use `as.matrix` and/or `as.data.frame` if needed. The first four columns
in the count data matrix contain gene information. Think about why we
matched the rows of `matched.meta.df` with the columns of
`matched.count.df` data matrix.

``` r
# YOUR CODE HERE
```

### Question 2: Gene-level quality control (5 POINTS)

Should we keep all the genes for downstream analysis? It is often
recommended to remove genes that contain very little information. But
how do we know which genes can be safely removed or not?

#### A. Compute gene-level [coefficient of variation (CV)](https://en.wikipedia.org/wiki/Coefficient_of_variation) values in CPM (1 pt).

  - Assign “0” CV values to zero mean/variance genes.

  - Hint: `cpm(dge)` is a matrix (gene by sample).

  - Compute the mean
    ![\\mu](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%5Cmu
    "\\mu") and standard deviation
    ![\\sigma](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%5Csigma
    "\\sigma") for each row (0.5 pt):

<!-- end list -->

``` r
# YOUR CODE HERE
```

  - Compute the coefficient of variation of genes (0.5 pt).

<!-- end list -->

``` r
# YOUR CODE HERE
```

#### B. Explore gene-level statistics (1 pt).

Considering that CPM values below one
(![\< 1](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%3C%201
"\< 1")) are technically missing observations, compute the frequency of
missing observations for each gene and investigate its relationship with
the *`log1p`-transformed* coefficient of variation (CV) and the
*`log1p`-transformed* mean values graphically.

  - Compute the missing frequency, and show its relationship with CV and
    ![\\mu](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%5Cmu
    "\\mu") (0.5 pt).

<!-- end list -->

``` r
# YOUR CODE HERE
```

  - Why is it desired to compare the missing frequency scores against
    the gene-level CV values rather than variance (0.5 pt)?

ADD YOUR INTERPRETATION.

#### C. Justify why we choose some cutoff value (1 pt)

On top of each plot, draw vertical lines on the same plot at 75%; and
discuss about why retaining genes with 25% observation rate could make
sense to you.

``` r
# YOUR CODE HERE
```

#### D. Determine a data-driven decision rule (1 pt).

Can we come up with a similar decision rule in a data-driven way?
Construct a simple feature matrix only including the missing fraction
values (gene x 1). Perform
[`kmeans`](https://en.wikipedia.org/wiki/K-means_clustering) clustering
with three clusters and 100 random restarts, and show how `0.75` cutoff
aligns with the clustering results.

  - Hint: Colour the points based on the cluster membership.

<!-- end list -->

``` r
## Fix the random seed to make sure that we get the same results
set.seed(1)
```

  - Perform `kmeans` clustering (0.5 pt):

<!-- end list -->

``` r
# YOUR CODE HERE
```

  - Demonstrate the clustering results in the same type of plots (0.5
    pt).

<!-- end list -->

``` r
# YOUR CODE HERE
```

ADD YOUR THOUGHTS.

#### E. Removing less-informative genes (1 pt)

Remove lowly expressed genes by retaining genes that have CPM
![\>](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%3E
"\>") 1 in at least 25% of samples and report how many genes are left
after filter.

  - Apply the filtering step (0.5 pt):

<!-- end list -->

``` r
# YOUR CODE HERE
```

  - Report the remaining number of genes (0.5 pt):

<!-- end list -->

``` r
# YOUR CODE HERE
```

### Question 3: Assessing overall distributions (5 POINTS)

#### A. TMM normalization (1 pt).

The expression values are raw counts.

  - Calculate TMM normalization factors (and add them to your DGEList
    object) (0.5 pt).

<!-- end list -->

``` r
# YOUR CODE HERE
```

  - Show the distribution (density, histogram, or both) of library sizes
    (0.5 pt).

<!-- end list -->

``` r
# YOUR CODE HERE
```

#### B. Visualizing sample-specific distributions (1 pt).

Examine the distribution of gene expression on the scale of log2 CPM on
the first 20 samples using box plots (with samples on the x-axis and
expression on the y-axis).

  - Hint 1: Add a small pseudo count of 1 before taking the log2
    transformation to avoid taking log of zero - you can do this by
    setting log = TRUE and prior.count = 1 in the cpm function.

  - Hint 2: To get the data in a format amenable to plotting with
    ggplot, some data manipulation steps are required. Take a look at
    the pivot\_longer function in the tidyr package to first get the
    data in tidy format (with one row per gene and sample combination).

<!-- end list -->

``` r
# YOUR CODE HERE
```

#### C. Compare distributions across samples (1 pt).

Examine the distribution of gene expression in units of log2 CPM across
all samples using overlapping density plots (with expression on the
x-axis and density on the y-axis; with one line per sample and lines
coloured by sample).

``` r
# YOUR CODE HERE
```

#### D. Compare distributions across samples and gene types (1 pt).

Let’s compare the distributions **visually** while restricting the
analysis to specific gene types. We investigate the first 20 samples and
partition genes into two groups, protein-coding and non-protein-coding,
for simplicity. What is your interpretation?

``` r
# YOUR CODE HERE
```

ADD YOUR THOUGHTS.

#### E. Focus on protein-coding genes (1 pt)

How many protein-coding genes are left after filtering? Can you create a
table that counts each type of genes after the filtering (0.5 pt)?

``` r
# YOUR CODE HERE
```

Since protein-coding genes constitute a majority of features and are
highly expressed, let’s take yet another subset of data by only
retaining protein-coding genes (0.5 pt).

``` r
# YOUR CODE HERE
```

### Question 4: Exploratory data analysis (EDA) of the samples (5 POINTS)

#### A. How do the samples correlate with one another (2 pt)?

Examine the correlation between samples using a heatmap with samples on
the x-axis and the y-axis and colour indicating a correlation between
each pair of samples. Again, use the log2 transformed CPM values.
Display other variables, such as “cohort,” “consensus\_sex,” “vital
status,” “overall survival,” “platelet count,” and “wbcCount” along with
the correlation matrix.

  - HINT: You may suppress sample names in `pheatmap`
    (`show_rownames=F`, `show_colnames=F`).

<!-- end list -->

``` r
# YOUR CODE HERE
```

#### B. Display the observed cell type fractions (1 pt)

Show a different set of annotation variables in the same correlation
heatmap. Display the observed cell type fractions (variables ending with
“in.PB”). HINT: use `dplyr::ends_with` for your convenience.

``` r
# YOUR CODE HERE
```

#### C. Display the estimated cell type fractions (1 pt)

Add the estimated cell type scores (variables ending with “like”) in the
same heatmap. For better visualization, apply [sigmoid
transformation](https://en.wikipedia.org/wiki/Sigmoid_function) on your
annotation matrix. A sigmoid function converts logit into probabilistic
scores.

``` r
# YOUR CODE HERE
```

#### D. Interpretation (1 pt)

Provide your interpretation of these results.

ADD YOUR THOUGHTS.

### Question 5: Find cell-type-specific genes (5 POINTS)

#### A. Set up differential gene expression analysis (1 pt)

First, set up a model matrix with cell type estimation scores (variables
ending with “like”) as covariates. Then calculate variance weights with
`voom`, and generate the mean-variance trend plot (1 pt). HINT: Filter
out samples with “NA” values in your design matrix. Include all the cell
type scores in your design matrix.

``` r
# YOUR CODE HERE
```

#### B. Fit linear models (1 pt)

Use `limma` (`lmFit` and `eBayes`) to fit the linear model with the
model matrix you just created.

``` r
# YOUR CODE HERE
```

#### C. Top genes (1 pt)

Print the 10 top-ranked genes by adjusted p-value for the “GMP.like”
coefficient using topTable.

``` r
# YOUR CODE HERE
```

#### D. Quantify the number of genes differentially expressed (1 pt)

Using the linear model defined above, determine the number of genes
differentially expressed in different cell types at a FWER (use
`adjust.method = "holm"` in topTable) less than 0.01. Count the number
of DEGs for each cell type and print a table.

  - HINT: Count how many significantly associated genes for each
    coefficient. Use `dplyr::count` with `dplyr::group_by`.

<!-- end list -->

``` r
# YOUR CODE HERE
```

#### E. Interpretation of the DEG results (1 pt)

As explored in the previous question (Q.4), many genes fluctuate with
cell type scores. It is not surprising to see many genes are strongly
associated with more than one cell types.

  - Count how many cell types each gene is significantly associated with
    (under the same FWER cutoff). Show your results by drawing histogram
    (0.5 pt). HINT: use `group_by(stable_id)`.

<!-- end list -->

``` r
# YOUR CODE HERE
```

  - Can you further break down the number of genes while distinguishing
    the sign of `logFC` (0.5 pt)?

<!-- end list -->

``` r
# YOUR CODE HERE
```

### Question 6: Understand cell-type-specific cancer mechanisms (5 POINTS)

#### A. Differential expression analysis for good and poor outcomes (2 pt)

Of the deceased patients (`vitalStatus == "Dead"`), we may define two
patient groups according to their survival time, using the median value
as a threshold value. More precisely, we have

  - good prognosis: a patient deceased with the survival time greater
    than or equal to the median;

  - poor prognosis: a patient deceased with the survival time less than
    the median.

First, create a new `dge` by subsetting on the deceased individuals (0.5
pt).

``` r
# YOUR CODE HERE
```

How many samples do have in each prognosis group (0.5 pt)?

``` r
# YOUR CODE HERE
```

Set up the model matrix with cell type estimation scores and the new
prognosis variable. Then calculate variance weights with voom, and
generate the mean-variance trend plot (1 pt). HINT: Filter out samples
with “NA” values in your design matrix. Make sure that the prognosis
variable is of factors.

``` r
# YOUR CODE HERE
```

Use `limma` (`lmFit` and `eBayes`) to fit the linear model with the
model matrix you just created. Print the 10 top-ranked genes by adjusted
p-value for the “progpos” coefficient using topTable (0.5 pt).

``` r
# YOUR CODE HERE
```

How many genes are differentially expressed between the poor and good
prognosis groups at FDR cutoff 10% (0.5 pt)?

``` r
# YOUR CODE HERE
```

#### B. Identify cell-type-specific differentially expressed genes (2 pt)

Build a new model matrix that includes all the marginal
cell-type-scores, and interaction terms between the prognosis and the
cell type scores; as usual, do weight adjustment via `voom` (1 pt).

``` r
# YOUR CODE HERE
```

Use `limma` (`lmFit` and `eBayes`) to fit the linear model with the
model matrix you just created. Print the 10 top-ranked genes by adjusted
p-value for the `Progenitor.like:progpoor` using topTable (0.5 pt).

``` r
# YOUR CODE HERE
```

Report how many interaction terms are significantly associated at FDR
10%. Break down these numbers into cell types (0.5 pt).

``` r
# YOUR CODE HERE
```

#### C. Visualize top interaction DEGs (1 POINT)

Show condition-specific scatter plots for these top genes (x-axis: the
cell type scores; y-axis: gene expression). Show all the significant
genes and the relevant cell types. HINT: Create a long-form data
frame/tibble after subsetting top genes.

``` r
# YOUR CODE HERE
```
