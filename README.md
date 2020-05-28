# col11a1-wes-pipeline
Pipeline used to call somatic variants for "Mutant collagen COL11A1 enhances invasive cancer" by Lee et. al. 2020. 

## Summary
### Preprocessing
Reads were aligned to the hg38 reference genome using BWA-MEM and marked for duplicates using GATK MarkDuplicates.
Reads then underwent base quality score recalibration using GATK BaseRecalibrator. Hg38 reference files were downloaded
from the Broad's GATK Resource Bundle.

### Somatic Variant Calling
Normal samples were used to create a Panel of Normals (PON) using GATK Mutect2 in tumor only mode and GATK CreateSomaticPanelOfNormals. Variants were then called using Mutect2 in tumor matched normal mode. Variants
called by Mutect2 were filtered by GATK FilterMutectCalls to avoid false positives. Variants that passed filtering
were annotated using VEP and VCF2MAF. 

The pipeline aggregates mutations from each SCC and saves them to `variants.maf`.
`variants.maf` was renamed `lee.maf` for downstream analysis to avoid confusion with the other callsets
(Pickering and Durinck).

## Prerequisites
Before you start setting up the pipeline, make sure you have your reference genome assembled.
The GRCh38 (hg38) genome is available on the Broad's
GATK [website](https://software.broadinstitute.org/gatk/download/bundle).

You'll also need the `cache` files for 
[Variant Annotation Predictor (VEP)](https://github.com/Ensembl/ensembl-vep).
Follow the tutorial 
[here](https://uswest.ensembl.org/info/docs/tools/vep/script/vep_cache.html#cache) 
to download the data files. Install version 99.
Don't forget to index the files before running the pipeline.

## Usage
After finishing the setup and enabling the `conda` environment, inside the analysis directory with
`Snakefile` do a dry run to check for errors
```
snakemake -n
```
Once you're ready to run the analysis navigate to the base directory with `Snakefile` and type
```
snakemake --use-conda --use-singularity
```
If your machine has multiple cores, you can use these cores with
```
snakemake -j [cores]
```
This will run multiple rules simultaneously, speeding up the analysis.

The pipeline produces two key files: `mafs/variants.maf` and `qc/multiqc_report.html`.
`variants.maf` includes somatic variants from all samples that passed Mutect2 filtering.
They have been annotated with VEP and a single effect has been chosen by [VCF2MAF](https://github.com/mskcc/vcf2maf)
using the Ensembl database. Ensembl uses its canonical isoforms for effect selection. Other isoforms
can be specified by modifying the `vcf2maf` rule.
`multiqc_report.html` includes quality metrics like coverage for the fully processed BAM files. 
Individual VCF files for each sample prior to VCF2MAF selection are named `{patient}.filtered.vep.vcf` in `vcfs/`.


### Cluster Execution
If you're using a compute cluster, you can take advantage of massively
parallel computation to speed up the analysis. Only SLURM clusters are
currently supported, but if you work with another cluster system (SGE etc)
`snakemake` makes it relatively easy to add cluster support.

Follow these instructions to enable SLURM usage
1. Edit the `out` field in `cluster.json` to tell SLURM where to save pipeline `stdout` and `stderr`.
2. Run `snakemake` from the command line with the following options
```
snakemake --cluster-config cluster.json -j 100 --cluster 'sbatch -p {cluster.partition} -t {cluster.time} --mem {cluster.mem}  -c {cluster.ncpus} -o {cluster.out}'
```
This command can be run in an `sbatch` job.
In the `sbatch` script, don't forget to `conda activate`
before running snakemake.

If for some reason you can't leave the master `snakemake` process running, `snakemake`
offers the ability to launch all jobs using `--immediate-submit`. This
approach will submit all jobs to the queue immediately and finish the master `snakemake`
process. The downside of this method is that temporary files are not supported, so
the results directories will be very cluttered. 
To use `--immediate-submit` follow these steps
1. Give `parseJobID.sh` permission to run
```
chmod +x parseJobID.sh
```
2. Submit the `snakemake` jobs
```
snakemake --cluster-config cluster.json --cluster 'sbatch $(./parseJobID.sh {dependencies}) -t {cluster.time} --mem {cluster.mem} -p {cluster.partition} -c {cluster.ncpus} - o {cluster.out}' --jobs 100 --notemp --immediate-submit
```

## Environments
`snakemake` is required to run `ngs-pipeline`, and other programs (`samtools`, `gatk`, etc)
are required for various steps in the pipeline. There are many ways to manage the required
executables.

### Singularity Container + Conda Environments
`snakemake` can run `ngs-pipeline` in a `singularity` container. Inside this container
each step is executed with a `conda` environment specified in `envs/`. This approach
controls the OS and individual packages, ensuring that certain software versions are
used for analysis. This approach can be enabled with the `--use-conda --use-singularity`
flags. **This approach is recommended because using .yaml files to specify the environment records the
software version used for each step, helping others reproduce your results.**

### Other
Although `conda` and `singularity` are recommended, as long as all the packages are installed
on your machine, the pipeline will run. You can also only use `conda` environments and
skip the `singularity` container with `--use-conda`, although this means the OS may be
different from other users.

## Test Dataset
A small sample of `chr21` reads are supplied from the 
[Texas Cancer Research Biobank](http://txcrb.org/index.html) for
end-to-end pipeline tests. 
This data is made available as open access data with minimal privacy
restrictions. Please read the [Conditions of Use](http://txcrb.org/data.html)
before using the data.

## Tips
### FASTQ Formatting
The first line of each read (called the sequence identifier) should consist of a single string not separated
by any spaces. An example of a properly sequence identifier is
```
@MGILLUMINA4_74:7:1101:10000:100149
```
Sequence identifiers that consist of multiple space separated words will cause problems because `gatk` captures the entire
string and uses it as the read ID but `bwa` only parses the first word as the read group when it writes aligned reads
to BAM files. This sequence identifier would break the pipeline
```
@MGILLUMINA4_74:7:1101:10000:100149     RG:Z:A470018/1
```
Check the sequence identifiers if you encounter a `Aligned record iterator is behind the unmapped reads` error.

## Citations
This pipeline is based on `dna-seq-gatk-variant-calling` by 
[Johannes Köster](https://github.com/snakemake-workflows/dna-seq-gatk-variant-calling).
See [here](https://github.com/tjbencomo/ngs-pipeline/blob/master/citations.md) 
for a list of software used in this pipeline.
