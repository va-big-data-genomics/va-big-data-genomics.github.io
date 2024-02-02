---
layout: post
title:  "#41 TelSeq tutorial, CHIP calling and SV calling in MVP"
date:   2024-02-02 10:05:00 -0800
author: Prathima Vembu 
categories: jekyll update
---

## TelSeq tutorial - Draft 1

### 1. Create docker image and upload to Dockerhub:

The first step to running a tool on Google cloud is to dockerize the tool. The dockerized tool is then pushed to Dockerhub which is a cloud-based registry service that allows developers to store, manage, and distribute Docker container images. It serves as a central repository where users can find, download, and share container images created by others. The complete start guide can be found [here](https://docs.docker.com/docker-hub/quickstart/).

#### 1.1. Create a Docker Hub account:
- Create a docker hub account [here](https://hub.docker.com/), if you do not have one. 
- To create a repository, select the `Create Repository` icon on the top right-hand side of the homepage. 
- Choose a suitable name and description for your repository and make sure to set the visibility of the image to `Public`. Failure to do so will inhibit the docker container from accessing the image when running on cloud.

#### 1.2. Docker desktop installation:
- To begin, download and install `Docker Desktop` by following the instructions provided [here](https://docs.docker.com/desktop/install/mac-install/). 
- Once Docker is installed, launch it to ensure it's running correctly. The status can then be confirmed by checking for its icon in the system tray or taskbar.

#### 1.3: Creating Dockerfile :
- Create a new file named `Dockerfile` (without any file extensions) within the desired directory, using a text editor on the command line. The absence of file extensions is essential for Docker to recognize the file.
- Include the instructions to download and install the packages required for running TelSeq on GCP within the file. Refer to the official Docker documentation [here](https://docs.docker.com/engine/reference/builder/) for further information. 
- The `Dockerfile` should be in the same directory where you are executing the Docker commands and save the.  
- Build the Docker image using the command `docker build`. Specify the path to the directory containing the Dockerfile by using a period (.) to represent the current working folder. 

```
docker build -t username/tool:version .
```

#### 1.4: Verify the new docker image
After successfully building the Docker image, the user can verify the installation by running the docker command on the command line interface. This will ensure that Docker is properly installed and functioning. To interactively test the Docker image and check for desired results, use the `docker run` command followed by the `-it` flag to run the image in an interactive mode. Specify the name of the image and the input files as arguments. 

```shell
docker run -it username/toolname:version input_file
```

#### 1.5: Pushing the image to DockerHub:
- To share your Docker image with others, you can push it to DockerHub. 
- From the command line, use the `docker push` command, followed by the name of the image to upload the image to DockerHub. 
- The docker image utilized for this particular study is accessible at pvembu/telseq. The corresponding Dockerfile can be found here. 
- For detailed instructions on constructing Dockerfiles and pushing images to Docker Hub, you can refer to the documentation provided here.

### 2. Scripts required to run TelSeq on GCP

To run TelSeq on GCP, three different scripts will be required. 

#### 2.1. TelSeq command script

- This is a shell script that encompasses all the essential parameters for TelSeq, as well as additional commands such as naming the output file with the input file’s name. 
- If desired, the user can dynamically estimate the read length within this script and provide it as an input parameter. Standard command for running TelSeq on CRAMs:

```shell
samtools -u $CRAM $REF | telseq -r 150 -k 7 -u > Ouptut.txt
```
Where:

* samtools -u : reads in a CRAM file and converts it into an uncompressed BAM file.
* $CRAM : represents the input CRAM file.
* $REF : denotes the reference genome in FASTA format. It is required for the CRAM to BAM conversion. For this study, GRCh38 was selected as the reference genome.
* -r : specifies the read length for each read. The default value is 100, but it can be customized according to the specific read length.
* -k : sets the threshold for the number of TTAGGG/CCCTAA repeats in a read to be considered telomeric. The default value is 7.
* -u : indicates that all the reads are from the same read group.
* The output is redirected to a file named “Output.txt”

*Note : The values of -r and -k according to the specific requirements of the input BAMs/CRAMs. Also,` samtools -u` need not be used when the input file is in the BAM format.*

The read length was identified to be 150 by running the command:

```shell
samtools view input.cram | awk '{print length($10)}' | head -1000 | sort -u
```

Consequently, a repeat number of 12 for k was chosen to run TelSeq on MVP samples. Additional information regarding the read length and the selected value of k can be found in this study. For reference, the complete script used for TelSeq can be accessed here(Add link from public repository, once ready).

#### 2.2. Tab-separated file (TSV)

Instead of running the dsub command multiple times for each of the MVP samples individually, an alternative approach is to consolidate the necessary variables, inputs, and outputs for each task into a tab-separated values (TSV) file. This TSV file can then be used as input to invoke the dsub command once. This approach generates a single job ID that encompasses multiple tasks.

Although each task runs independently, they can be collectively monitored and managed. This consolidated approach allows for easier tracking of the job status and simplifies the process of deleting or managing the tasks as needed.

Should the project have a quota limit on the number of virtual machines that can be deployed simultaneously, the remaining tasks will be queued and executed as resources become available. This helps optimize resource utilization and ensures that tasks are completed in an orderly manner.

To check the status of the job and monitor the progress of the tasks, users can utilize the `dstat` command. This command provides real-time updates on the job status, allowing users to track the execution and identify any issues or errors that may arise. By employing this approach, you can streamline the execution of multiple tasks, efficiently manage resources, and closely monitor the progress of the job.

The initial line of the TSV file defines the parameter names and their corresponding types. Each additional line in the file provides the input, reference, and output values for each task, as shown below:

```
--input CRAM    --input REF --input REF_FAI --output OUTPUT
gs://PATH/TO/.cram  gs://PATH/TO/.fa    gs://PATH/TO/.fai   gs://PATH/TO/OUTPUT/*.txt
gs://PATH/TO/.cram  gs://PATH/TO/.fa    gs://PATH/TO/.fai   gs://PATH/TO/OUTPUT/*.txt
.
.
gs://PATH/TO/.cram  gs://PATH/TO/.fa    gs://PATH/TO/.fai   gs://PATH/TO/OUTPUT/*.txt

```
   
*Note: Ensure that the names of the parameters are the same in the `telseq.tsv` and the `telseq_script.sh `scripts*

#### 2.3. dsub script

Dsub is a powerful utility offered by GCP that facilitates the submission and management of batch jobs on the cloud. To begin using dsub, you need to install it on your system by following the instructions provided [here](https://github.com/DataBiosphere/dsub). By leveraging dsub, you can define job parameters, input data, and output locations, and submit them as a batch. 

A dsub script serves as a structured representation of instructions and parameters necessary for executing a specific task or batch job. It offers a systematic approach to define various aspects of the job submission, including inputs, outputs, environment configurations, resource demands, and other pertinent details.

Typically, the dsub script is authored in a plain text format like YAML or JSON, adhering to a specific syntax or schema prescribed by the dsub tool. By utilizing a dsub script, users can maintain a well-organized and comprehensive specification for their job requirements, facilitating reproducibility and ease of execution.

During the execution of the dsub command, when a script is provided as input, the tool parses and comprehends the script’s contents instigating and overseeing the associated batch job on the cloud platform. This process entails a range of operations, including the seamless provisioning of requisite resources, the execution of commands or scripts within the job’s context, the vigilant monitoring of job progress, and the adept handling of job culmination or unexpected failure scenarios. By interpreting the script’s instructions, the dsub tool orchestrates a sophisticated workflow, ensuring the smooth execution and management of the designated batch job within the cloud environment.

The dsub script used to run TelSeq can be found here(Add link from public repository, once ready). Shown below is an example of a simple dsub script. 

```
#!/bin/bash

dsub \
--provider google-v2 \
--project <PROJECT_ID> \
--regions <REGION_ID> \
--machine-type <MACHINE_TYPE> \
--logging <PATH_TO_OUTPUT_LOGS> \
--name <JOB_NAME>\
--image <DOCKER_IMAGE> \
--task <PATH_TO_TSV_FILE> \
--script <PATH_TO_INPUT_SCRIPT>
```

### 3. Uploading scripts to GCS

To enable command-line interaction with Google Cloud Storage (GCS), it is essential to install the Google Cloud SDK, which can be installed from the official documentation here. During the installation process, it is important to ensure the selection and installation of the components specifically related to gsutil and gcloud. These components provide the necessary tools and functionalities for seamless communication with GCS using the command line, as shown below.

```
gsutil cp <script> gs://bucket_name 
```

However, if you are not currently located in the same directory as the script files, it is essential to provide the complete path to the script files to ensure their successful upload. Transferring these files to GCS is crucial as it ensures accessibility for dsub. Failure to upload the files to GCS will result in dsub being unable to access them, thereby hindering the execution of tasks.

### 4. Running TelSeq on GCP

To initiate a TelSeq dsub job on GCP, follow the steps outlined below:

- Edit the dsub script and verify that the file paths referenced for the `telseq.tsv` and `telseq_script.sh` files are accurate and correctly specified. It is crucial to ensure that the file paths are correctly mentioned to enable proper execution of the dsub job. save the edited file after making the necessary changes. 
- Execute the command `bash telseq_dsub.sh` to launch the TelSeq dsub job. This command triggers the execution of the specified dsub script, initiating the TelSeq job on GCP.
To achieve a streamlined execution, you can follow the steps outlined in this best practices [guide](https://github.com/DataBiosphere/dsub). 
- Monitoring the job’s progress is facilitated by the `dstat` command, which provides real-time updates on the job’s status and can be conveniently accessed through the standard output (STDOUT) using the corresponding job ID.

It is important to note that running TelSeq tasks on Google Cloud incurs charges based on various factors such as the selected machine type, disk size, and region. To estimate the associated runtime and costs, you can refer to the detailed information provided here. Monitoring the log files is highly recommended to ensure the continuous execution of the job.

### 5. Merging TelSeq '.txt' files into a single dataframe
Section update in progress

#### 5.1. Reading TelSeq '.txt' files from GCP into a dataframe 

#### 5.2. Uploading merged TelSeq output dataframe to GCS  


### 6. Pre-processing of demographic file  

#### 6.1. Removing duplicate values from demographic file
A check for duplicate values was performed on the demographic data. A total of 257 duplicate values were identified and subsequently removed from the dataset, ensuring the elimination of any redundancy.

#### 6.2. Checking demographic file for negative age values
The dataset was then examined for negative age values. By excluding samples with missing ethnicity (NA) we confirmed that no negative age values were present in the dataset

#### 6.3. Merging with TelSeq dataframe and filtering for 'NA' values

The results obtained from TelSeq analysis were imported into a dataframe for further examination. To enrich the dataset, these TelSeq results were merged with corresponding phenotypic data, incorporating age, sex, and ethnicity values for each sample. 

Following the merging process, samples that did not have ethnicity information (represented as NA, accounting for 6292 samples) were carefully excluded from the study. This filtering ensured that the final dataset consisted only of samples with complete demographic information, enabling more robust and accurate analyses.

---

## CHIP calling in MVP Genomes 
To optimize Mutect2 analysis for our non-tumor/normal samples, we consulted with Alex and team to acquire necessary reference files and establish the optimal running mode. Mutect2 has multiple modes in which it can be run depending on the research question and the type of samples available.  

* tumor-normal mode : A tumor sample is matched with a normal sample in analysis
* tumor-only mode : A single sample's alignment data undergoes analysis
* mitochondrial mode : Where sensitive calling at high depths is desirable

Given the MVP genomes do not have paired tumor samples, our analysis will utilize the tumor-only mode in Mutect2 for CHIP calling.

Mutect2 is a part of the GATK suite of tools and its architecture inherently supports input files in the CRAM format, eliminating the need for pre-processing file conversions. Direct CRAM handling streamlines workflow, reducing computational overhead and resource utilization. 

Following our meeting yesterday, we have received the list of CHIP driver genes and a link to download the germline resource file from the Broad Institute. These resources are key and will be incorporated into the Mutect2 pipeline.  

### Steps involved:

##### 1: Run Mutect2 on MVP CRAMs 

Example script for tumor-only mode:
```shell
 gatk Mutect2 \
  -I sample.bam \
  -L CHIP_driver_genes.bed \
  -R reference.fa \
  --germline-resource germline-resource-broad.hg38.vcf.gz \
  -O sample-output.vcf.gz 
```

##### 2: Filtering the VCFs 
To enhance the quality of the variants for downstream analysis, a filtering process is employed using `bcftools filter`. This process eliminates variants that exhibit insufficient sequencing depth (minimum read depth, DP < 20) or those exclusively detected on either the forward or reverse strand. The resulting output consists of a more compact, bgzipped VCF file, accompanied by its corresponding index. 

##### 3: Merging the VCFs ahead of annotation
This step streamlines the downstream analysis by consolidating multiple variant calls into a unified dataset using `bcftools merge`. Alex and team performed this in batches on 500, and then merged the resulting files into one large VCF. The resulting file is a large bzipped VCF.  

##### 4: Annotating merged VCF
The merged VCF was annotated using `Annovar`, by Alex and team. Once we have the merged VCF for the initial run, subsequent discussions will be held to identify the best course of action to annotate the merged VCF. 

### Plan of action: 
* Dockerize Mutect2 - do we already have a GATK docker image?
* Initiate Mutect2 pipeline for 10 CRAMs on GCP
* Perform cost estimation and machine type comparison
* Get results validated by team to ensure accuracy remaining samples 

---

## SV calling in MVP Genomes

I met with the Argonne team yesterday for the first time to understand their work on structural variant calling and how they wanted to collaborate with the VA/Stanford team. 

To summarize: 

- The Argonne team has been working on developing a deep learning model based tool called [Cue](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10152467/) to call structural variants from WGS data. 
- They have also been working with Nvidia to accelerate the initial pre-processing steps involved in SV calling through their suite of tools.  
- Their aim is to find a way to implement both the standard methods ([Parliament2](https://academic.oup.com/gigascience/article/9/12/giaa145/6042728)) as well the deep learning method (Cue) on the VA approved platform, GCP. 
- We would need to have more in depth conversations with them to understand how we can make this feasible and also discuss the timeline, project scope and onus for key components.  

---

Join the discussion [here!](https://github.com/orgs/va-big-data-genomics/discussions/42)
