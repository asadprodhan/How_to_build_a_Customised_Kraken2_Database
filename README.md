
<h1 align="center">How to Build a Customised Kraken2 Database</h1>


<h3 align="center">M. Asaduzzaman Prodhan<sup>*</sup> </h3>


<div align="center"><b> DPIRD Diagnostics and Laboratory Services </b></div>


<div align="center"><b> Department of Primary Industries and Regional Development </b></div>


<div align="center"><b> 3 Baron-Hay Court, South Perth, WA 6151, Australia <sup>*</sup>Correspondence: prodhan82@gmail.com </b></div>


<br />


<p align="center">
  <a href="https://github.com/asadprodhan/How-to-automatically-download-reads-from-the-NCBI-SRA/tree/main#GPL-3.0-1-ov-file"><img src="https://img.shields.io/badge/License-GPL%203.0-yellow.svg" alt="License GPL 3.0" style="display: inline-block;"></a>
  <a href="https://orcid.org/0000-0002-1320-3486"><img src="https://img.shields.io/badge/ORCID-green?style=flat-square&logo=ORCID&logoColor=white" alt="ORCID" style="display: inline-block;"></a>
</p>


<br />


## **Step 1: Download the genomes**


👉 [How to Retrieve Public Data](https://github.com/asadprodhan/Practical_Bioinformatics_for_Biologists#chapter-05--how-to-retrieve-public-data)


Keep these genome files (.fna or .fa) in a single directory.


<br />

## **Step 2: Download the taxonomy**


Kraken2 requires taxonomy files from NCBI to classify sequences.

Below is a Slurm batch script for downloading taxonomy files on an HPC cluster:


```
#!/bin/bash --login
#SBATCH --job-name=K2DB       		      # Name of the job
#SBATCH --account=XXXX             	    # Project account
#SBATCH --partition=work                # Partition (queue) to run the job
#SBATCH --time=1-00:00:00               # Time limit (1 day)
#SBATCH --ntasks=1                      # Single task/job
#SBATCH --ntasks-per-node=1
#SBATCH --nodes=1                       # Request 1 node
#SBATCH --exclusive                     # Request exclusive access to the entire node (all CPUs), alternative --cpus-per-task=128
#SBATCH --no-requeue
#SBATCH --export=none
#
# Activate the Kraken2 Conda environment
conda activate kraken2
#
# Download the taxonomy file
#
# Define DB directory
DBDIR=kraken2DB
export KRAKEN_DB_NAME=$DBDIR   # <-- important fix
#
# Step 1: Create DB directory and download taxonomy
kraken2-build --db $DBDIR --download-taxonomy --threads 120 --use-ftp
#
# End of script
```

<br />


Step 3: Add the genomes to library


```
#!/bin/bash --login
#SBATCH --job-name=K2DB
#SBATCH --account=XXXX
#SBATCH --partition=work
#SBATCH --time=1-00:00:00
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --exclusive
#SBATCH --no-requeue
#SBATCH --export=none

# Activate Kraken2 environment
conda activate kraken2

# Fix Kraken2 environment variables
DBDIR=kraken2DB
export KRAKEN_DB_NAME=$DBDIR
export KRAKEN_DIR=/path/to/miniconda3/envs/kraken2/libexec

# Add the genome(s) to the library
find . -name "*.fna" -print0 | \
  xargs -0 -I {} kraken2-build --db $DBDIR --add-to-library "{}" --threads 120
```

<br />


Step 4: Generate the seqid2taxid.map file


Step 4.1 

```
wget ftp://ftp.ncbi.nlm.nih.gov/pub/taxonomy/accession2taxid/nucl_gb.accession2taxid.gz
```

Step 4.2

```
gunzip nucl_gb.accession2taxid.gz
```

Step 4.3

```
cut -f2,3 nucl_gb.accession2taxid > seqid2taxid.map
```

Step 4.4


Copy the seqid2taxid.map to $Kraken2DB (not in the taxonomy or library directory)
 

Step 5: Build the DB


```
#!/bin/bash --login
#SBATCH --job-name=K2DB_BAFVP-job        # Name of the job
#SBATCH --account=pawsey0792             # Pawsey project account
#SBATCH --partition=highmem              # Partition (queue) to run the job
#SBATCH --time=1-00:00:00                # Time limit (1 day)
#SBATCH --ntasks=1                       # Single task/job
#SBATCH --ntasks-per-node=1
#SBATCH --nodes=1                        # Request 1 node
#SBATCH --exclusive                      # Request exclusive access to the entire node (all CPUs), alternative --cpus-per-task=128
#SBATCH --no-requeue
#SBATCH --export=none
#
# Activate Kraken2 environment
conda activate kraken2

# Fix Kraken2 environment variables
DBDIR=kraken2DB_ABFVP_072025
export KRAKEN_DB_NAME=$DBDIR
export KRAKEN_DIR=/software/projects/pawsey0792/prodhan21/miniconda3/envs/kraken2/libexec
#
# Build the database
#
kraken2-build --db $DBDIR --build --threads 120
# End of script
```


