# gtdb2cat

Notes about creating necessary inputs for `CAT prepare` based on the GTDB data.

## Requirements

Some commands in the notebooks are provided as markdown code blocks. To be 
able to run thesea you need a Linux environment (was `Ubuntu LTS 20.04.2` was 
used here), with the following core utils :

  - `wget`
  - `cut`
  - `grep`
  - `sort`
  - `uniq`
  - `sed`

I also used `parallel` at some point, but it might not be necessary.

For fetching protein files from the NCBI ftp and parsing the fasta files 

- `ncbi-genome-download` (`0.3.0` was used here)
- `biopython` (`1.79` was used here)

CAT itself: `cat` (`5.2.3` was used here)

All other stuff is in standard `python 3.9.4`.

* **Recommended**

Create an isolated `conda` environment with the provided 
`environment.yaml`. This will also install the `ipykernel` module that should 
make the whole environment visible to your jupyter instance.

So 

```
$ git clone <this repo>
$ cd <this repo>
$ conda env create -n gtdb --file=environment.yaml
```

You might need to install `nb_conda_kernels` in the environment you are 
running jupyter from, to make the `gtdb` kernel available.

Testing this:

You can use a platform specific environmnet provided within the `conda_envs` 
directory to get the exact same versions of packages, always. 
Use the appropriate lock file for your platform. E.g if you are on `OSX` 

```
$ conda create -n gtdb --file conda_envs/conda-osx-64.lock
```

These files were produced with 
[conda-lock](https://github.com/conda-incubator/conda-lock/) and

```
$ conda-lock -f environment.yaml \
--filename-template conda_envs/conda-{platform}.lock

# Windows fails...  Microtears (TM)
```

## Usage

There are 2 main jupyter notebooks.

1. `taxonomy.ipynb`

Includes notes on the process of creating NCBI-like taxonomy files 

  - `nodes.dmp`
  - `names.dmp`

* Currently unique numeric ids are produced for all ranks in the taxonomy.
This might be unecessary, since the prefixed names (e.g. `p__UBA9089`) are 
already unique. Not sure if there if CAT expects `int`s or just any unique 
string will do

 
2. `sequences.ipynb`

Includes notes on the process of fetching protein sequences for all genomes 
represented in the GTDB. These results in

  - `gtdb.nr.fa`
  All proteins fetched for as many genomes possible. Not all assembly 
  accessions have proteins.faa.gz files available.

  - `prot.accession2taxid.txt`
  A mapping of all protein accessions, included in the fasta file above,
  to their respective taxonomy unique identifier.

Note that these are written uncompressed and they take up quite some space.
  - `gtdb.nr.fa` - `309G`
  - `prot.accession2taxid.txt` - `23G`

All protein fasta files take up `156G` in total.

So, don't try this at home.


# Output

All files generated by the code and commands in the notebooks are written in the 
`results` dir. 

This is currently structured as follows

```
$ tree -d -L3 results
results
├── db  --------------- Contains the gtdb.nr.gz and prot.taxid2accession.gz
├── proteins ---------|
│   ├── genbank       |
│   │   ├── archaea   |
│   │   └── bacteria  | -- The raw fasta files for proteins,
│   └── refseq        |    per NCBI section, per domain
│       ├── archaea   |    (default ncbi-genome-download output structure)
│       └── bacteria  |
└── taxonomy  --------- Contains nodes.dmp and names.dmp
```

The gzipped files for the `gtdb.nr.gz` and `prot.taxid2accession.gz` are 
manually created with 

```
$ cat results/db/gtdb.nr.fa | gzip -c > results/db/gtdb.nr.gz
$ cat results/db/prot.accession2taxid.txt | gzip -c > results/db/prot.taxid2accession.gz

# Optionally remove the original files if gzip finished successfully.
```

# CAT database creation

Use the `--existing` option with `CAT prepare`.
The `prepare` module checks for the existence of certain files based on their 
names, so it needs to be fooled.

1. Prepare some clean dirs and copy/rename/move files accordingly

```
## <date> is of the form YYYY-MM-DD
$ mkdir -p results/CAT_prepare/CAT_taxonomy.<date>
$ mkdir -p results/CAT_preapare/CAT_database.<date>

# Copy the required files

## Sequences
$ mv results/db/gtdb.nr.gz results/CAT_prepare/CAT_database.<date>/<date>.nr.gz

## Taxonomy
$ mv results/db/prot.accession2taxid.gz results/CAT_prepare/CAT_taxonomy.<date>/<date>.prot.accession2taxid.FULL.gz

$ mv results/taxonomy/names.dmp results/CAT_prepare/CAT_taxonomy.<date>/names.dmp

$ mv results/taxonomy/nodes.dmp results/CAT_prepare/CAT_taxonomy.<date>/nodes.dmp

```
2. Launch the job

* Don't forget to activate the environment, or make sure you have `CAT` and 
`DIAMOND` available in ypur `$PATH`.

* Make sure you are on a machine with enough memory (at least `200GB`). 

```
# Get in the dir from above
$ cd results/CAT_prepare

# Do it
$ CAT prepare --existing --verbose \
-d CAT_database.<date> \
-t CAT_taxonomy.<date>
```

