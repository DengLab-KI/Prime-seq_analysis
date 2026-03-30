# Prime-seq Analysis Pipeline – Deng Lab

<br>**Paulo Jannig** - [paulo.jannig@ki.se](mailto:paulo.jannig@ki.se) | [paulo.jannig@su.se](mailto:paulo.jannig@su.se) | [GitHub account](https://github.com/paulojannig)
<br>**Hong Jiang** - [hong.jiang@ki.se](mailto:hong.jiang@ki.se) | [GitHub account](https://github.com/brainfo)

This repository contains the Deng Lab's pipeline for analyzing **Prime-seq libraries** sequenced on the **DNBSEQ-G400 platform** or **Novogene-Illumina platform**, using **zUMIs**.

---

## 📋 Overview

This workflow covers:

* Quality control (QC) of raw and processed FASTQ files
* Preparation of sample-specific barcodes and zUMIs configuration
* Running zUMIs with Prime-seq specific parameters

---

## ⚙️ Installation and Setup

### 1. Install [Pixi](https://pixi.sh/latest/advanced/installation) (to manage environment)

```bash
curl -fsSL https://pixi.sh/install.sh | sh
pixi self-update
```

---

### 2. Clone Required Repositories

Create dependencies:
```bash
mkdir ~/github_resources
cd ~/github_resources
```

Clone this Prime-seq pipeline repository:
```bash
git clone https://github.com/DengLab-KI/Prime-seq_analysis.git
```

Clone zUMIs repository:
```bash
git clone https://github.com/sdparekh/zUMIs.git
```


---

### 3. Set Up Pixi Environment

```bash
cd ~/github_resources/Prime-seq_analysis
tmux new -s primeseq
```

```bash
pixi install
pixi shell -e default --manifest-path pixi.toml
```

---

## 🚀 Running the Pipeline

### Step 1: QC of Raw Reads

1. **Edit `config.sh`** to match your paths and project variables using VS Code (or by `nano config.sh`):

Example:
```bash
EXPERIMENT=PJ101_TEMPLATE
PATH_EXPERIMENT=/mnt/run/paulo/${EXPERIMENT}
PATH_RAW_DATA=/mnt/storage/paulo/PJ101_TEMPLATE/
PLATFORM=MGI
FLOWCELL=V350293965
PRIMESEQ_INDEX=IDTi51i7N701
```

Raw sequencing data for each user should be stored under `/mnt/storage/USER/`
You'll find our G400 runs within `/mnt/storage/G400/`

2. **Run QC script:**

This script will:

* Create the full project folder structure under your `${PATH_EXPERIMENT}`
* Copy raw FASTQ files and sequencing run reports from the server (`/mnt/storage/USER/`)
* Merge data from multiple sequencing lanes
* Run initial quality control (**FastQC** + **MultiQC**) on the **untrimmed reads**
* Trim Prime-seq specific adapter and unwanted regions (from Read 1 and Read 2)
* Run quality control again on the **trimmed reads**
* Organize logs, config files, and R scripts needed for the next steps

Run the script like this:

```bash
cd ~/github_resources/Prime-seq_analysis
nohup ./scripts/01.primeseq_QC.sh >> log.01.primeseq_QC.txt
```

This will keep the script running in the background and log the progress to `log.01.primeseq_QC.txt`.

**Expected runtime:**
Approximately 1–5 hours, depending on the number of samples and lanes in the flowcell.

3. **Check QC Reports:**

cd ~/EXPERIMENT/Data/00.reports


Open the untrimmed MultiQC report and go to <i>Per Base Sequence Content</i>:
```
~/$PATH_EXPERIMENT/Data/00.reports/Untrimmed/MultiQC_untrimmed_output/multiqc_report.html
```

✅ **QC Expectations:**

* **Read 1**: Contains **Cell Barcodes**, **UMIs**, and potentially some insert sequence.

  * Barcodes = noisy base distribution
  * UMIs = smoother, constant bases
  * Downstream insert (after BC/UMI) = T-rich (expected for Prime-seq)
* **Read 2**: Actual **cDNA fragment**

  * Check correct read length (e.g., 100 bp or 150 bp)
  * Note: zUMIs will typically **skip bases 1-14** of Read 2 during mapping (to avoid adapter/low-quality sequence).

---

### Step 2: Prepare Barcode and YAML Configs

#### 2a. Prepare the sample barcode file

Use `templates/sampleInfo.xlsx` and/or `templates/Template_Primeseq_barcodes_sample.txt` as a starting point.

* The file must contain exactly two columns: **`barcode`** (first) and **`sample`** (second)
* Save as **tab-delimited text** (`.txt` or `.tsv`)
* Each barcode corresponds to a unique well/sample in your Prime-seq plate

> 💡 Avoid spaces or special characters — these can cause issues in downstream R analysis.

---

#### 2b. Configure the zUMIs YAML file

Open `templates/primeseq_zUMIs.yaml` in VS Code and adjust the following:

**Project and file paths:**
* Set `EXPERIMENT` to match the name used in `config.sh`
* Set `USER` to your username
* Update paths to your FASTQ files (R1 and R2)
* Set the output directory: `/mnt/run/USER/$EXPERIMENT/`
* Set `barcode_file` to the path of your prepared `.tsv` file (e.g., `Primeseq_barcodes_samples.tsv`)

**Read configuration — adjust based on sequencing mode:**

| Sequencing mode | STAR index | `base_definition` |
|---|---|---|
| PE100 | `STAR_index_85` | `cDNA(15-100)` |
| PE150 | `STAR_index_135` | `cDNA(15-150)` |

**Reference genome:**
- Verify the STAR index path and GTF file match your target species

**Resources:**
- Adjust the number of threads based on available CPUs on the server

> ⚠️ Make sure the STAR index was built with a `sjdbOverhang` compatible with your read length (`read length − 1`). A mismatch is a common cause of poor mapping rates.

Save the configured file within `~/github_resources/Prime-seq_analysis/primeseq_zUMIs.yaml`.

### Step 3: Run zUMIs

```bash
cd ~/github_resources/Prime-seq_analysis
nohup ./scripts/02.primeseq_zUMIs.sh >> log.02.primeseq_zUMIs.txt
```

### Step 4: Downstream analysis in R
* The R Markdown templates for downstream analysis are available in `~/$PATH_EXPERIMENT/scripts/`
* You can either:
  * run the analysis directly on our workstation (recommended for large datasets or for VS code), or
  * transfer the `$EXPERIMENT` folder to your local machine (recommended for RStudio Desktop).
	
* Note that large files are stored in `~/EXPERIMENT/Data/`. If you download the experiment to your local machine, avoid syncing this folder.
* When the analysis is completed, move at least the `~/EXPERIMENT/Data/` directory to `/mnt/USER/storage/` for long-term storage. Do not keep raw data under `/mnt/run/`.

---

## ✅ Notes:

* Always monitor your log files for errors (log.01.primeseq_QC.txt and log.02.primeseq_zUMIs.txt)

## tmux quick cheatsheet:
- New session: `tmux new -s session_name` or simply `tmux`
- List sessions: `tmux ls`
- Attach: `tmux attach -t session_name`
- Detach: `Ctrl-b d`

