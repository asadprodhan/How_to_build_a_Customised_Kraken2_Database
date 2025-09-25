
<h1 align="center">How to Build a Customised Kraken2 Database</h1>


<h3 align="center">M. Asaduzzaman Prodhan<sup>*</sup> </h3>


<div align="center"><b> DPIRD Diagnostics and Laboratory Services </b></div>


<div align="center"><b> Department of Primary Industries and Regional Development </b></div>


<div align="center"><b> 3 Baron-Hay Court, South Perth, WA 6151, Australia </b></div>


<div align="center"><b> *Correspondence: prodhan82@gmail.com </b></div>


<br />


<p align="center">
  <a href="https://github.com/asadprodhan/How_to_build_a_Customised_Kraken2_Database/blob/main/LICENSE"><img src="https://img.shields.io/badge/License-GPL%203.0-yellow.svg" alt="License GPL 3.0" style="display: inline-block;"></a>
  <a href="https://orcid.org/0000-0002-1320-3486"><img src="https://img.shields.io/badge/ORCID-green?style=flat-square&logo=ORCID&logoColor=white" alt="ORCID" style="display: inline-block;"></a>
</p>


<br />


## **Introduction**

In metagenomics and microbial ecology, accurately classifying sequencing reads depends heavily on the quality and relevance of the reference database used. Kraken2 is a high-performance tool for taxonomic classification, but using its default or standard databases may not always be sufficient—especially if you want to focus on a particular taxonomic group, include novel genomes, or exclude irrelevant ones.

This tutorial walks through the steps of building a customised Kraken2 database from scratch. 

By following these steps, you’ll be able to generate a Kraken2 database that is tailored to your project’s needs—maximising accuracy, relevance, and computational efficiency.


<br />

---

## **Step 1: Download the genomes**


👉 [How to Retrieve Genomic Data](https://github.com/asadprodhan/Practical_Bioinformatics_for_Biologists#chapter-05--how-to-retrieve-public-data)


- Keep these genome files (.fna or .fa) in a single directory

- Within that directory, run the script presented in Step 2.  


---


<br />


## **Step 2: Download the taxonomy**


- Kraken2 requires taxonomy files from NCBI to classify sequences.

- Below is a Slurm batch script for downloading taxonomy files on an HPC cluster:


```
#!/bin/bash --login
#SBATCH --job-name=Taxonomy             # Name of the job
#SBATCH --account=XXXX             	    # Project account
#SBATCH --partition=xxx                 # Partition (queue) to run the job
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


### **How does the script work**

- conda activate kraken2 → loads Kraken2 environment

- DBDIR=kraken2DB → assigns the name for the database directory. Change it if you like 

- export KRAKEN_DB_NAME=$DBDIR → creates the database directory and tells Kraken2 to store the taxonomy files in there

- kraken2-build --download-taxonomy → fetches NCBI taxonomy files

- --threads 120 → uses 120 CPU threads for faster download. Change it based on your computing resources

- --use-ftp → ensures NCBI files are retrieved via FTP


---


<br />


## **Step 3: Add the genomes to library**


- At Step 3, you will supply the downloaded genomes to Kraken2 for converting them into Kraken2 database at Step 4.


```
#!/bin/bash --login
#SBATCH --job-name=K2DB
#SBATCH --account=XXXX
#SBATCH --partition=xxx
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


### **How does the script work**


- find . -name "*.fna" → looks for all .fna files in current directory

- xargs ... kraken2-build --add-to-library → adds each genome file into the Kraken2 library

- --threads 120 → speeds up processing when handling many genomes

---

<br />


## **Step 4: Generate the seqid2taxid.map file**


Kraken2 needs a mapping file between genome sequence IDs and taxonomy IDs. This ensures correct classification.


### **Step 4.1**

```
wget ftp://ftp.ncbi.nlm.nih.gov/pub/taxonomy/accession2taxid/nucl_gb.accession2taxid.gz
```

Explanation:

- NCBI provides accession2taxid files mapping every accession to its taxonomy ID

  
### **Step 4.2**

```
gunzip nucl_gb.accession2taxid.gz
```

### **Step 4.3**

```
cut -f2,3 nucl_gb.accession2taxid > seqid2taxid.map
```

Explanation:

- Kraken2 needs a slimmed-down seqid2taxid.map

- Only column 2 (accession.version) and column 3 (taxid) are kept

  
### **Step 4.4**


**Copy the seqid2taxid.map to $Kraken2DB (not in the taxonomy or library directory)**

---

<br />


## **Step 5: Build the DB**


Now that taxonomy and genome libraries are in place, the final step is to compile the database.


```
#!/bin/bash --login
#SBATCH --job-name=K2DB                  # Name of the job
#SBATCH --account=xxxx                   # Pawsey project account
#SBATCH --partition=xxx                  # Partition (queue) to run the job
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
#
# Fix Kraken2 environment variables
DBDIR=kraken2DB
export KRAKEN_DB_NAME=$DBDIR
export KRAKEN_DIR=/path/to/miniconda3/envs/kraken2/libexec
#
# Build the database
#
kraken2-build --db $DBDIR --build --threads 120
# End of script
```

Explanation:

- kraken2-build --build → processes taxonomy + genomes into a searchable DB

- --threads 120 → uses multiple cores for faster DB building

- DBDIR naming convention (e.g., kraken2DB) helps version-control DBs



## **At this stage, your custom Kraken2 database is ready for classification!**
