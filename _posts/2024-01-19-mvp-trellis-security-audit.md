---
layout: post
title:  "#40 Trellis Security Audit"
date:   2024-01-19 09:00:00 -0800
author: Daniel Cotter
categories: jekyll update
---

# How Trellis uses dsub
As part of a review of Million Veteran Program data security practices, our
team at Stanford wanted to verify that data within Google Cloud Platform was
encrypted at all times, meaning both at rest and in transit.

MVP uses Trellis, "a cloud-based data management framework ... that uses a
graph database to automatically track data and coordinate jobs," to handle
the processing of genomic data after it is deposited in a Google Cloud Storage
(GCS) bucket by the sequencing company. Here, "cloud-based" means both that
the application interacts with data in the cloud, but also more broadly that it
lives in the cloud:

> The application follows a microservice architecture where each service is
> implemented as a serverless function and the state of the system is tracked
> using metadata stored in a Neo4j property graph database.

Another way of saying this is that Trellis is a distributed application,
meaning it does not run as a process on a single server or even a fixed number
of servers. Although this complicates the application architecture, this allows
Trellis to use "cloud-native technologies optimized for massively parallel
operations" and "automatically scale to meet the demands of large-scale
analyses." The combination of a distributed architecture and the use of 

As an aside, Trellis is both an application and a model for a type of
application. At Stanford, it is implemented on Google Cloud Platform (GCP)
services, but it could just as well be built with analogous services from
another cloud provider or even in a private on-premises cloud.

In any case, Trellis as implemented at Stanford uses several Google Cloud
Platform (GCP) services:
- Cloud Functions - to implement application logic
- Cloud Storage - to store data
- Container Registry - to store Docker images used by `dsub`
- Pub/Sub - to relay messages between stateless cloud functions

In our security review, the first question we asked was whether data was
encrypted when being transferred between two cloud services. We received
assurance from Google that data exchanged between GCP services was encrypted
in transit. However, we knew that within Trellis much of the actual processing
of the data is handled by open-source bioinformatics utilities, which are in
turn coordinated by an open-source job scheduler, `dsub`.

