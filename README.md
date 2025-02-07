# R Package - CellTagR

## Important Notice
We recently found that inside the setter function, the column names of the filtered count matrix are possibly shuffled around during the second round of filtering, thus some CellTags were associated with the wrong cell barcodes. This could lead to inaccurate clone-calling. We suggest users reinstall the package and empty the slot with the following line of code and restart the pipeline from this step: https://github.com/morris-lab/CellTagR#6-additional-filtering.
```r
celltag.obj@metric.filtered.count <- as(matrix(NA, 0, 0), "dgCMatrix")
```

## Description
This is a wrapped R package of the workflow (https://github.com/morris-lab/CellTagWorkflow) with additional assessment of the complexity of the Celltag Library sequences. Additionally, previous version of this package can be found https://github.com/morris-lab/PreviousCloneHunter. ***Note: This has been changed and improved. Analysis with previous version will not be compatible.*** This package have a dependency on R version (R >= 3.5.0). This can be used as an alternative approach for this pipeline. For details regarding development and usage of CellTag, please refer to the following papaer - *Biddy et. al. Nature, 2018*, https://www.nature.com/articles/s41586-018-0744-4, *Kong et al., Nature Protocol, 2020*, https://www.nature.com/articles/s41596-019-0247-2

Install devtools
```r
install.packages("devtools")
```
Install the package from GitHub.
```r
library("devtools")
devtools::install_github("morris-lab/CellTagR")
```
Load the package
```r
library("CellTagR")
```

## Assessment of CellTag Library Complexity via Sequencing
In this first section, we evaluate the CellTag library complexity using sequencing. Following is an example using the sequencing data we generated in lab for pooled CellTag library V2. 
### 1. Read in the fastq sequencing data and extract the CellTags
The extracted CellTags will be stored as an attribute (fastq.full.celltag & fastq.only.celltag) in the resulting object.
```r
# Read in the data file that come with the package
fpath <- system.file("extdata", "V2-1_R1.zip", package = "CellTagR")
extract.dir <- "."
# Extract the dataset
unzip(fpath, overwrite = FALSE, exdir = ".")
full.fpath <- paste0(extract.dir, "/", "V2-1_S2_L001_R1_001.fastq")
# Set up the CellTag Object
test.obj <- CellTagObject(object.name = "v2.whitelist.test", fastq.bam.directory = full.fpath)
# Extract the CellTags
test.obj <- CellTagExtraction(celltag.obj = test.obj, celltag.version = "v2")
```

### 2. Count the CellTags and sort based on the occurrence of each CellTag
```r
# Count and Sort the CellTags in descending order of occurrence
test.obj <- AddCellTagFreqSort(test.obj)
# Check the stats
test.obj@celltag.freq.stats
```

### 3. Generation of a whitelist for the CellTag library
Here, we generating the whitelist for this CellTag library - CellTag V2. This will remove the CellTags with an occurrence number below the threshold. The threshold (using 90th percentile as an example) is determined: floor[(90th quantile)/10]. The percentile can be changed while calling the function. A plot of CellTag reads will be plotted and it can be used to further choose the percentile. If the output directory is offered, whitelist files will be stored in the provided directory. Otherwise, whitelist files will be saved under the same directory as the fastq files with name as <CellTag Version Number>_whitelist.csv (Example: v2_whitelist.csv). 

```r
# Generate the whitelist
test.obj <- CellTagWhitelistFiltering(celltag.obj = test.obj, percentile = 0.9, output.dir = NULL)
```
The generated whitelist for each library can be used to filter and clean the single-cell CellTag UMI matrices.

## Single-Cell CellTag Extraction and Quantification
In this section, we are presenting an alternative approach that utilizes this package to carry out CellTag extraction, quantification, and generation of UMI count matrices. This can be also accomplished via the workflow supplied - https://github.com/morris-lab/CellTagWorkflow. 
#### Note: Using the package could be slow for the extraction part. For reference, it took approximately an hour to extract from a 40Gb BAM file using a maximum of 8Gb of memory.

### 1. Download the BAM file 
Here we follow the same step as in https://github.com/morris-lab/CellTagWorkflow to download the a BAM file from the Sequence Read Archive (SRA) server. Again, this file is quite large. Hence, it might take a while to download. The file can be downloaded using wget in terminal as well as in R.
```r
# bash
wget https://sra-pub-src-1.s3.amazonaws.com/SRR7347033/hf1.d15.possorted_genome_bam.bam.1
```
OR
```r
download.file("https://sra-pub-src-1.s3.amazonaws.com/SRR7347033/hf1.d15.possorted_genome_bam.bam.1", "./hf1.d15.bam")
```

### (RECOMMENDED) Optional Step: BAM File Filtering
***NOTE:*** If BAM file filtering is **NOT** required (although, we strongly recommend this), skip this step and move to *Step 2 - Create a CellTag Object*, in which the entire BAM file will be used. Otherwise, before generating a CellTag object and extracting the CellTags, we will carry out the following BAM filtering step, from which a subset of reads in the BAM file will be searched during CellTag extraction.

In this step, we will filter the BAM file to reduce the possibility that false positive CellTags will be identified. Briefly, the 17-20 bp sequence that comprises the CellTag barcode may appear by chance in other regions of the transcriptome. These may be identified as CellTags and cells expressing these transcripts may be falsely called as clones. By filtering reads in the BAM file to only include those which are unmapped as well as those mapped to GFP or (optionally) the CellTag UTR, we reduce the chances of extracting false positive CellTags.

We recommend adding the CellTag UTR as a transgene to the reference used during alignment. This sequence is stored in https://github.com/morris-lab/CellTagR/Exmples/CellTag_UTR.fa. More information on adding a marker gene to a reference can be found here: https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/latest/using/tutorial_mr.

### I. Filter unmapped reads
First, we will use samtools to efficiently filter umapped reads.

```r
# bash
samtools view -b -f 4 ./hf1.d15.bam > ./hf1.d15.filtered.bam
```

### II. Filter transgene reads
Next, we will filter reads aligned to GFP or the CellTag UTR.

```r
# bash
samtools view -b  ./hf1.d15.bam GFP >> ./hf1.d15.filtered.bam
```

If the CellTag UTR was not included in the reference, the following line may be omitted.

```r
# bash
samtools view -b ./hf1.d15.bam CellTag.UTR >> ./hf1.d15.filtered.bam
```

### 2. Create a CellTag Object
In this step, we will initialize a CellTag object with a object name and the path to where the bam file is stored **if only one bam file is processed.**

```r
# Set up the CellTag Object
bam.test.obj <- CellTagObject(object.name = "bam.cell.tag.obj", fastq.bam.directory = "./hf1.d15.filtered.bam")
```

**Update: CellTagR now enables read-in of multiple BAM files at a time.** When multiple BAM files need to be processed, ***please use a folder that contains ONLY BAM files and put the fastq.bam.directory as the path of the folder.*** For instance, two bam files need to be processed named as *bam1.bam* and *bam2.bam*. They will be put into a folder named as *beautiful_bams* in the *Desktop*. Then, the input will be *fastq.bam.directory="~/Desktop/beautiful_bams/"* as below.

```r
## NOT RUN
# Set up the CellTag Object
# bam.test.obj <- CellTagObject(object.name = "bam.cell.tag.obj", fastq.bam.directory = "~/Desktop/beautiful_bams/")
```

***Note: The following tutorials are only intended for processing ONE CellTag version. To obtain information for all three versions of CellTags, running the following pipeline is required for each CellTag version independently, i.e. finishing process for V1 and then repeating the procedure for V2, and so on. After running the pipeline for each CellTag version, the clonal information of each will be stored in the same object, which can be used to carry out network construction and visualization.***

### 3. Extract the CellTags from the BAM file
In this step, we will extract the CellTag information from the BAM file, which contains information including cell barcodes, CellTag and Unique Molecular Identifiers (UMI). The result generated from this extraction will be a data table containing the following information. The result will then be saved into the slot "bam.parse.rslt" in the object in the following format.

|Cell Barcode|Unique Molecular Identifier|CellTag Motif|
|:----------:|:-:|:---------:|
|Cell.BC|UMI|Cell.Tag|
```r
# Extract the CellTag information
bam.test.obj <- CellTagExtraction(bam.test.obj, celltag.version = "v1")
# Check the bam file result
head(bam.test.obj@bam.parse.rslt[["v1"]])
```
**Update: CellTagR now enables read-in of multiple BAM files at a time.** Extraction with multiple samples will automatically add prefixes to different samples in the order of BAM file given, i.e. Sample-\<i\>_\<Cell Barcode\>. The order of BAM file processing will be printed as it processes along. Prefixes assignments from users will be coming soon!

### 4. Quantify the CellTag UMI Counts and Generate UMI Count Matrices
In this step, we will quantify the CellTag UMI counts and generate the UMI count matrices. This function will take in two inputs, including the barcode tsv file generated by 10X and celltag object processed from Step 2. The barcode tsv file can be either filtered or raw. **However, note that using the raw barcodes file could require a large amount of memory for using this function**. If filtered barcode files are used, **only cell barcodes that appear in the filtered barcode file** will be preserved. The result will also be saved as a *dgCMatrix* in a slot - "raw.count" - under the object. At the same time, initial CellTag statistics will be saved as another slot under the object. The matrix will be in the format as following. ***If multiple BAM files, please follow the updated.***

||CellTag Motif 1|CellTag Motif 2|\<all tags detected\>|CellTag Motif N|
|:----------:|:-:|:---------:|:--:|:--:|
|Cell.BC|Motif 1|Motif 2|\<all tags detected\>|Motif N|

```r
# Generate the sparse count matrix
bam.test.obj <- CellTagMatrixCount(celltag.obj = bam.test.obj, barcodes.file = "./barcodes.tsv")
# Check the dimension of the raw count matrix
dim(bam.test.obj@raw.count)
```

**Update: CellTagR now enables read in of multiple BAM files at a time.** An aggregated barcode file needs to be generated for multiple BAM file processed with proper prefixes. Please use the *Barcode.Aggregate* function to generate a aggregated barcode file. This function takes in an **ordered** list of barcodes files. The order should be the same as the BAM file order.

```r
Barcode.Aggregate(list("barcode_1.tsv", "barcode_2.tsv"), "./barcodes_all.tsv")
# Generate the sparse count matrix
bam.test.obj <- CellTagMatrixCount(celltag.obj = bam.test.obj, barcodes.file = "./barcodes_all.tsv")
# Check the dimension of the raw count matrix
dim(bam.test.obj@raw.count)
```

The generated CellTag UMI count matrices can then be used in the following steps for clone identification.

## Single-cell CellTag UMI Count Matrix Processing
In this section, we are presenting an alternative approach that utilizes this package we established to carry out clone calling with single-cell CellTag UMI count matrices. In this pipeline below, we are using a subset of dataset generated from the full data (Full data can be found here: https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE99915). Briefly, in our lab, we reprogram mouse embryonic fibroblasts (MEFs) to induced endoderm progenitors (iEPs). This dataset is a single-cell dataset that contains cells collected from different time points during the process. This subset is a part of the first replicate of the data. It contains cells collected at Day 15 with three different CellTag libraries - V1, V2 & V3. 

### 1. Read in the single-cell CellTag UMI count matrix
We generated this object from the above steps, using BAM files. As above, BAM files take a long time to process. Hence, in this repository, we include a sample object saved as .Rds file from the previous steps, in which raw count matrix is included in the slot - "raw.count"
```r
# Read the RDS file and get the object
dt.mtx.path <- system.file("extdata", "Demo_V1.Rds", package = "CellTagR")
bam.test.obj <- readRDS(dt.mtx.path)
```

### (RECOMMENDED) Optional Step: CellTag Error Correction
***NOTE:*** If CellTag error correction is **NOT** required (although, we strongly recommend this), skip this step and move to *Step 2 - binarization*, in which the raw matrix will be used. Otherwise, before binarization and additional filtering, we will carry out the following error correction step via Starcode, from which a collapsed matrix will be used further for binarization.

In this step, we will identify CellTags with similar sequences and collapse similar CellTags to the centroid CellTag. For more information and installation, please refer to starcode software - https://github.com/gui11aume/starcode. Briefly, starcode clusters DNA sequences based on the Levenshtein distances between each pair of sequences, from which we collapse similar CellTag sequences to correct for potential errors occurred during single-cell RNA-sequencing process. Default maximum distance from starcode was used to cluster the CellTags.

### I. Prepare for the data to be collapsed
First, we will prepare the data to the format that is accepted by starcode. This function accepts two inputs including the CellTag object with raw count matrix generated and a path to where to save the output text file. The output will be a text file with each line containing one sequence to collapse with others. In this function, we concatenate the CellTag with cell barcode and use the combined sequences as input to execute Starcode. The file to be used for Starcode will be stored under the provided directory.
```r
# Generating the collapsing file
bam.test.obj <- CellTagDataForCollapsing(celltag.obj = bam.test.obj, output.file = "~/Desktop/collapsing.txt")
```

**Update: CellTagR now enables read-in of multiple BAM files at a time.** Multiple files with their prefixes used before will be incorporated into the output files. Hence multiple files will be generated for collapsing in the given directory. For instance, if there are 2 samples and to be saved on Desktop, they will be named as *collapsing_Sample-1.txt* and *collapsing_Sample-2.txt*.

### II. Run Starcode to cluster CellTags
Following the instruction for Starcode, we will run the following command to generate the result from starcode. **Make sure to run each file generated for each sample if multiple are processed**

```r
./starcode -s --print-clusters ~/Desktop/collapsing.txt > ~/Desktop/collapsing_result.txt
```

***Please use a folder containing ONLY the collapsing results! And please name the collapsing results corresponding to their sample names if multiple samples are processed.*** For example, use the name *collapsing_result_Sample-1.txt* for *collapsing_Sample-1.txt*. 

### III. Extract information from Starcode result and collapse similar CellTags
With the collapsed results, we will regenerate the CellTag x Cell Barcode matrix. The collpased matrix will be stored in a slot - "collapsed.count" - in the CellTag object. This function takes two inputs including the CellTag Object to modify and the path to th result file from collapsing. ***If multiple BAM files generated the collapsing result, check the update***

```r
# Recount and generate collapsed matrix
bam.test.obj <- CellTagDataPostCollapsing(celltag.obj = bam.test.obj, collapsed.rslt.file = "~/Desktop/collapsing_rslt.txt")
# Check the dimension of this collapsed count.
head(bam.test.obj@collapsed.count)
```

***Update: If with multiple BAM file generated collapsing result, run the following lines*** Example: the result files are saved on the desktop in the folder named *star_collapse*.

```r
collapsed.rslt.dir <- "~/Desktop/star_collapse"
# Recount and generate collapsed matrix
bam.test.obj <- CellTagDataPostCollapsing(celltag.obj = bam.test.obj, collapsed.rslt.file = list.files(collapsed.rslt.dir, full.names = T))
# Check the dimension of this collapsed count.
head(bam.test.obj@collapsed.count)
```

Below is an example Jaccard Analysis result with Error Correction using Starcode collapsing (top - without collapsing, bottom - with collapsing):
<p align="center">
    <img src="/Exmples/jaccard wo collapsing.png" height="480" width="720">
</p>

<p align="center">
    <img src="/Exmples/jaccard example.png" height="480" width="720">
</p>

### 2. Binarize the single-cell CellTag UMI count matrix
Here, we binarize the count matrix to contain 0 or 1, where 0 indicates no such CellTag found in a single cell and 1 reports CellTag expression. The suggested cutoff that marks presence or absence is at least 2 counts per CellTag per Cell. For details regarding cutoff choice, please refer to the paper - https://www.nature.com/articles/s41586-018-0744-4. The binary matrix will be stored in a slot - 'binary.mtx' - as a *dgCMatrix*. **Note: If collapsing was performed, binarization will be based on the collapsed count matrix. Otherwise, it will be based on the raw count matrix**
```r
# Calling binarization
bam.test.obj <- SingleCellDataBinatization(bam.test.obj, 2)
```

### 3. Metric plots to facilitate for additional filtering
We then generate scatter plots for the number of total celltag counts in each cell and the number each CellTag across all cells. These plots assist filtering and cleaning of the data.
```r
MetricPlots(bam.test.obj)
```
Below is an example plot that you could obtain from this object
<p align="center">
  <img src="/Exmples/pre_filtering.png" height="720" width="720">
</p>

### 4. Apply the whitelisted CellTags generated from assessment
Based on the whitelist generated earlier, we filter the UMI count matrix to contain only whitelisted CelTags for the current version under processing. The function takes in two inputs including the CellTag object with binarization performed and the path to the whitelist csv file. The whitelist result will be saved in a slot - "whitelisted.count".
```r
# Read the RDS file and get the object
dt.mtx.whitelist.path <- system.file("extdata", "v1_whitelist.csv", package = "CellTagR")
bam.test.obj <- SingleCellDataWhitelist(bam.test.obj, dt.mtx.whitelist.path)
```

### 5. Check metric plots after whitelist filtering
Recheck the metric similar to Step 3
```r
MetricPlots(bam.test.obj)
```

### 6. Additional filtering
#### Filter out cells with more than 20 CellTags
```r
bam.test.obj <- MetricBasedFiltering(bam.test.obj, 20, comparison = "less")
```
#### Filter out cells with less than 2 CellTags
```r
bam.test.obj <- MetricBasedFiltering(bam.test.obj, 2, comparison = "greater")
```
### 7. Last check of metric plots
```r
MetricPlots(bam.test.obj)
```
Example plot of last check!
<p align="center">
  <img src="/Exmples/post_filtering.png" height="720" width="720">
</p>
If it looks good, proceed to the following steps to call the clones.

### 8. Clone Calling
#### I. Jaccard Analysis
This calculates pairwise Jaccard similarities among cells using the filtered CellTag UMI count matrix. This function takes the CellTag object with metric filtering carried out. This will generate a Jaccard similarity matrix, which is saved as a part of the object in a slot - "jaccard.mtx". It also plots a correlation heatmap with cells ordered by hierarchical clustering. 

```r
bam.test.obj <- JaccardAnalysis(bam.test.obj)
```
##### Note: For large sparse matrix, a fast version can be chosen using the parameter *fast*.
```r
bam.test.obj <- JaccardAnalysis(bam.test.obj, fast = T)
```
#### II. Clone Calling
Based on the Jaccard similarity matrix, we can call clones of cells. A clone will be selected if the correlations inside of the clones passes the cutoff given (here, 0.7 is used. It can be changed based on the heatmap/correlation matrix generated above). Using this part, a list containing the clonal identities of all cells and the count information for each clone will be stored in the object in slots - "clone.composition" and "clone.size.info". 

##### Clonal Identity Table `clone.composition`

|clone.id|cell.barcode|
|:-------:|:------:|
|Clonal ID|Cell BC |

##### Count Table `clone.size.info`
|Clone.ID|Frequency|
|:------:|:-------:|
|Clonal ID|The cell number in the clone|

```r
# Call clones
bam.test.obj <- CloneCalling(celltag.obj = bam.test.obj, correlation.cutoff=0.7)
# Check them out!!
bam.test.obj@clone.composition[["v1"]]
bam.test.obj@clone.size.info[["v1"]]
```

## Network Construction And Visualization
Having all three CellTag version analyzed and stored in one CellTag object, we will construct network of each individual clone connecting to its descendents. As well as connections between clones, cells in each clone will be visualized on the network as leaf nodes. In the network, each center node denotes a clone. Connections between those nodes suggest a "parent-child" relationship between the clones. Each leaf node denotes a cell. Connections between leaf nodes and center nodes suggest a "belonging" relationship. Additionally, we allow users to further construct a stacked bar chart to facilitate further analysis of the dynamics of different timepoints. 

***Note:*** Here, we provide a demo object in .Rds format that is generated with all three versions processed. The R notebook used to process all three versions are included in the Examples folder.

### 1. Read in the object
```r
# Read the RDS file and get the object
dt.mtx.path <- system.file("extdata", "bam_v123_obj.Rds", package = "CellTagR")
bam.test.obj <- readRDS(dt.mtx.path)
```

### 2. Calculate the link list
Here, we convert the CellTag Matrix into a form of link list, which will be further used to construct the linkages in the network
```r
bam.test.obj <- convertCellTagMatrix2LinkList(bam.test.obj)
```
The linked list is saved in the slot - "network.link.list", in the following format.

|source|target|tag|target_unmodified|
|:-------:|:------:|:------:|:------:|
|The Source Node|The Target Node|Associated CellTag|Original Target Name|

In the source node, the data is formatted as \<CellTag Version\>_\<Clone Number\>. These are the centroid nodes for the network. The clone number can be found in the previously filled slot - "clone.composition". In the target node, there are two possibilities. One of possible targets are cells that belong to the centroid clone. The others are clones that are related to the centroid clone, which will suggest "parent-child" relationship between clones. For example, in the table below, the first row describes the belonging relationship of cell with barcode "AAGCCGCAGCTAGCCC-1" to Clone3 from CellTag V1, while the second row indicates a "parent-child" relationship between Clone 1 from CellTag V1 and Clone 42 from CellTag V2. 

|source|target|tag|target_unmodified|
|:-------:|:------:|:------:|:------:|
|CellTagV1_3|AAGCCGCAGCTAGCCC-1_V1|CellTagV1|AAGCCGCAGCTAGCCC-1|
|CellTagV1_1|CellTagV2_42|CellTagV1|CellTagV2_42|

### 3. Get nodes from the link list
This will obtain all the nodes that are involved in this network.
```r
bam.test.obj <- getNodesfromLinkList(bam.test.obj)
```

### 4. Add additional information
For each leaf node (each cell), other information, such as cluster/cell types, can be available via other analysis. In this step, we will add these information into each node such that these information can be visualized on the network as well. In this scenario, for demo purposes, we used a simulation data frame to serve as a mock cluster information for each node.
```r
# Simulate some additional data
additional_data <- data.frame(sample(1:10, size = length(rownames(bam.test.obj@celltag.aggr.final)), replace = TRUE), row.names = rownames(bam.test.obj@celltag.aggr.final))
colnames(additional_data) <- "Cluster"
# Add the data to the object
bam.test.obj <- addData2Nodes(bam.test.obj, additional_data)
```

### 5. Network visualization and plot
Here, we will visualize the network!
```r
# Network Visualization
bam.test.obj <- drawSubnet(tag = "CellTagV1_2", overlay = "Cluster", celltag.obj = bam.test.obj)
bam.test.obj@network
```

Additionally, the network can be saved to a html file, allowing better visualization and overview. Please make sure to have pandoc to support markdown and output this network.
```r
saveNetwork(bam.test.obj@network, "~/Desktop/presentation/Demo/hf1.d15.network.construction.html")
```

### 6. Stack bar chart generation
An important aspect of using CellTagging is to analyze the clonal dynamics of a population of cells. Here, we provide a stack bar chart option to provide some insights.
```r
# Get the data for ploting
bar.data <- bam.test.obj@celltag.aggr.final
bar.data$Cell.BC <- rownames(bar.data)

bar.data <- gather(bar.data, key = "CellTag", value = "Clone", 1:3, na.rm = FALSE)

# Using ggplot to plot
ggplot(data = bar.data) + 
  geom_bar(mapping = aes(x = CellTag, fill = factor(Clone)), position = "fill", show.legend = FALSE) + 
  scale_y_continuous(labels = scales::percent_format()) +
  theme_bw()
```
Below is a sample bar chart!
<p align="center">
  <img src="/Exmples/bar_Chart.png" height="540" width="720">
</p>

## Contact Us
