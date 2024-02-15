-   This is a companion doc to the batchfile
    “qiime\_processing\_batch\_file.batch”, which contains the code to
    run a QIIME job on the HPC.
-   I have run everything on the Norman servers, though a similar batch
    job can be run on the OUHSC Mustang server as well with some
    modifications (and after QIIME is installed to the server–I don’t
    have a tutorial for that, though the person (currently Jessica Lam:
    jessica.lam @ ou.edu) running the servers may be willing to globally
    install packages if you ask).
    -   The OSCER servers in Norman operate on SGE, while the OUHSC
        Mustang servers use LSF.
        -   More info can be found at:
            <https://www.med.upenn.edu/hpc/assets/user-content/documents/SGE_to_LSF_User_Migration_Guide.pdf>
-   For our purposes, the only difference between the two is syntax.
    Let’s look at the batchfile header for SGE on the Norman OSCER
    servers, which uses qsub to submit jobs:

<!-- -->

    #!/bin/bash
    #SBATCH --ntasks=1
    #SBATCH --time=6:00:00
    #SBATCH --job-name=Assembly
    #SBATCH --mem=8G
    #SBATCH --output=16S_%J_stdout.txt
    #SBATCH --error=16S_%J_stderr.txt
    #SBATCH --mail-user=
    #SBATCH --mail-type=ALL
    #SBATCH --chdir=