[dsub](https://github.com/DataBiosphere/dsub) is a "tool that makes it easy to submit and run batch scripts in the cloud
... with Docker." We wanted to verify that `dsub` handled data securely as
well. Fortunately, `dsub` is written to be used with Google Cloud Storage, and
all exchanges of data within `dsub` are handled by `gsutil`:

> With dsub, your input files reside in a Google Cloud Storage bucket and your
> output files will also be copied out to Cloud Storage. When you submit a job
> with dsub:
> * Your input files will be automatically copied from bucket paths to local
>   disk.
> * Your code will work on the local file system inside the Docker container.
> * Your output files will be automatically copied from local disk back to
>   bucket paths.

I was only familiar with Trellis's architecture at the high level described in
the [paper](paper.md) introducing it and still had a few questions about how `dsub` was
being used:
- Which Storage buckets will `dsub` be directed by Trellis to use?
- Where are the Docker images `dsub` uses located?
- How do the programs `dsub` is calling get loaded onto the Docker images?

I searched the code repository for `dsub` and found it was called from a number
of cloud functions:
  + [launch-cnvnator](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/8c6054e3e367c227accf2f6ac9fed4942c67cff4/functions/launch-cnvnator/main.py#L141)
  + [launch-text-to-table](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/8c6054e3e367c227accf2f6ac9fed4942c67cff4/functions/launch-text-to-table/main.py#L108)
  + [launch-vcfstats](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/8c6054e3e367c227accf2f6ac9fed4942c67cff4/functions/launch-vcfstats/main.py#L66)
  + [launch-bam-fastqc](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/8c6054e3e367c227accf2f6ac9fed4942c67cff4/functions/launch-bam-fastqc/main.py#L66)
  + [launch-fastq-to-ubam](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/8c6054e3e367c227accf2f6ac9fed4942c67cff4/functions/launch-fastq-to-ubam/main.py#L66)
  + [launch-view-gvcf-snps](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/8c6054e3e367c227accf2f6ac9fed4942c67cff4/functions/launch-view-gvcf-snps/main.py#L169)
  + [launch-flagstat](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/8c6054e3e367c227accf2f6ac9fed4942c67cff4/functions/launch-flagstat/main.py#L66)
  + [launch-gatk-5-dollar](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/8c6054e3e367c227accf2f6ac9fed4942c67cff4/functions/launch-gatk-5-dollar/main.py#L71)

All of these cloud functions follow a very similar pattern, so I will use
[launch-cnvnator](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/8c6054e3e367c227accf2f6ac9fed4942c67cff4/functions/launch-cnvnator/main.py) as a representative example for this blog post. Lots of code
below, so feel free to skim, but I wanted to include it for documentation &
future use. At line 141, there is a function called `launch_dsub_task(dsub_args)`:

```python
def launch_dsub_task(dsub_args):
    try:
        result = dsub.dsub_main('dsub', dsub_args)
    ...
    return(result)
```

This function is called exactly once, at line 299:

```python
dsub_result = launch_dsub_task(dsub_args)
```

The parameter `dsub_args` is a key-value dictionary defined just previously, at
line 253:

```python
dsub_args = [
    "--name", f"{task_name}-{job_dict['inputHash'][0:5]}",
    ...
    "--image", job_dict["image"],
    ...
    "--script", job_dict["script"],
    "--input-recursive", job_dict["inputRecursive"]
]
```

The key-value dictionary `job_dict`, in turn, is defined at line 213, from
environment variables, hardcoded values, and local variables (which are
themselves derived from the Pub/Sub event and context passed into the cloud
function):

```python
job_dict = {
    ...
    "image": f"gcr.io/{TRELLIS.GOOGLE_CLOUD_PROJECT}/clinicalgenomics/cnvnator:0.4.1",
    ...
    "script": f"gs://{TRELLIS.TRELLIS_BUCKET}/functions/{FUNCTION_NAME}/CNVnator.sh",
    ...
    "inputs": {
        "BAM": f"gs://{cram['bucket']}/{cram['path']}",
        # Trying to resolve an issue using CRAMs(?): https://github.com/DecodeGenetics/graphtyper/issues/57
        "REF_CACHE_SOURCE": "gs://gcp-public-data--broad-references/hg38/v0/Homo_sapiens_assembly38.ref_cache.tar.gz"
    },
    "inputRecursive": f"DIR=gs://{TRELLIS.GOOGLE_CLOUD_PROJECT}-genomics-public-data/references/GRCh38/unzipped",
    "outputs": {
        "ROOT": f"gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}.root",
        "CALL_OUT": f"gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}.out",
        "EVAL_OUT": f"gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}.txt",
        "CALL_VCF": f"gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}.vcf",
        "GENOTYPE_OUT": f"gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}_genotype.out"
    },
    ...
    "sample": sample,
    "plate": plate,
    "name": task_name,
    ...
}
```

I have pared down the code to the entries we care about for the security
review:
- `image`: the Docker image `dsub` is using
- `script`: the script it is running within the image (note, however, that some
  cloud functions use `command` instead of `script`, although they are similar
  in what they imply).
- `inputs`: The input file(s) to the script
- `outputs`: The output files(s) from the script
  
The values of these variables are parameterized by the variables:
- `TRELLIS.GOOGLE_CLOUD_PROJECT`
- `TRELLIS.TRELLIS_BUCKET`
- `TRELLIS.DSUB_OUT_BUCKET`
- `FUNCTION_NAME`

The first three are defined in a YAML configuration file stored in the Trellis
bucket in Cloud Storage and the last by an environment variable provided by the
Cloud Function service. The location of the configuration file, in turn, is
specified by the environment variables `CREDENTIALS_BUCKET` and
`CREDENTIALS_BLOB`, which are set by the developer on the Variables tab of the
cloud function itself and are not publicly shared. These variables are set
between lines 26 and 42 of our `launch-cnvnator` function:

```python
ENVIRONMENT = os.environ.get('ENVIRONMENT', '')
...
if ENVIRONMENT == 'google-cloud':
    FUNCTION_NAME = os.environ['FUNCTION_NAME']

    vars_blob = storage.Client() \
                .get_bucket(os.environ['CREDENTIALS_BUCKET']) \
                .get_blob(os.environ['CREDENTIALS_BLOB']) \
                .download_as_string()
    parsed_vars = yaml.load(vars_blob, Loader=yaml.Loader)
    TRELLIS = Struct(**parsed_vars)

    PUBLISHER = pubsub.PublisherClient()
    CLIENT = storage.Client()
```

The other variables `job_dict` uses are set from lines 168-209 in the function
`launch_cnvnator` and come from the event and context passed to the function by
Trellis through the Pub/Sub (i.e. "publish–subscribe") service.

```python
def launch_cnvnator(event, context, test=False):
    """When an object node is added to the database, launch any
       jobs corresponding to that node label.

       Args:
            event (dict): Event payload.
            context (google.cloud.functions.Context): Metadata for the event.
    """

    # Parse message
    message = TrellisMessage(event, context)
    cram = message.results['cram']
    ...

    # Optional fields
    study = message.results.get('study')
    hospitalized = message.results.get('hospitalized')
    recvdActureCare = message.results.get('recvdActureCare')
    stayedInIcu = message.results.get('stayedInIcu')

    ...

    # Create unique task ID
    datetime_stamp = get_datetime_stamp()
    task_id, trunc_nodes_hash = make_unique_task_id([cram], datetime_stamp)

    # Database entry variables
    plate = cram['plate']
    sample = cram['sample']
    basename = cram['basename']

    study_metadata_path = f"study{study}/hospitalized{hospitalized}/recvdActureCare{recvdActureCare}/stayedInIcu{stayedInIcu}"

    task_name = 'cnvnator'
```

The entirety of `cnvnator.sh`, by the way, is as follows:

```bash
#!/bin/bash 
#CNVnator BASH Script

# https://github.com/DecodeGenetics/graphtyper/issues/57
echo "Untarring reference cache source"
tar xzf ${REF_CACHE_SOURCE} -C ${REF_CACHE_SOURCE%/*}

echo "Specifying paths to reference caches"
export REF_PATH="${REF_CACHE_SOURCE%/*}/ref/cache/%2s/%2s/%s:http://www.ebi.ac.uk/ena/cram/md5/%s"
export REF_CACHE="${REF_CACHE_SOURCE%/*}/ref/cache/%2s/%2s/%s"

echo "Running CNVnator"
cnvnator -unique -root ${ROOT} -tree ${BAM} -chrom $(seq -f 'chr%g' 1 22) chrX chrY chrM 
cnvnator -root ${ROOT} -his ${BIN_SIZE} -d ${DIR} 
cnvnator -root ${ROOT} -stat ${BIN_SIZE} 
cnvnator -root ${ROOT} -eval ${BIN_SIZE} > ${EVAL_OUT} 
cnvnator -root ${ROOT} -partition ${BIN_SIZE} 
cnvnator -root ${ROOT} -call ${BIN_SIZE} > ${CALL_OUT} 
perl /app/CNVnator_v0.4.1/src/cnvnator2VCF.pl ${CALL_OUT} ${DIR} > ${CALL_VCF} 
awk '{print $2} END {print "exit"}' ${CALL_OUT} | cnvnator -root ${ROOT} -genotype ${BIN_SIZE} > ${GENOTYPE_OUT}
```

The environment variables used in that script are passed in to the Docker image
from the arguments passed to `dsub` in `dsub_args`.

I included what may be a tedious amount of detail above because there are so
many steps involved in tracing back the location of the image, script, inputs,
and outputs. For the security review, however, I summarized my findings in the
following table:

**dsub inputs & outputs**

| Function              | Storage URL                                                                                                              | input / output | notes       | from trellis.yaml**                                            | from pub/sub msg***                          | hardcoded |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------|----------------|-------------|----------------------------------------------------------------|----------------------------------------------|-----------|
| launch-bam-fastqc     | gs://{bucket}/{path}                                                                                                     | input          |             |                                                                | bucket, path                                 |           |
| launch-bam-fastqc     | gs://{OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{basename}.fastqc.data.txt                               | output         |             | OUT_BUCKET                                                     | plate, sample, task_id, basename             | task_name |
| launch-cnvnator       | gs://{cram['bucket']}/{cram['path']}                                                                                     | input          |             |                                                                | cram                                         |           |
| launch-cnvnator       | gs://gcp-public-data--broad-references/hg38/v0/Homo_sapiens_assembly38.ref_cache.tar.gz                                  | input          | public data |                                                                |                                              |           |
| launch-cnvnator       | gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}.root         | output         |             | TRELLIS.DSUB_OUT_BUCKET                                        | plate, sample, task_id, study_metadata_path  | task_name |
| launch-cnvnator       | gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}.out          | output         |             | TRELLIS.DSUB_OUT_BUCKET                                        | plate, sample, task_id, study_metadata_path  | task_name |
| launch-cnvnator       | gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}.txt          | output         |             | TRELLIS.DSUB_OUT_BUCKET                                        | plate, sample, task_id, study_metadata_path  | task_name |
| launch-cnvnator       | gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}.vcf          | output         |             | TRELLIS.DSUB_OUT_BUCKET                                        | plate, sample, task_id, study_metadata_path  | task_name |
| launch-cnvnator       | gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{study_metadata_path}/{sample}_genotype.out | output         |             | TRELLIS.DSUB_OUT_BUCKET                                        | plate, sample, task_id, study_metadata_path  | task_name |
| launch-fastq-to-ubam  | gs://{bucket}/{path}                                                                                                     | input          |             |                                                                | bucket, path                                 |           |
| launch-fastq-to-ubam  | gs://{OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{sample}_{read_group}.ubam                               | output         |             |                                                                | plate, sample, task_id, read_group           | task_name |
| launch-flagstat       | gs://{bucket}/{path}                                                                                                     | input          |             |                                                                | bucket, path                                 |           |
| launch-flagstat       | gs://{OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{basename}.flagstat.data.tsv                             | output         |             | OUT_BUCKET                                                     | plate, sample, task_id, basename             | task_name |
| launch-gatk-5-dollar  | gs://{TRELLIS_BUCKET}/{GATK_MVP_DIR}/{GATK_MVP_HASH}/{GATK_GERMLINE_DIR}/google-adc.conf                                 | input          |             | TRELLIS_BUCKET, GATK_MVP_DIR, GATK_MVP_HASH, GATK_GERMLINE_DIR |                                              |           |
| launch-gatk-5-dollar  | gs://{OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/inputs/{sample}.google-papi.options.json                        | input          |             | OUT_BUCKET                                                     | plate, sample, task_id, sample               | task_name |
| launch-gatk-5-dollar  | gs://{TRELLIS_BUCKET}/{GATK_MVP_DIR}/{GATK_MVP_HASH}/{GATK_GERMLINE_DIR}/fc_germline_single_sample_workflow.wdl          | input          |             | TRELLIS_BUCKET, GATK_MVP_DIR, GATK_MVP_HASH, GATK_GERMLINE_DIR |                                              |           |
| launch-gatk-5-dollar  | gs://{TRELLIS_BUCKET}/{GATK_MVP_DIR}/{GATK_MVP_HASH}/{GATK_GERMLINE_DIR}/tasks_pipelines/*.wdl                           | input          |             | TRELLIS_BUCKET, GATK_MVP_DIR, GATK_MVP_HASH, GATK_GERMLINE_DIR |                                              |           |
| launch-gatk-5-dollar  | gs://{OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/inputs/inputs.json                                              | input          |             | OUT_BUCKET                                                     | plate, sample, task_id, sample               | task_name |
| launch-gatk-5-dollar  | gs://{OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output                                                          | output         |             | OUT_BUCKET                                                     | plate, sample, task_id, sample               | task_name |
| launch-text-to-table  | gs://{bucket}/{path}                                                                                                     | input          |             |                                                                | bucket, path                                 |           |
| launch-text-to-table  | gs://{OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{task_group}/{basename}.csv                              | output         |             | OUT_BUCKET                                                     | plate, sample, task_id, task_group, basename |           |
| launch-vcfstats       | gs://{bucket}/{path}                                                                                                     | input          |             |                                                                | bucket, path                                 |           |
| launch-vcfstats       | gs://{OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{sample}.rtg.vcfstats.data.txt                           | output         |             | OUT_BUCKET                                                     | plate, sample, task_id                       | task_name |
| launch-view-gvcf-snps | gs://{vcf['bucket']}/{vcf['path']}                                                                                       | input          |             |                                                                | vcf['bucket'], vcf['path']                   |           |
| launch-view-gvcf-snps | gs://{index['bucket']}/{index['path']}                                                                                   | input          |             |                                                                | index['bucket'], index['path']               |           |
| launch-view-gvcf-snps | SIGNATURE_SNPS                                                                                                           | input          |             |                                                                |                                              |           |
| launch-view-gvcf-snps | REF_FASTA                                                                                                                | input          | public data |                                                                |                                              |           |
| launch-view-gvcf-snps | REF_FASTA_INDEX                                                                                                          | input          | public data |                                                                |                                              |           |
| launch-view-gvcf-snps | gs://{TRELLIS.DSUB_OUT_BUCKET}/{plate}/{sample}/{task_name}/{task_id}/output/{sample}.signatureSNPs.vcf.gz               | output         |             | TRELLIS.DSUB_OUT_BUCKET                                        | plate, sample, task_id                       | task_name |

And:

**dsub images & scripts**

| Function              | dsub Docker image                                                      | Script / command                                                                                                                                                                                                                |
|-----------------------|------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| launch-bam-fastqc     | gcr.io/XXXXXXX/biocontainers/fastqc:v0.11.5_cv4                        | fastqc.sh                                                                                                                                                                                                                       |
| launch-cnvnator       | gcr.io/XXXXXXX/biocontainer/clinicalgenomics/cnvnator:0.4.1            | gs://XXXXXXX/functions/launch-cnvnator/CNVnator.sh                                                                                                                                                                              |
| launch-fastq-to-ubam  | gcr.io/XXXXXXX/biocontainer/broadinstitute/gatk:4.1.0.0                | /gatk/gatk --java-options '-Xmx8G -Djava.io.tmpdir=bla' FastqToSam -F1 ${FASTQ_1} -F2 ${FASTQ_2} -O ${UBAM} -RG ${RG} -SM ${SM} -PL ${PL}                                                                                       |
| launch-flagstat       | gcr.io/XXXXXXX/biocontainer/biocontainers/samtools:v1.9-4-deb_cv1      | samtools flagstat ${INPUT} > ${OUTPUT}                                                                                                                                                                                          |
| launch-gatk-5-dollar  | gcr.io/XXXXXXX/biocontainer/broadinstitute/cromwell:53                 | java -Dconfig.file=${CFG} -Dbackend.providers.${BACKEND_PROVIDER}.config.project=${PROJECT} -Dbackend.providers.${BACKEND_PROVIDER}.config.root=${ROOT} -jar /app/cromwell.jar run ${WDL} --inputs ${INPUT} --options ${OPTION} |
| launch-text-to-table  | gcr.io/XXXXXXX/biocontainer/stanfordbioinformatics/text-to-table:0.2.1 | text2table -s ${SCHEMA} -o ${OUTPUT} -v series=${SERIES},sample=${SAMPLE_ID} ${INPUT}                                                                                                                                           |
| launch-vcfstats       | gcr.io/XXXXXXX/biocontainer/realtimegenomics/rtg-tools:3.7.1           | rtg vcfstats ${INPUT} > ${OUTPUT}                                                                                                                                                                                               |
| launch-view-gvcf-snps | gcr.io/XXXXXXX/biocontainer/bschiffthaler/bcftools:1.11                | bcftools view ${VCF} -R ${SNP_LIST} -Ou \| bcftools convert --gvcf2vcf --fasta-ref ${REF_FASTA} -Ou \| bcftools view -T ${SNP_LIST} -Oz -o ${OUTPUT}                                                                            |

In the end, what we found is that Trellis uses only one bucket for input
and one for output and both are within the MVP Google Cloud project, meaning
they are encrypted at rest and in transit between GCP services.

### Other concerns

- Trellis also uses Cromwell, another pipeline manager, "because the $5 GATK
  pipeline that we used for variant calling was already defined in Cromwell’s
  native Workflow Definition Language (WDL) and the pipeline had already been
  optimized to run on Google Cloud using Cromwell."
- Trellis also uses Neo4j, a graph database, to track the state of the system;
  however, Neo4j does not store any genomics data, only metadata relating to
  the job and data files, so it is not subject to the same scrutiny as the GCP
  components of Trellis. 

### References
[Ross, P.B., Song, J., Tsao, P.S. et al. Trellis for efficient data and task management in the VA Million Veteran Program. Sci Rep 11, 23229 (2021). https://doi.org/10.1038/s41598-021-02569-5](https://www.nature.com/articles/s41598-021-02569-5)
