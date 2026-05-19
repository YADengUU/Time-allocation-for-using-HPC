# Time-allocation-for-using-HPC

Efficient cluster usage ensures your jobs launch faster in the queue, run without failing due to timeouts or out-of-memory errors, and respect shared infrastructure limitations.

- [Core Architecture: Anatomy of Computing Nodes](#core-architecture-anatomy-of-computing-nodes)
- [Resource "Prediction": Benchmarking via Minimal Examples](#resource-prediction-benchmarking-via-minimal-examples)
- [Automation: Batch Scripts and Job Arrays](#automation-batch-scripts-and-job-arrays)
- [Disk I/O Optimization: Exploiting Virtual Storage (`$TMPDIR`)](#disk-io-optimization-exploiting-virtual-storage-tmpdir)

## Core Architecture: Anatomy of Computing Nodes
A cluster is not one massive computer; it is a network of individual computers called Compute Nodes connected by a high-speed interconnect.

### Node vs. Core
- Compute Node: A single physical server blade.

- Core (CPU): An individual processing unit within that server. A single node contains multiple cores.

- Hyperthreading: On some UPPMAX systems, physical cores may expose virtual cores (threads). For heavy scientific computing, we generally benchmark and track performance based on physical cores.

#### The `uquota` Command
Storage and core usage are strictly tracked via project allocations. Before submitting jobs, check your project balance and disk quotas by typing: `uquota`. This will show your project’s monthly core-hour allocation and how much of your allocation has been spent.

## Resource "Prediction": Benchmarking via Minimal Examples
SLURM (Simple Linux Utility for Resource Management) relies on your estimates to schedule jobs. If you request too much time or memory, your job sits in the queue forever. If you request too little, SLURM kills your job mid-execution.

To "predict" your needs, you can map the scaling behavior of your software using small-scale test runs.

### Step 1: Run an Interactive Test (with your mMinimal example)
Do not guess your resource usage on a 100 GB dataset. Scale your dataset down to 1% or 5% (e.g., a single chromosome, a subset of images, or a brief simulation timeline).

Request an interactive node for testing, for example: 
```
interactive -A sensxxxxx -p core -n 2 -t 01:00:00
```
(This grants you 2 cores for 1 hour to test interactively).

### Step 2: Profile the Execution
Run your program inside the interactive session prefixed with the Linux time command or a wrapper tool like /usr/bin/time -v:
```
/usr/bin/time -v python3 my_analysis_script.py --input mini_dataset.txt
```

Look for two critical metrics in the output: elapsed (wall clock) time: How long it took and maximum resident set size: The peak RAM (memory) used by the process.

### Step 3: Extrapolate and Add a Safety Buffer
If your 10% mini-dataset took 10 minutes and consumed 8 GB of RAM:

- Linear Scaling Assumption: A 100% dataset might take roughly 100 minutes and 8 GB of RAM (if memory scales by dataset structure, or more if it scales by data size).

- For SLURM Requests: Add a 20–50% safety buffer to your predictions. Requesting 3 hours (180 minutes) and 12 GB of RAM ensures that minor data anomalies won't trigger an automated SLURM abort.

### Step 4: Post-Mortem Analysis (seff)
Once a batch job completes, inspect its actual resource efficiency using the SLURM efficiency tool:
```
seff <job_id>
```
This returns a report showing exactly how much CPU and Memory was utilized. If your memory efficiency is below 50%, lower your requests for future runs.

## Automation: Batch Scripts and Job Arrays
Manually submitting 100 separate jobs is tedious and clogs the scheduler. Instead, we use SLURM batch scripts for single workflows and Job Arrays for parallel execution.

Structure of a SLURM Batch Script (submit_job.sh):
```
#!/bin/bash
#SBATCH -A sens202X-X-XX       # Project allocation account
#SBATCH -p core                 # Partition (core or node)
#SBATCH -n 4                    # Number of cores requested
#SBATCH -t 04:00:00             # Time limit (HH:MM:SS)
#SBATCH -J MyJob          # Job name
#SBATCH -o logs/job_%j.out      # Standard output log (%j expands to jobID)
#SBATCH -e logs/job_%j.err      # Standard error log

# Clean the environment and load required modules
module purge
module load bioinfo-tools
module load samtools/1.17

# Execute the software
samtools sort -@ 4 -o output_sorted.bam input.bam
```
Submit this script using: `sbatch submit_job.sh`

### Leveraging Job Arrays for Parameter Sweeps or Multi-Sample Processing
If you have 50 fastq files to process using identical parameters, do not write 50 scripts. Use a Job Array. SLURM creates 50 instances of your job but assigns a unique index variable (`$SLURM_ARRAY_TASK_ID`) to each one.
Here is a robust Job Array script (`array_job.sh`):
```
#!/bin/bash
#SBATCH -A sens202X-X-XX
#SBATCH -p core
#SBATCH -n 1
#SBATCH -t 02:00:00
#SBATCH -J SampleProcessing
#SBATCH --array=1-50            # Creates 50 tasks indexed 1 to 50
#SBATCH -o logs/array_%A_%a.out # %A = Master JobID, %a = Array Task index

# Design a mapping file (samples.txt) containing one sample name per line.
# Line 1: sample_A
# Line 2: sample_B ...
SAMPLE_NAME=$(sed -n "${SLURM_ARRAY_TASK_ID}p" samples.txt)

echo "Processing task ID ${SLURM_ARRAY_TASK_ID} corresponding to sample: ${SAMPLE_NAME}"

# Run analysis on the specific sample
module load blast/2.14.0
blastn -query data/${SAMPLE_NAME}.fasta -db nt -out results/${SAMPLE_NAME}_blast.txt
```

Also submit with: `sbatch array_job.sh`

## Disk I/O Optimization: Exploiting Virtual Storage (`$TMPDIR`)
High-performance computing clusters suffer from a major bottleneck: Disk Input/Output (I/O). When hundreds of jobs try to read and write millions of tiny, temporary files to the shared network file system (/home or /proj) simultaneously, the global storage system grinds to a halt.

To prevent this, every compute node features its own fast, isolated, local storage drive mapped to an environment variable called `$TMPDIR`.

### When to use $TMPDIR

Your software creates thousands of intermediate `.tmp`, `.cache`, or swap files.

Your pipeline unzips a massive archive, reads individual lines, and deletes them before finishing.

### How $TMPDIR Works
When your SLURM job starts, a unique, secure directory is automatically generated on the local Solid State Drive (SSD) of the specific node hosting your job.

The environment variable `$TMPDIR` points directly to this location.

Crucial: The moment your job ends, the node clears this directory completely. You must copy important outputs back to /proj/ before your script terminates.

Example implementation in a batch script:
```
#!/bin/bash
#SBATCH -A sens202X-X-XX
#SBATCH -p core
#SBATCH -n 2
#SBATCH -t 05:00:00

# Step 1: Stage input data from shared project space to local node storage
echo "Staging data to local node..."
cp /proj/my_project/large_input_dataset.tar.gz $TMPDIR/
cd $TMPDIR

# Step 2: Perform heavy I/O operations locally
tar -xzf large_input_dataset.tar.gz
rm large_input_dataset.tar.gz # Free up space inside $TMPDIR

module load R/4.3.1
# My R script writes 10,000 temporary matrix files right here in the current folder ($TMPDIR)
Rscript run_heavy_io_simulation.R 

# Step 3: Save only the final, compiled results back to project storage
echo "Job calculations finished. Compiling and migrating final results..."
mkdir -p /proj/my_project/final_results/
cp compiled_summary_report.pdf /proj/my_project/final_results/

# End of script. SLURM will now automatically wipe everything left behind in $TMPDIR.
```

