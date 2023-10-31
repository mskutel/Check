# DoubleA: <a href=""><img src="img/2a.svg" align="right" width="150" ></a>  <h3> Semi-manual assembly&annotation Snakemake-based pipeline </h3> 


Result of the pipeline: 
- assembly (nucleotide fasta file); 
- annotation (gff, gbk, tbl files)
## #1. Activate virtual environment (from my dir only)
```commandline
export PATH="$PWD/bin:$PATH"
eval "$(micromamba shell hook --shell bash)"
micromamba activate snakemake 
```
## #2. Check snakemake version
```commandline
snakemake --version
```
Should be 7.32.3.
## #3. Create a working directory and move the pipeline description there
```commandline
mkdir <dir-name>
cp doubleapipe.zip <dir-name>
cd <dir-name>
unzip doubleapipe.zip
```
The working directory should have the following folder structure:  
```
 <dir-name>  
   | workflow  
   |   | envs  
   |   |   | <files>.yml  
   |   | snake_<files>.Snakefile  
   | config.yaml  
   | example_samples.txt  
   | example_samples_nodes.tsv  
```
## #4. Configure workflow.
### 1. Look into `config.yaml`.
- Define the file with names of the samples (`samples`, see `example_samples.txt`).
Note: the reads should have the same prefix, as names in samples file.  
Example: 
```text
plasmid253.1

bananaphage

sample3
```
Should correspond to the following names of FASTQ files ("." before R1 and R2):
```text
plasmid253.1.R1.fg.gz
plasmid253.1.R2.fg.gz

bananaphage.R1.fg.gz
bananaphage.R2.fg.gz

sample3.R1.fg.gz
sample3.R2.fg.gz
```
- Define the folder where the raw reads placed (`raw_reads_dir`). 

- Define number of threads and max memory consumption (if not sure, leave it default).

- Define reads processing procedure (if not sure, leave it default).

### 2. Launch first part of Snakemake pipeline (assembly).
First, check that there is no errors:
```commandline
snakemake --use-conda --snakefile workflow/snake_assemble_circles.Snakefile -c <threads> -n
```
Run pipeline:
```commandline
snakemake --use-conda --snakefile workflow/snake_assemble_circles.Snakefile -c <threads> --report assembly_report.html
```

Inspect the files in `stats_dir` folder to explore reads quality and 
`.log` files in `assembly_dir/<sample>` to explore quality of assemblies.

### 3. Check assemblies in Bandage to select nodes
Inspect the `<files>.gfa` in `assembly_dir/<sample>` directories, using Bandage.  
Write the nodes of interest into file (see `example_sample_nodes.tsv`), and specify 
the file in `config.yaml` in `sample_nodes` section.
It's useful to make table of such kind:  

|   Sample    | fastp + seqtk |                    | assembly stats |              |                |               |                       |                         | Blast results |                     |         | PhageTerm |      |
|:-----------:|:-------------:|:------------------:|:--------------:|:------------:|:--------------:|:-------------:|:---------------------:|:-----------------------:|:-------------:|:-------------------:|:-------:|:---------:|:----:|
|             |  # reads, M   |        mode        |   # contigs    | total length | N50, if needed | Viral node(s) | Viral depth (mean), x | Viral contig length, bp |   Best Hit    | Percent Identity, % | Comment |  success  | type |
| SamplePhage |       1       | + dedup + sampling |       1        |   167,747    |    167,747     |       1       |         1.00          |         167,747         |  OL539458.1   |        96.76        |         |   True    |  p1  |

### 4. Run the second part of the pipeline (re-organization of the nodes).
First, check that there is no errors:
```commandline
snakemake --use-conda --snakefile workflow/snake_fix_assemblies.Snakefile -c <threads> -n
```
Run pipeline:
```commandline
snakemake --use-conda --snakefile workflow/snake_fix_assemblies.Snakefile -c <threads> --report  fix_report.html
```

Sometimes the PhageTerm can not re-organize the genome.
Inspect the files in `phageterm_output` directory (`.pdf`).

### #5. Run the last part of the pipeline.
It's time to run annotation. If you launch the part of the pipeline not the first time,
and the launch was recently, you can specify the absolute path to Pharokka db in the `config.yaml`, section `pharokka_db`.
Otherwise, the pipeline will download it into the working directory. Better to have up-to-date databases.  
First, check that there is no errors:
```commandline
snakemake --use-conda --snakefile workflow/snake_aggregate_and_annotate_new.Snakefile -c <threads> -n
```
Run pipeline:
```commandline
snakemake --use-conda --snakefile workflow/snake_aggregate_and_annotate_new.Snakefile -c <threads> --report annotation_report.html
```
Inspect `.png` files and `.gff` files in `annotation_dir`.

#### If there was an issue with PhageTerm, manually re-organize the fasta-file in `fixed_assemblies`, remove corresponding dir in `annotations_dir`  and `.png` file, then re-run this part of pipeline.

Inspect snakemake reports (`.html`) in working directory to see launch statistics.  
**Enjoy the results!**

## #5. After that all things are done
What should be deleted?
- decompressed reads files (without .gz extension).
- conda environments, located in folder `.snakemake/conda`
- conda metadata, located in folder `.snakemake/metadata`

### How to edit gff and gbk files?
To edit gff file, run `process_gff.py`. For example, the following script adds the gene entries to gff 
(useful for open in genome viewers), add label and removes the wrong annotation, specified at 
file `wrong_example.txt`  
```commandline
python scripts/process_gff.py -g -l -r wrong_example.txt \
        -i path/to/gff.gff -o path/to/changed_gff.gff   
```

To convert the obtained .gff file into .gbk, use the `bcbio-gff` ([can be installed with conda/mamba](https://anaconda.org/bioconda/bcbio-gff)):
```commandline
conda create -n bcbio_gff_env -c bioconda bcbio-gff
```

Run the following line:
```commandline
conda activate bcbio_gff_env
python scripts/gff_to_genbank.py path/to/changed_gff.gff
```


## #6. Output structure
    - nucleotide fasta file in `fixed_assemblies`;
    - image of genome organization in working directory (`{sample}.png`);
    - annotation (`.gbk`, `.gff` and `.tbl`) files, and proteome (`.faa`) files in `annotations_dir/{sample}`;
    - other potentially useful files.