-   The commands in the header are fairly self-explainatory, so I won’t
    go into too many details here. But there are a few things you should
    be aware of, depending on the server/cluster/nodes you have access
    to
    -   You need to include the first line of code (#!/bin/bash)
    -   Make sure each line starts with “\#”
    -   “–ntasks” sets the number of computing *nodes* that will be used
        by the program. Each node contains several processors (which can
        also be coded into the batchfile, though it is not shown here).
        If you use more processors (and/or nodes) than one, you’ll
        likely need to make sure that the program’s code itself
        specifies this.
        -   I’ll show a code snippet of this later
    -   Time and mem specify how much time the run should take and how
        much RAM is needed for these computations, respectively. Each
        node/processor has a maximum amount of RAM it can pull from, so
        requesting as much as you need can be challenging if you aren’t
        sure of run parameters
        -   Note that there’s nothing wrong, per se, with requesting
            more than you think you’ll need, just make sure you aren’t
            requesting more memory than is available on a given node;
            moreover, the more resources requested, the harder it can be
            to obtain server time
-   You’ll need a few files for your run:
    -   Raw sequencing files
    -   File with sample ID and file path to forward and reverse reads
        (you have single rather than pair-end reads, you’ll need to
        modify the code)
        -   It’s named sample\_file\_list.tsv
    -   The base pair length of your sequencing region
        -   Here it is around 250 bp
    -   (optional) The primers used for sequencing (F/R)
-   One other thing to note before we get into the code: I’m using the
    greengenes2 database for this run. It’s required to get gene
    annotation predictions form Picrust. The gg2 database was just
    updated last year, but you can also use other annotation DBs like
    SILVA (<https://www.arb-silva.de/>)
    -   If you use SILVA, you’ll need to look up how to download and
        process the database so that it is in a correct format for your
        needs; Google is your friend!
-   There are many pipelines online showing you how to use QIIME with
    different types of sequencing data, different annotation databases,
    etc. This is a pretty good one and covers more than we will here:
    -   <https://chmi-sops.github.io/mydoc_qiime2.html>
-   The full batch file to run QIIME on raw 16S sequences looks like
    this:
    -   See the comments in the code for more info on how to alter the
        run parameters

<!-- -->

    #!/bin/bash
    #SBATCH --ntasks=1
    #SBATCH --time=6:00:00
    #SBATCH --job-name=Assembly
    #SBATCH --mem=8G
    #SBATCH --output=16S_%J_stdout.txt
    #SBATCH --error=16S_%J_stderr.txt
    #SBATCH --mail-user=
    #SBATCH --mail-type=ALL
    #SBATCH --chdir=

    module load python
    module load QIIME2

    #sample_file_list.tsv is in here. This basically is a tab-separated (the .ts part of tsv) text file containing sample names in column 1, forward read file in column 2, and reverse read file in column 3
    #I usually pull the sequence names via the command line using ls to output my file. You can also assemble the file in excel (though I'd still use ls and copy/paste the file names to save some time)
    qiime tools import \
     --type 'SampleData[PairedEndSequencesWithQuality]' \
     --input-path sample_file_list.tsv \
     --output-path paired-end-demux.qza \
     --input-format PairedEndFastqManifestPhred33V2

    #Sometimes the sequences won't be demultiplexed (i.e., sequences won't be separated by sample yet). Most of the time, like they are here, the data will be demultiplexed, but if they aren't refer to this guide: https://chmi-sops.github.io/mydoc_qiime2.html
    qiime demux summarize \
     --i-data paired-end-demux.qza \
     --o-visualization demux-summary-1.qzv

    # you'll want to replace 50 and 300 with numbers appropriate for the length of your sequencing region; here we know that our sequences should all be 250 bp long
    qiime dada2 denoise-paired \
      --p-trim-left-f 50 \
      --p-trim-left-r 50 \
      --p-trunc-len-f 300 \
      --p-trunc-len-r 300 \
      --i-demultiplexed-seqs paired-end-demux.qza \
      --o-representative-sequences rep-seqs-1.qza \
      --o-table table-1.qza \
      --o-denoising-stats stats-1.qza

    qiime metadata tabulate \
      --m-input-file stats-1.qza \
      --o-visualization denoising-stats-1.qzv

    qiime feature-table summarize \
      --i-table table-1.qza \
      --o-visualization table.qzv \
     
    qiime feature-table tabulate-seqs \
      --i-data rep-seqs-1.qza \
      --o-visualization rep-seqs.qzv

    qiime tools export \
      --input-path table-1.qza \
      --output-path exported-otu-table
      
    #in the resulting folder:
    cd exported-otu-table
    biom convert -i feature-table.biom -o table.from_biom.txt --to-tsv
    cd ../ #need to move back to the main folder

    ###
    #greengenes2 annotations
    # See this link for more infor on gg2 and how to use it:
      # https://forum.qiime2.org/t/introducing-greengenes2-2022-10/25291
      
    # need to download the database
    wget http://ftp.microbio.me/greengenes_release/2022.10/2022.10.backbone.full-length.fna.qza

    wget http://ftp.microbio.me/greengenes_release/2022.10/2022.10.taxonomy.asv.nwk.qza

    qiime greengenes2 non-v4-16s \
        --i-table table-1.qza \
        --i-sequences rep-seqs-1.qza \
        --i-backbone 2022.10.backbone.full-length.fna.qza \
        --o-mapped-table gg2.biom.qza \
        --o-representatives icu.gg2.fna.qza

    qiime greengenes2 taxonomy-from-table \
        --i-reference-taxonomy 2022.10.taxonomy.asv.nwk.qza \
        --i-table gg2.biom.qza \
        --o-classification gg2.taxonomy.qza

-   That’s basically it! Using the HPC servers is a whole other thing
    that I don’t want to get into, but to run this job on the OSCER
    servers, you’d need to set up a few things:
    -   Put all the sequences, batch job, and other files in a folder
    -   Navigate to the folder with your data, and run:

<!-- -->

    qsub qiime_batch.txt

-   This will put your job in the queue to run and return a job ID. To
    check on your job runs:

<!-- -->

    squeue -u $USER

-   To cancel a run, replace *jobID* with the job ID given by either of
    the two steps above:

<!-- -->

    scancel *jobID*

-   So, I have less experience with LSF on the OUHSC servers, but I can
    show how I’ve been submitting jobs:

<!-- -->

    #!/bin/bash
    #BSUB -J "mpa"
    #BSUB -o assembly_%J_stdout.txt
    #BSUB -e assembly_%J_stderr.txt
    #BSUB -n 4

    eval "$(conda shell.bash hook)"
    conda activate mpa

    metaphlan CHO57M.merged_cat_output.fastq_u.fastq_ec.fastq.fixed_assembly.fastq_sequence_min1000.fastq --bowtie2out CHO57M.merged_cat_output.fastq_u.fastq_ec.fastq.fixed_assembly.fastq_sequence_min1000.fastq_metagenome.bowtie2.bz2 --input_type fasta -x mpa_vJan21_CHOCOPhlAnSGB_202103 --bowtie2db /storage01/home/ksugino/mpa_db/ --nproc 4

-   The header looks similar to OSCERs SGE setup, but I haven’t bothered
    to designate the time or memory needed by my programs; these are
    private servers for just the HHDC (at the moment at least) so there
    are fewer issues with memory and time allocations–they just use
    what’s needed
-   Note that I designate 4 nodes (-n 4) and I subsequently designate
    the number of processors I’ve allocated in the code for metaphlan
    (–nproc 4)
    -   I honestly don’t understand how the multithread processing
        works, but in short, it’s a way to put more resources towards a
        job so that it runs faster (though I’m sure it’s much deeper
        than that)