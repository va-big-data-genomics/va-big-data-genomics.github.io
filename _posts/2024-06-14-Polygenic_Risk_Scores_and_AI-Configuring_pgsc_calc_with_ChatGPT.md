# Polygenic Risk Scores & AI: Configuring pgsc_calc with ChatGPT
Recently, Rodrigo introduced our group to a software for calculating polygenic risk scores, pgsc_calc. I developed an interest in learning more about this software and running it on our Google Cloud architecture. My goal is to make this feature available to Rodrigo and the rest of our group for use with our data.

ChatGTP is an AI designed to respond to user input, typically in a chat box. [Our internal group at Stanford](https://med.stanford.edu/gbsc/scgpm-team.html) has shown increasing interest in the applications of ChatGTP in our field. Owing to the challenges associated with installing bioinformatics software, I wanted to leverage ChatGTP to get pgsc_calc operational. The following text details the results of my discussions with ChatGTP.

## Installation and Test Guide for PGSC_Calc with Nextflow on MacOS

### Install Java on MacOS

First, check if Java is already installed:

java -version

If Java is already installed, this command will show the current Java version.

If Java is not installed:

- Visit [Oracle's Java SE Downloads Page](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)
- Download the package relevant to MacOS
- After downloading, install Java by clicking on the downloaded file and following the instructions

### Install Nextflow

Nextflow requires Unix-like operating system (Linux, Solaris, OS X, etc) and bash shell.

Confirm that Java v8 or later is installed on your computer:

java -version

Enter the following command to download and install Nextflow:

curl -s https://get.nextflow.io | bash

This will create the Nextflow main executable file in the current directory. Optionally, move the Nextflow binary to a folder that's included in your `$PATH`:

echo $PATH

If `/usr/local/bin/` is included in the output, move Nextflow to `/usr/local/bin/`:

sudo mv nextflow /usr/local/bin/

Verify the installation using:

nextflow run hello.

### Install Docker on MacOS

Docker can be installed on MacOS using Docker Desktop.

- Visit [Docker's Desktop for Mac installation page](https://docs.docker.com/desktop/mac/install/)
- Download the Docker Desktop installer
- Open the downloaded Docker.dmg file and drag the Docker app to the Applications folder. Double click Docker.app to start Docker Desktop
- Confirm Docker Desktop is running properly by clicking on the Docker icon in your menu bar. It should show "Docker Desktop is running"

### Running PGSC_Calc Pipeline Locally

Once Nextflow and Docker are successfully installed on your machine, you can run the PGSC_Calc pipeline locally using Docker.

To simply test the installation using the PGSC_Calc's provided test profile, use the following Nextflow command:

nextflow run pgscatalog/pgsc_calc -profile test,docker

Please replace or adjust command parameters based on your specific data and requirements.

This guide should suffice to get you started running `pgsc_calc` with Nextflow and Docker on a MacOS. Remember to handle your Docker credentials securely, if you're working with private Docker images, and always manage your data securely.

### Comments on running pgsc_calc locally

In the text above I set up my workstation to use a public Docker image of pgsc_calc. The steps provided included installing all necessary dependencies. This includes nextflow, which provides a simple way to run dockerized containers in environments such as Google Cloud. The [git repo](https://github.com/PGScatalog/pgsc_calc) containing the public docker image also includes the two files necessary to run Nextflow - main.nf and nextflow.config. These two files represent the script to run the pipeline and the configuration file containing information on how to run it, respectively.
In running this, I also added a local nextflow.config file to allow output data to be exported to my local machine. When running nextflow, it will prioritize the local configuration file, bypassing the one provided on the git repo. The configuration file contains one specification that provides the output path on my local machine. The file I created looks like this:
```
Replace Google Cloud Storage bucket with local directory path
workDir = '/Users/jsarro/pgsc-calc-pipeline/pgsc-calc-pipeline/local_output
```
The test run was successful using the provided test input. I next wanted to see if I could get pgsc_calc to run on Google Cloud through my workstation commands. I found there are three options for this; run on a virtual machine, run using Kubernetes, and run using Google Life Sciences. Because we are moving away from virtual machines due to FedRAMP restrictions, Kubernetes would be the best option moving forward. While Daniel works on getting Kubernetes configured, I wanted to see if I could run the test data using Google Life Sciences and then run a test using our own data. The following are the two options I worked out with ChatGPT for using either a public docker image (like above) or creating your own docker image.

## Configuring and Running PGSC-CALC in Nextflow Using Google Life Sciences

The container image in the Nextflow configuration specifies the Docker image that will be used to run the processes in your workflow. This image should include all the necessary tools and dependencies required by the PGSC-CALC pipeline.

For PGSC-CALC, you can either use a publicly available image that contains the necessary tools or build your own Docker image with the required software. Here is how you can proceed with each option:

### Option 1: Use a Publicly Available Image

If there is a publicly available Docker image that fits your needs, you can specify it directly in your `nextflow.config`. For example:

```groovy
process {
    executor = 'google-lifesciences'
    container = 'broadinstitute/gatk:4.1.8.1' // Replace with an appropriate image
}

google {
    project = 'YOUR_PROJECT'
    region = 'YOUR_REGION'
}

workDir = 'gs://YOUR_BUCKET_NAME/work'
```

### Option 2: Build Your Own Docker Image

If you need a custom Docker image with specific tools and dependencies, you can create a Dockerfile and build your own image. Here’s an example Dockerfile for PGSC-CALC:

```Dockerfile
## Use a base image with Ubuntu
FROM ubuntu:20.04

## Install required packages
RUN apt-get update && apt-get install -y \
    openjdk-11-jdk \
    python3 \
    python3-pip \
    git \
    curl \
    && apt-get clean

## Install Nextflow
RUN curl -s https://get.nextflow.io | bash \
    && mv nextflow /usr/local/bin/

## Install other dependencies for PGSC-CALC
## Add specific installation commands for the tools required by PGSC-CALC

## Set the default command
CMD ["nextflow", "-version"]
```

Build and push your Docker image to Google Container Registry:

- Build the Docker Image:

  ```bash
  docker build -t gcr.io/YOUR_PROJECT_ID/pgsc-calc:latest .
  ```

- Push the Docker Image:

  ```bash
  docker push gcr.io/YOUR_PROJECT_ID/pgsc-calc:latest
  ```

### Update Your Nextflow Configuration

After building or choosing the Docker image, update your `nextflow.config` to use this image:

```groovy
process {
    executor = 'google-lifesciences'
    container = 'gcr.io/YOUR_PROJECT_ID/pgsc-calc:latest'
}

google {
    project = 'YOUR_PROJECT'
    region = 'YOUR_REGION'
}

workDir = 'gs://YOUR_BUCKET_NAME/'
```

### Example of Running PGSC-CALC

After setting up the container image, you can run the PGSC-CALC workflow with Nextflow:

```bash
nextflow run pgsc-calc.nf -params-file params.input
```

### Summary

- **Container Image:** This is a Docker image that contains all the tools and dependencies required to run your workflow.
- **Public Image:** Use a pre-built public image if it meets your needs.
- **Custom Image:** Build your own Docker image if specific tools or versions are required.
- **Nextflow Configuration:** Update `nextflow.config` to specify the container image and Google Cloud settings.

By following these steps, you should be able to configure and run Nextflow with PGSC-CALC using Google Life Sciences.

### Comments on Running pgsc_calc in the Cloud

I tried using both of the above methods. I became interested in generating a custom Docker image for two reasons: First, in case we needed to modify any configurations, and second, to mitigate potential issues with FedRAMP when using a public Docker image. I discovered that some of the configurations required updates for my local system. Below are some of the changes I made with ChatGPT.

## Clone the pgs_calc Repository

```bash
git clone https://github.com/PGScatalog/pgsc_calc.git
cd pgsc_calc
```

### Create a Dockerfile
Create a Dockerfile in the `pgsc_calc` directory with the following content:

```Dockerfile
## Use the official Ubuntu base image
FROM ubuntu:latest

## Install required system dependencies
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    python3-venv \
    git \
    && apt-get clean

## Set the working directory
WORKDIR /app

## Clone the PGSC-Calc repository
RUN git clone https://github.com/PGScatalog/pgsc_calc.git /app/pgsc_calc

## Create a virtual environment and activate it
RUN python3 -m venv /app/venv
ENV PATH="/app/venv/bin:$PATH"

## Copy requirements.txt from the cloned repository into the Docker image
COPY pgsc_calc/requirements.txt /app/pgsc_calc/requirements.txt

## Install Python packages specified in requirements.txt within the virtual environment
RUN pip3 install --no-cache-dir -r /app/pgsc_calc/requirements.txt

## Make port 80 available to the world outside this container
EXPOSE 80

## Command to run the application
CMD ["/bin/bash"]
```

### Build the Docker Image

```bash
docker build -t my_nextflow_image .
```

### Set Up Google Cloud SDK and Authenticate

- Install Google Cloud SDK: Follow the instructions on Google Cloud SDK installation.
- Authenticate with Google Cloud:

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT
```

### Modify Nextflow Configuration to Use Google Cloud Storage
Update your nextflow.config file with the following content:

```groovy
process {
    executor = 'google-lifesciences'
    container = 'my_nextflow_image' // Use your Docker image
}

google {
    project = 'YOUR_PROJECT'
    region = 'YOUR_REGION'
}

workDir = 'gs://YOUR_DIRECTORY'
```
### Create the Nextflow Workflow Script
Create a `main.nf` file in your project directory with the following content:

```groovy
#!/usr/bin/env nextflow

params.test_file = '/app/test_file'
params.output_dir = 'gs://YOUR_BUCKET'

process pgs_calc {
    container 'my_nextflow_image'
    
    input:
    path test_file from params.test_file

    output:
    path '**' into results

    script:
    """
    cd /app/pgsc_calc
    python3 conftest.py ${test_file}
    """
}

results.view { it }
```

### Run Nextflow
Execute the Nextflow workflow:

```bash
nextflow run main.nf -c nextflow.config
```

This configuration will allow you to run the `pgs_calc` application using Nextflow, with outputs written to your specified Google Cloud Storage bucket.

## Final comments and future work
I was not able to run pgsc_calc successfully on the Cloud using Google Life Sciences with either the public docker image or a custom-made one. I determined the issue lay in passing my Docker login credentials to Google Life Sciences. I tried storing them in secrets, but my permissions did not allow me to store them for the Google Life Sciences service account. The option of storing them somewhere else remained, but it introduced too many security risks.

## Ancestry prediction using the PGScatalog

One of my main interests in using pgsc_calc was testing its ability to predict ancestry in our WGS dataset. I found that this could be achieved without running the pgsc_calc program itself. pgsc_calc uses an ancestry prediction script within [PGScatalog-utils](https://github.com/PGScatalog/pgscatalog_utils). The documentation for this tool is limited, but I have been able to install it locally and have begun looking at how it works. Alternatively, the same group provides another ancestry prediction method in its PCA analysis tool, [Fraposa](https://github.com/PGScatalog/fraposa_pgsc). This tool seems like it would suit our needs better, as it performs both PCA and ancestry prediction, both of which are values our group requires. This ancestry prediction tool could also be a better option, as it is designed to handle genotype data from large genomics datasets. In the near future, I would like to test both of these methods alongside [Hail's methodology](https://hail.is/docs/0.2/methods/relatedness.html) on a portion of our dataset.
