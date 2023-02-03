# Dataproc, Docker, and Hail - oh my!

This post is a quick rundown of some computing tools we're using to do genomic analysis.

## Dataproc

### What is it?
A managed Apache Hadoop and Spark service. Dataproc is a service on Google Cloud Platform that combines the two services seamlessly and provides a number of resources for managing the cluster as well.

**Apache Hadoop** is a framework for running MapReduce jobs. MapReduce jobs are computational jobs that consist of *mapping* operations, in which one output is computed from one input, and *reduction* operations, in which one output is computed from many inputs. An example is the sum of squares of a series of numbers:

$$ y = x_1^2 + x_2^2 + ... + x_n^2 $$

Each $x_i$ is used as the input to a squaring function in the mapping phase, then the squares are added together in the reduction phase. (It is called "reduction," by the way, because it reduces multiple numbers to one. "Aggregation" is a synonym).

It doesn't sound very relevant to the sort of computing we do, but it turns out many computing tasks can be recast as a combination of mapping and reducing operations. This model is ideally suited to distributed computing on big data: The data can be split up into a large number of subsets and distributed to "worker" computers, which each do some mapping operation independently before sending the results back to a "master" computer that does the reduction operation after getting the results back from the workers, then the cycle repeats.

**Apache Spark** is another distributed computation framework that addresses some of the deficiencies of Hadoop, among which are a) the clunkiness of rewriting an algorithm as a series of map-reduce steps (I had to do this once for a college course, and it was painful to do and extremely awkward to read compared to the original algorithm) and b) the latency of reading and writing to disk repeatedly. Spark jobs tend to be much less verbose than MapReduce jobs, and Spark is distributed *in memory*, and it can run on top of Hadoop Distributed File System for persistent storage.

## Docker

### What is it?
Docker is a tool for working with *containers*: self-contained, portable, walled-off computing environments.

### Why containers?

Software deployment is complicated. Every developer has run into the "but it works on my machine!" problem. Even a simple script will only run within a certain carefully configured environment (the operating system, the shell, the scripting language, the libraries, packages, and modules, reference data, and so on). To complicate things further, each of these dependencies has its own dependencies, is available in many different versions, and is only compatible with certain versions of the other dependencies. Managing dependencies, environments, and infrastructure is a crucial, but not very glamorous, part of software development.

The naïve way of ensuring a consistent environment is writing deployment scripts that download, install, and configure all of these dependencies. However, this can still go wrong, for instance, if you, as the developer, are not allowed to install or configure the components you need (in shared computing environments, for instance). A better way is to use *virtual environments*, in which the scripting language and all the libraries required are installed in the correct versions in their own directory, but a) this may not work still if some pieces of the environment don't support virtual environments and b) even if they do, the configuration of the environment managers can be complicated and error prone. Speaking of shared computing environments, deploying and running software in such an environment always carries with it the risk that your script may have unintended effects that cause problems for you or others, such as erasing data.

Enter containers: A container is a self-contained, portable computing environment that can be run locally or deployed to a virtual machine in the cloud. This has advantages both for security, scalability, and reproducibility: security, because nothing running in the container can affect the environment outside it, except perhaps indirectly through API calls to a web service; scalability, because the container can be deployed as a unit to an arbitrary number of virtual machines; reproducibility, because the container and/or its build script can be published, downloaded, and run on someone else's computing resources.

As the name suggests, containers are self-contained: All the data required by the container must be inside the container or mapped to it from the host environment. You can "enter" the container and execute shell commands if necessary or just call the program you need to run from outside the container with its parameters specified.

A container has its own version of a deployment script, called a *Dockerfile*, that specifies all the components of the environment mentioned above. This script is used to build an *image* – a blueprint for a container – which is then instantiated as a *container*, which can be used as the basis for a *virtual machine*.

### An example Dockerfile
From the `telseq` repository:
```

FROM ubuntu:14.04
MAINTAINER Zhihao Ding <zhihao.ding@gmail.com>
LABEL Description="Telseq docker" Version="0.0.1"

VOLUME /tmp

WORKDIR /tmp

RUN apt-get update && \
    apt-get install -y \
        automake \
        autotools-dev \
        build-essential \
        cmake \
        libhts-dev \
        libhts0 \
        libjemalloc-dev \
        libsparsehash-dev \
        libz-dev \
        python-matplotlib \
        wget \
        zlib1g-dev

# build remaining dependencies:
# bamtools
RUN mkdir -p /deps && \
    cd /deps && \
    wget https://github.com/pezmaster31/bamtools/archive/v2.4.0.tar.gz && \
    tar -xzvf v2.4.0.tar.gz && \
    rm v2.4.0.tar.gz && \
    cd bamtools-2.4.0 && \
    mkdir build && \
    cd build && \
    cmake .. && \
    make


# build telseq
RUN mkdir -p /src && \
    cd /src && \
    wget https://github.com/zd1/telseq/archive/v0.0.1.tar.gz && \
    tar -xzvf v0.0.1.tar.gz && \
    rm v0.0.1.tar.gz && \
    cd telseq-0.0.1/src && \
    ./autogen.sh && \
    ./configure --with-bamtools=/deps/bamtools-2.4.0 --prefix=/usr/local && \
    make && \
    make install


ENTRYPOINT ["/usr/local/bin/telseq"]
CMD ["--help"]
```

## Hail

### What is it?
A genomics analysis library that can run on Spark and Hadoop for large datasets.

### How are we using it?
The burden testing code that Jina Song produced uses a combination of these techniques to filter the Data Release 1 dataset to those rows with rare variants (allele frequency less than 1%), joins these to data from the Human Genome Diversity Project filtered to alleles influencing height, annotates these rows with variant effect predictor (VEP) information, and filters the rows to those with at least one rare variant matching the VEP information. She then adds height, covariates, and principal componets to the matrix table and runs a linear regression per gene on height.

### Basic elements of Hail
**Data**
- Can be imported from vcf, bgen, plink, tsv, gtf, and bed files and exported 

**MatrixTable**
- One of Hail's main abstractions. It combines multiple axes of a matrix (e.g. variants and samples) with the structured data of tables (e.g. genotypes).

**Rows**
- can be keyed by locus and alleles
- can be filtered by allele frequency (e.g. for rare variants)
- can be annotated

**Aggregation**
- has lots of methods - some generic (min, max, sum, any, all, etc.), others genomics-specific (call_stats, inbreeding, hardy_weinberg_test, etc.)
