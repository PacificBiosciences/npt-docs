# PacBio PureTarget Carrier Pipeline (PTCP) on HPC

The PacBio PureTarget Carrier Pipeline is a WDL-based workflow designed to genotype tandem repeat regions and homologous genes with segmental duplications using PacBio PureTarget HiFi data. By cloning this repository and setting up a virtual environment you can install PTCP and run it on your HPC.

## Quick-start

## Table of Contents

___

## 1. Install requirements in a virtual environment

Below are example commands used to install the requirements needed to run PTCP. This example uses `conda` and `pip`, but other package/environment managers can be used.

```bash
git clone https://github.com/PacificBiosciences/ptcp.git
cd ptcp/
conda create -n ptcp python=3.12.11
conda activate ptcp
python -m pip install -r requirements.txt
which miniwdl
```

## 2. Setup the image of PTCP dependencies

PTCP is a WDL workflow that relies on many software pacakges. These software are containerized in an image for PTCP to call to. It can be invoked as a Docker container or a Singularity image file. Instructions for setting up both are shown below.

### 2.1. Docker image of PTCP dependencies

A pre-built Docker image of these dependencies can be found on quay.io at: (INSERT).

```bash
docker pull quay.io/pacbio/ptcp-dependencies:latest
```

### 2.2 Singularity image file of PTCP dependencies

You can build a `.sif` of PTCP dependencites for the verion of PTCP you are installing by using `apptainer` (or `singularity`) to buil the `.sif` from the Docker image hosted on quay.io. An example command is shown below:

```bash
apptainer pull ptcp_2.0.sif docker://quay.io/ORG/ptcp:2.0
```

Once you have build the `.sif` file you should move it to the `ptcp` software folder, like so:

```bash
cd /Path/to/ptcp
mkdir miniwdl_singularity_cache
cd miniwdl_singularity_cache
mv /Path/to/Downloads/ptcp_dependencies.sif .
```

## 3. Gather input information

PTCP requires several inputs, all summarized in an input JSON file. For detailed information on each of the input files required, please refer to the [Input Files page](./Input_files.md). This section will briefly cover how to create the `sample-sheet.csv` and the input JSON file used to run the pipeline.

### 3.1 Create the sample-sheet.csv

The `sample-sheet.csv` contains three comma separated fields. For more information on how to format this file, please refer to the [Input Files page](./Input_files.md). You will need to create this file with one row per sample before you can create the input JSON in the next section.

### 3.2 Create the input JSON

The input JSON contains all of the input information required by PTCP. Because the input JSON contains sample-specific information, it needs to be generated for each run of PureTarget Carrier Panel data you want to analyze. This repository contains a script called [create_input_json.py](../docker/ptcp/scripts/create_input_json.py) to make it easier to generate the input JSON with the sample and other input information. This script requires a template JSON be passed to it with certain fields filled in.

An example template of the input JSON is provided in this repository called [inputs_json_template.json](../tests/inputs/templates/inputs_json_template.json). It is recommended that you copy this template and update all of the fields with preset example paths to point to their file locations on your server. For instance, in the example below, you should update all fields **_except for_** `ptcp.sample_sheet`, `ptcp.hifi_reads`, and `ptcp.fail_reads`:

```bash
{
    "ptcp.sample_sheet": "",
    "ptcp.ref_fasta": "/path/to/reference/hg38.fa",
    "ptcp.ref_index": "/path/to/reference/hg38.fa.fai",
    "ptcp.trgt_bed": "/path/to/ptcp/meta/trgt/PureTarget_repeat_expansion_panel_2.0.repeat_definition.GRCh38.bed",
    "ptcp.paraphase_config_yaml": "/path/to/ptcp/meta/paraphase/paraphase_config.GRCh38.yaml",
    "ptcp.genome_version": "38",
    "ptcp.ptcp_qc_bed": "/path/to/ptcp/meta/ptcp-qc/ptcp-qc.GRCh38.bed",
    "ptcp.hifi_reads": [],
    "ptcp.fail_reads": []
}
```

Once you have your template updated, you can use it to create the input JSON with the `create_input_json.py` script like so:

```bash
python create_input_json.py \
    --data /Path/to/demux/PacBio/Run \
    --sample_sheet /Path/to/sample-sheet.csv \
    --template /Path/to/template.json \
    > ptcp_inputs.json
```

### 3.3 MiniWDL configuration

PTCP also requires a **MiniWDL configuration file** as input. This file defines important runtime settings such as image cache, task concurrency, container backends, and more. You must update the configuration file to match your local system or execution environment.

Make sure to review and customize the configuration according to your hardware, software paths, and environment preferences before running the workflow.

**Example Configuration File**  
You can find an example MiniWDL configuration file here at
[test/miniwdl.cfg](../tests/miniwdl.cfg).

For more information about configuring MiniWDL, refer to the [official MiniWDL documentation](https://miniwdl.readthedocs.io/en/latest/GettingStarted/#configuration).

## 4. Running PTCP

Once you have your environment configured, the SMRT Tools image installed, and your inputs gathered you can run PTCP locally. Below is an example command used to run the pipeline:

```bash
conda activate ptcp

cd /Path/to/ptcp

miniwdl run \
    --verbose \
    --dir /Path/to/output_dir \
    --cfg /Path/to/miniwdl.cfg \
    --input /Path/to/ptcp_inputs.json \
    main.wdl
```

___

#### Old docs

1) Build a docker image. Optionally build singularity image.

    ```bash
    cd docker/ptcp/

    bash -x ./build.sh

    # to build the sif image for singularity, e.g.,
    # singularity build -d "docker____ptcp_latest.sif" docker-daemon://ptcp:latest
    ```

2) Optional. Set up miniwdl to test pipeline.

    ```bash
    # create and source virtual environment
    python3 -m venv venv
    source venv/bin/activate

    # install dependencies
    python3 -m pip install -r requirements.txt

    # edit tests/miniwdl.cfg as desired
    ```

3) Set up inputs.json.

    ```bash
    cp tests/inputs/templates/* tests/inputs/
    # edit to test
    ```

4) Run pipeline.

    ```bash
    miniwdl run --verbose --dir ./miniwdl_test_output --cfg ./tests/miniwdl.cfg --input ./tests/inputs/main.inputs.json main.wdl
    ```

#### Makefile

The Makefile contains a few useful targets for linting the workflow, creating templates for tasks/workflows, and running tasks/workflows with miniwdl.

```bash
# lint common.wdl
make check wdl=common

# lint all wdl
make check-all

# list all tasks in common.wdl
make list-tasks wdl=common

# create template for task
make task-template wdl=common task=collect_inputs

# test a task
make task-run wdl=common task=collect_inputs

# create template for workflow
make template wdl=main

# run workflow
make run wdl=main
```