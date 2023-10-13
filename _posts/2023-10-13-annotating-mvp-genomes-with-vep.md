### Standalone VEP

I'm also working on running VEP on a standalone Compute VM, i.e. not a Dataproc cluster. To that end, I:

- Created a new Compute VM instance: `dcotter-standalone-vep` with `n1-highmem-32` machine type and a 20 Gb boot disk:
  ```
  gcloud compute instances create dcotter-vep-standalone \
    --project=gbsc-gcp-project-mvp \
    --zone=us-west1-b \
    --machine-type=n1-highmem-8 \
    --network-interface=network=default,network-tier=PREMIUM,stack-type=IPV4_ONLY \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=963911152157-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
    --create-disk=auto-delete=yes,boot=yes,device-name=dcotter-vep-standalone,image=projects/debian-cloud/global/images/debian-11-bullseye-v20230912,mode=rw,size=20,type=projects/gbsc-gcp-project-mvp/zones/us-west1-b/diskTypes/pd-balanced \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --labels=goog-ec-src=vm_add-gcloud \
    --reservation-affinity=any
  ```
- Created an additional 500 Gb data disk:
  ```
      gcloud compute disks create data-disk \
        --size=500GB \
        --type=pd-balanced \
        --region=us-west1 \
        --zone=us-west1-b
  ```
- Attached the disk:
  ```
  gcloud compute instances attach-disk dcotter-vep-standalone --disk=data-disk
  ```
- SSH'd into the VM and:
  + Formatted the additional disk (`sdb`) according to [these instructions](https://devopscube.com/mount-extra-disks-on-google-cloud/):
    ```
    sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
    mkdir ~/data
    sudo mount -o discard,defaults /dev/sdb ~/data
    sudo chmod a+rwx ~/data
    sudo cp /etc/fstab /etc/fstab.backup
    echo UUID=`sudo blkid -s UUID -o value /dev/sdb` /home/dlcott2/data ext4 discard,defaults,noatime,nofail 0 2 | sudo tee -a /etc/fstab
    ```
  + Installed Docker according to [official instructions](https://docs.docker.com/engine/install/debian/):
    ```
    # Set up Docker's Apt repository
    # - add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    # - add the repository to Apt sources:
    echo \
      "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
      "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    
    # Install the Docker packages.
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

    # Verify that the installation is successful by running the hello-world image:
    sudo docker run hello-world
    ```
  + Added my user to the Docker group according to [these instructions](https://stackoverflow.com/questions/48957195/how-to-fix-docker-got-permission-denied-issue#48957722):
    ```
    sudo groupadd docker
    sudo usermod -aG docker $USER
    newgrp docker
    docker run hello-world
    ```
  + Downloaded the Docker VEP image:
    ```
    docker pull ensemblorg/ensembl-vep
    ```
  + Downloaded the MVP DR2 VCF file and the VEP cache files and cloned the LOFTEE
    repository onto the additional data disk:
    ```
    # Enter shared directory (which will be mapped to /opt/vep/.vep in the Docker container)
    cd /home/dlcott2/data
    
    # Download MVP DR2 VCF file (VERY large, 237 GB)
    gsutil cp gs://gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/rel2_100k-var.vcf.bgz .
    
    # Download the VEP cache files and unpack them (also large, 20 GB)
    curl --remote-name https://ftp.ensembl.org/pub/release-110/variation/indexed_vep_cache/homo_sapiens_vep_110_GRCh38.tar.gz
    tar xzf homo_sapiens_vep_110_GRCh38.tar.gz
   
    # Clone LOFTEE plugin repo
    git clone https://github.com/konradjk/loftee.git
    ```
  + Started the image interactively and mapped a local directory to a directory
    in the container for file sharing:
    ```
    # To log in as root, add "--user root" to the following:
    docker run --tty --interactive --volume /home/dlcott2/data:/opt/vep/.vep ensemblorg/ensembl-vep
    ```
    - Ran VEP:
      ```
      vep --cache \
          --verbose \
          --input_file  rel2_100k-var.vcf.bgz \
          --output_file rel2_100k-var-vep.tsv \
          --host        useastdb.ensembl.org \
          --plugin      LoF,loftee_path=/opt/vep/.vep/loftee \
          --dir_plugins /opt/vep/.vep/loftee
      ```

The output from VEP:
```
vep@46a76261b877:~/.vep$   vep --cache \
>       --verbose \
>       --input_file  rel2_100k-var.vcf.bgz \
>       --output_file rel2_100k-var-vep.tsv \
>       --host        useastdb.ensembl.org \
>       --plugin      LoF,loftee_path=/opt/vep/.vep/loftee/ \
>       --dir_plugins /opt/vep/.vep/loftee/
Smartmatch is experimental at /opt/vep/src/ensembl-vep/modules/Bio/EnsEMBL/VEP/AnnotationSource/File.pm line 472.
2023-09-22 18:02:31 - Ignored unsupported option 'no_plugins=1' from environment variable VEP_NO_PLUGINS
2023-09-22 18:02:31 - Ignored unsupported option 'pluginsdir=/plugins' from environment variable VEP_PLUGINSDIR
2023-09-22 18:02:31 - Ignored unsupported option 'no_update=1' from environment variable VEP_NO_UPDATE
2023-09-22 18:02:31 - Set 'dir_plugins=/plugins' from environment variable VEP_DIR_PLUGINS
2023-09-22 18:02:31 - Ignored unsupported option 'no_htslib=1' from environment variable VEP_NO_HTSLIB
2023-09-22 18:02:31 - Read configuration from environment variables
Will only load v110 databases
Species 'homo_sapiens' loaded from database 'homo_sapiens_core_110_38'
Species 'homo_sapiens' loaded from database 'homo_sapiens_cdna_110_38'
Species 'homo_sapiens' loaded from database 'homo_sapiens_otherfeatures_110_38'
Species 'homo_sapiens' loaded from database 'homo_sapiens_rnaseq_110_38'
homo_sapiens_variation_110_38 loaded
homo_sapiens_funcgen_110_38 loaded
No ancestral database found
No ontology database found
Use of uninitialized value $taxonomy_db in concatenation (.) or string at /opt/vep/src/ensembl-vep/Bio/EnsEMBL/Registry.pm line 2355.
ensembl_taxonomy API not found - ignoring 
Use of uninitialized value $ensembl_metadata_db in concatenation (.) or string at /opt/vep/src/ensembl-vep/Bio/EnsEMBL/Registry.pm line 2393.
ensembl_metadata API not found - ignoring 
No production database or adaptor found
Smartmatch is experimental at /opt/vep/.vep/loftee/de_novo_donor.pl line 175.
Smartmatch is experimental at /opt/vep/.vep/loftee/de_novo_donor.pl line 214.
Smartmatch is experimental at /opt/vep/.vep/loftee/splice_site_scan.pl line 191.
Smartmatch is experimental at /opt/vep/.vep/loftee/splice_site_scan.pl line 194.
Smartmatch is experimental at /opt/vep/.vep/loftee/splice_site_scan.pl line 238.
Smartmatch is experimental at /opt/vep/.vep/loftee/splice_site_scan.pl line 241.
Use of uninitialized value in string eq at /opt/vep/.vep/loftee/LoF.pm line 348.
WARNING: Failed to instantiate plugin LoF: Can't open /vep/loftee/splice_data/donor_motifs/ese.txt: No such file or directory at /opt/vep/.vep/loftee/loftee_splice_utils.pl line 212.

2023-09-22 18:02:33 - No input file format specified - detected vcf format

gzip: stdout: Broken pipe
```

For one thing, there is the warning message:
```
WARNING: Failed to instantiate plugin LoF: Can't open /vep/loftee/splice_data/donor_motifs/ese.txt: No such file or directory at /opt/vep/.vep/loftee/loftee_splice_utils.pl line 212.
```

This all the output there was from the program; there were no progress indicators. In order to get an idea of what was going on, I ran `ps` and `wc -l rel2_100k-var-vep.tsv`, which file was the specified output. The VPN connection timed out at some point, and the SSH session froze its output, but I don't know whether the Docker container continued to run in the abandoned session or what.

The final output of `wc` was 1.2M lines and 174MB:
```
1200745 rel2_100k-var-vep.tsv
```

The first hundred lines of that file are:
```
## ENSEMBL VARIANT EFFECT PREDICTOR v110.1
## Output produced at 2023-09-22 18:02:33
## Connected to homo_sapiens_core_110_38 on useastdb.ensembl.org
## Using cache in /opt/vep/.vep/homo_sapiens/110_GRCh38
## Using API version 110, DB version 110
## ensembl-funcgen version 110.24e6da6
## ensembl-io version 110.b1a0d57
## ensembl version 110.584a8f3
## ensembl-variation version 110.d34d25e
## regbuild version 1.0
## sift version 6.2.1
## gnomADg version v3.1.2
## HGMD-PUBLIC version 20204
## 1000genomes version phase3
## polyphen version 2.2.3
## gencode version GENCODE 44
## genebuild version 2014-07
## gnomADe version r2.1.1
## assembly version GRCh38.p14
## COSMIC version 97
## dbSNP version 154
## ClinVar version 202301
## Column descriptions:
## Uploaded_variation : Identifier of uploaded variant
## Location : Location of variant in standard coordinate format (chr:start or chr:start-end)
## Allele : The variant allele used to calculate the consequence
## Gene : Stable ID of affected gene
## Feature : Stable ID of feature
## Feature_type : Type of feature - Transcript, RegulatoryFeature or MotifFeature
## Consequence : Consequence type
## cDNA_position : Relative position of base pair in cDNA sequence
## CDS_position : Relative position of base pair in coding sequence
## Protein_position : Relative position of amino acid in protein
## Amino_acids : Reference and variant amino acids
## Codons : Reference and variant codon sequence
## Existing_variation : Identifier(s) of co-located known variants
## Extra column keys:
## IMPACT : Subjective impact classification of consequence type
## DISTANCE : Shortest distance from variant to transcript
## STRAND : Strand of the feature (1/-1)
## FLAGS : Transcript quality flags
## VEP command-line: vep --cache --database 0 --dir_plugins [PATH]/ --input_file rel2_100k-var.vcf.bgz --output_file rel2_100k-var-vep.tsv --plugin [PATH]/ --verbose
#Uploaded_variation	Location	Allele	Gene	Feature	Feature_type	Consequence	cDNA_position	CDS_positioProtein_position	Amino_acids	Codons	Existing_variation	Extra
chr1_13047_C/A	chr1:13047	A	ENSG00000223972	ENST00000450305	Transcript	non_coding_transcript_exon_variant	255	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13047_C/A	chr1:13047	A	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13047_C/A	chr1:13047	A	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1357;STRAND=-1
chr1_13047_C/A	chr1:13047	A	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4322;STRAND=-1
chr1_13052_G/C	chr1:13052	C	ENSG00000223972	ENST00000450305	Transcript	splice_region_variant,non_coding_transcript_exon_variant	260	-	-	-	-	-	IMPACT=LOW;STRAND=1
chr1_13052_G/C	chr1:13052	C	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13052_G/C	chr1:13052	C	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1352;STRAND=-1
chr1_13052_G/C	chr1:13052	C	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4317;STRAND=-1
chr1_13053_G/A	chr1:13053	A	ENSG00000223972	ENST00000450305	Transcript	splice_donor_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=HIGH;STRAND=1
chr1_13053_G/A	chr1:13053	A	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13053_G/A	chr1:13053	A	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1351;STRAND=-1
chr1_13053_G/A	chr1:13053	A	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4316;STRAND=-1
chr1_13053_G/C	chr1:13053	C	ENSG00000223972	ENST00000450305	Transcript	splice_donor_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=HIGH;STRAND=1
chr1_13053_G/C	chr1:13053	C	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13053_G/C	chr1:13053	C	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1351;STRAND=-1
chr1_13053_G/C	chr1:13053	C	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4316;STRAND=-1
chr1_13054_C/A	chr1:13054	A	ENSG00000223972	ENST00000450305	Transcript	splice_donor_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=HIGH;STRAND=1
chr1_13054_C/A	chr1:13054	A	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13054_C/A	chr1:13054	A	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1350;STRAND=-1
chr1_13054_C/A	chr1:13054	A	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4315;STRAND=-1
chr1_13058_C/G	chr1:13058	G	ENSG00000223972	ENST00000450305	Transcript	splice_donor_region_variant,intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=LOW;STRAND=1
chr1_13058_C/G	chr1:13058	G	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13058_C/G	chr1:13058	G	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1346;STRAND=-1
chr1_13058_C/G	chr1:13058	G	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4311;STRAND=-1
chr1_13073_C/A	chr1:13073	A	ENSG00000223972	ENST00000450305	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13073_C/A	chr1:13073	A	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13073_C/A	chr1:13073	A	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1331;STRAND=-1
chr1_13073_C/A	chr1:13073	A	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4296;STRAND=-1
chr1_13074_T/C	chr1:13074	C	ENSG00000223972	ENST00000450305	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13074_T/C	chr1:13074	C	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13074_T/C	chr1:13074	C	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1330;STRAND=-1
chr1_13074_T/C	chr1:13074	C	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4295;STRAND=-1
chr1_13076_G/A	chr1:13076	A	ENSG00000223972	ENST00000450305	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13076_G/A	chr1:13076	A	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13076_G/A	chr1:13076	A	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1328;STRAND=-1
chr1_13076_G/A	chr1:13076	A	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4293;STRAND=-1
chr1_13080_G/A	chr1:13080	A	ENSG00000223972	ENST00000450305	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13080_G/A	chr1:13080	A	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13080_G/A	chr1:13080	A	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1324;STRAND=-1
chr1_13080_G/A	chr1:13080	A	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4289;STRAND=-1
chr1_13084_G/C	chr1:13084	C	ENSG00000223972	ENST00000450305	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13084_G/C	chr1:13084	C	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13084_G/C	chr1:13084	C	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1320;STRAND=-1
chr1_13084_G/C	chr1:13084	C	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4285;STRAND=-1
chr1_13087_AG/-	chr1:13087-13088	-	ENSG00000223972	ENST00000450305	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13087_AG/-	chr1:13087-13088	-	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13087_AG/-	chr1:13087-13088	-	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	IMPACT=MODIFIER;DISTANCE=1316;STRAND=-1
chr1_13087_AG/-	chr1:13087-13088	-	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	IMPACT=MODIFIER;DISTANCE=4281;STRAND=-1
chr1_13087_A/G	chr1:13087	G	ENSG00000223972	ENST00000450305	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13087_A/G	chr1:13087	G	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13087_A/G	chr1:13087	G	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1317;STRAND=-1
chr1_13087_A/G	chr1:13087	G	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4282;STRAND=-1
chr1_13088_G/-	chr1:13088	-	ENSG00000223972	ENST00000450305	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13088_G/-	chr1:13088	-	ENSG00000290825	ENST00000456328	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
chr1_13088_G/-	chr1:13088	-	ENSG00000227232	ENST00000488147	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=1316;STRAND=-1
chr1_13088_G/-	chr1:13088	-	ENSG00000278267	ENST00000619216	Transcript	downstream_gene_variant	-	-	IMPACT=MODIFIER;DISTANCE=4281;STRAND=-1
chr1_13088_G/C	chr1:13088	C	ENSG00000223972	ENST00000450305	Transcript	intron_variant,non_coding_transcript_variant	-	-	-	-	-	-	IMPACT=MODIFIER;STRAND=1
```

So I have two problems:
1. The VPN timeout is killing my SSH session with the Compute VM
2. The LOFTEE plugin is not being loaded when VEP runs.

### Standalone VEP

Joe sent the Storage location on 9/19 for the Data Release 2 VCF (`gs://gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/rel2_100k-var.vcf.bgz`), a 236 GB bzipped file.

I'm trying to figure out how to shard the file into one file per chromosome so I can run VEP on smaller files one at a time. This is the header of the file:
```
$ zcat rel2_100k-var.vcf.bgz | grep -v "^##" | head -n 1
#CHROM	POS	ID	REF	ALT	QUAL	FILTER	INFO	FORMAT	SHIP5297471	SHIP5508159...
```

The list of header columns goes on for many screens, and I suspected there were as many columns as
there are samples (104,923 in DR2). To test, I did `wc -w`:
```
$ zcat rel2_100k-var.vcf.bgz | grep -v "^##" | head -n 1 | wc -w
104932
```

Which amounts to 9 columns (CHROM, POS, ID, REF, ALT, QUAL, FILTER, INFO, and FORMAT), plus 104,923 samples, so I was right. Then I wrote a quick Bash script to split the master file by contig:

```
# /bin/bash

for contig in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 X Y M
do
	zgrep -v "^##" rel2_100k-var.vcf.bgz | head -n 1 | gzip > chr$contig.gz
	zgrep "^chr$contig" rel2_100k-var.vcf.bgz | gzip >> chr$contig.gz
done
```

Later, I asked Paul whether `grep` was sufficient for splitting the file by chromosome – I was worried there might be exceptions to the simple regex I was using based on some anomalous chromosome names I'd run across – and he said `bcftools` might be a better option. `bcftools`, however, requires an index file for random access to the VCF, so the first step is to index the VCF file. Here is my revised process:

1. Index the VCF file:
   ```
   bcftools index rel2_100k-var.vcf.bgz
   ```
2. Filter by region for each chromosome:
   ```
   bcftools filter --regions chr1 > rel2_100k-var-chr1.vcf
   ```

Step 1 took 8 days to complete (from 9/27 to 10/4) and produced a 2.3 MB index file with a `.csi` extension. In the meantime, I happened to notice an index file by the name of `rel2_100k-var.vcf.bgz.tbi` in the same Storage directory Joe gave me earlier (`gs://gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505`). So recreating the index file was probably a waste of time versus just using the one that was already there that I didn't notice when I first looked at the directory (which, to be fair, was before I knew I would need an index file or that it would take so long to generate one). By the time the index had been created, we had realized there was a better way to do what I was trying to do (see update below), so I nixed step 2.

### UPDATE

The 236 GB file that Joe uploaded turned out to include all 105k sample columns and blank genotype entries for each sample at each variant, which added a lot of unneeded bulk to the table. We realized this during a Slack meeting on 10/3 between me, Paul, and Joe. Apparently, the commands he used in Hail rendered the genotype entries blank but didn't drop the sample columns, so each (blank) entry was represented by a single dot. Later that day, he recreated the variants-only dataset in Hail without the sample columns or genotype entries and split it out by chromosome. He posted the results, 24 separate `.bgz` files, to `gs://gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/var_vcfs`, and they are much smaller (from 12 MB to 180 MB each, for 24 files).



### Standalone VEP

Regarding the problem I was having with my terminal sessions being disconnected by the VPN timeout: Paul suggested a terminal multiplexer like `screen` or `tmux`, which would keep the session alive even after I was disconnected from the SSH session. I tried `screen` and it worked like a charm. I'm amazed I've never used a terminal emulator before...

Regarding the other problem, the LOFTEE plugin not loading, I found a solution, but I'm not totally sure why it works. Instead of pointing VEP to the LOFTEE plugin code in the cloned repository, I just copied all the code from the repo to the root-level plugins directory and ran VEP without specifying the LOFTEE plugin location.

NOT WORKING:
```
vep@46a76261b877:~/.vep$   vep --cache \
>       --verbose \
>       --input_file  rel2_100k-var.vcf.bgz \
>       --output_file rel2_100k-var-vep.tsv \
>       --host        useastdb.ensembl.org \
>       --plugin      LoF,loftee_path=/opt/vep/.vep/loftee/ \
>       --dir_plugins /opt/vep/.vep/loftee/
```

WORKING:
```
cp -r /opt/vep/.vep/loftee/* /plugins

vep --cache \
    --verbose \
    --input_file  /opt/vep/.vep/rel2_100k-var-chr1.vcf.bgz \
    --output_file /opt/vep/.vep/rel2_100k-var-chr1-vep.tsv \
    --host        useastdb.ensembl.org \
    --plugin      LoF,loftee_path:/opt/vep/.vep/loftee/
```

I would normally hesitate to copy plugin code to a global directory and risk overwriting existing files, especially when they use common directory names like `src`, but since this is running in a transient Docker container with no other users and a very temporary lifespan, I can't see the harm.


### Standalone VEP

My plan was to create one Compute VM per chromosome and run VEP on each of them, then stitch the results into a single output file – a sort of manually orchestrated compute cluster. The first problem I ran into was that VEP requires a homo sapiens variations file, which can either be downloaded on the fly from their [FTP site](https://ftp.ensembl.org/pub/release-110/variation/indexed_vep_cache/homo_sapiens_vep_110_GRCh38.tar.gz) or provided locally in a cache directory (they recommend the latter). I needed to get this file onto all 24 computers, and given it size, 20 GB, this was a non-trivial problem.

I had already downloaded the file to `dcotter-vep-standalone`, the Compute VM I used when trying to run VEP on the 236-gigabyte master VCF file, so I had a copy of the file. My first thought was to upload this to Storage from the first VM and then download it onto the other 24 VMs. However, when I tried this, I got `403: Access Denied` errors, which persisted even after I changed the Storage scope on that VM to `ReadWrite`.

Then I tried downloading it to my laptop so that I could upload it to Storage myself, since I knew I could upload files. But the download from their FTP site was extremely slow (300 KB/s), which I thought might be related to my being on Stanford's VPN at the time, so I disconnected and tried again. Still slow, but marginally faster.

Then I tried using Globus Connect Personal to download it to my laptop, but I kept getting errors within the GCP app. Example:
```
  2023-10-04 15:09:35 Connecting to server: relay.globusonline.org:2223
  2023-10-04 15:09:37 [ssh stderr] Allocated port 36925 for remote forward to 127.0.0.1:52659
  2023-10-04 15:09:37 [ssh stderr] rpc succeeded after 1 attempts(s)
  2023-10-04 15:09:37 [ssh stderr] Relay Service Error: The endpoint was deleted on 2022-03-02 23:05:49.753800 UTC
  2023-10-04 15:09:37 [ssh stderr] 
```
I knew from past experience that Globus's settings are frequently complicated and require a sysadmin with Globus expertise to debug, and since our Globus expert is filling in for our departing SCG sysadmin at the moment and very busy, I dismissed the idea of putting a ticket in with him for the Globus issue.

However, I tried downloading the file from Globus through the web app (i.e. not using Globus Connect Personal), and the average download speed is around 1 MB/s, so a 20 GB file should be done in 20,000 s or ~5.5 hours. Tolerable, but still slow, and if I have to upload it to Storage after that and download it onto 24 VMs after that, it would still delay me by days. I kept the download going while I thought about other options.

Finally, I hit on the idea that would drastically speed up the whole operation: **Clone the disk through the Compute API**. I ran the following:
```
gcloud compute disks create dcotter-vep-chry \
  --project=gbsc-gcp-project-mvp \
  --type=pd-balanced \
  --description=Includes\ 20G\ homo\ sapiens\ cache\ file\ required\ by\ VEP \
  --size=500GB \
  --zone=us-west1-b \
  --source-disk=projects/gbsc-gcp-project-mvp/zones/us-west1-b/disks/data-disk`)
```

It took less than a minute to create the disk; and after attaching the cloned disk to a new Compute VM and mounting the device, I can see the 20G file there, so I'm going to call it a success.



### Standalone VEP

Now, I'm wondering if I can create an image that includes VEP, LOFTEE, the 20G cache file, and any other requirements for running VEP, and then use that image for each of the 24 VEP VMs? I don't want to waste precious time on technical adventurism; on the other hand, it might pay off in a reusable image and some useful new knowledge. The GCP console notes that:

> A machine image contains a VM’s properties, metadata, permissions, and data from all its attached disks. You can use a machine image to create, backup, or restore a VM.

The [docs](https://cloud.google.com/compute/docs/machine-images) say an image is a good option for instance cloning, so I don't think I'm far afield. Let's give it a try:

  ```
  gcloud beta compute machine-images create dcotter-vep-standalone-image \
      --project=gbsc-gcp-project-mvp \
      --description=The\ image\ includes:$'\n'\ \ -\ VEP\ Docker\ image$'\n'\ \ -\ LOFTEE\ Git\ repo$'\n'\ \ -\ homo\ sapiens\ cache\ file\ \(20Gb\)\ and\ extracted\ directory\ tree \
      --source-instance=dcotter-vep-standalone \
      --source-instance-zone=us-west1-b \
      --storage-location=us-west1
  ```

And now I'm creating an instance based on the image (this will turn out to work so well I end up using it on all 24 chromosome-specific VMs):

  ```
  gcloud compute instances create dcotter-vep-standalone-chr01 \
    --project=gbsc-gcp-project-mvp \
    --zone=us-west1-b \
    --machine-type=n1-highmem-32 \
    --network-interface=network=default,network-tier=PREMIUM,stack-type=IPV4_ONLY \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=963911152157-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/trace.append,https://www.googleapis.com/auth/devstorage.read_write \
    --min-cpu-platform=Automatic \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --labels=goog-ec-src=vm_add-gcloud \
    --reservation-affinity=any \
    --source-machine-image=dcotter-vep-standalone-image
  ```

After creating the VM based on the image:
- SSH in to instance:
  ```
  gcloud compute ssh dcotter-vep-standalone-chr01
  ```
- Download the appropriate chromosome VCF:
  ```
  gsutil cp gs://gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/var_vcfs/rel2_100k-var-chr01.vcf.bgz /home/dlcott2/data/
  ```
- Start a new `screen`:
  ```
  screen
  ```
- Start the Docker container and run VEP:
  ```
  # Start the container interactively
  docker run --tty --interactive --volume /home/dlcott2/data:/opt/vep/.vep ensemblorg/ensembl-vep
  
  # From the Docker container:
  # - copy the LOFTEE plug into /plugins
  cp -r /opt/vep/.vep/loftee/* /plugins
  
  # - run VEP
  vep --cache \
      --verbose \
      --input_file  /opt/vep/.vep/rel2_100k-var-chr01.vcf.bgz \
      --output_file /opt/vep/.vep/rel2_100k-var-chr01-vep.tsv \
      --host        useastdb.ensembl.org \
      --plugin      LoF,loftee_path:/opt/vep/.vep/loftee/
  ```

- Detach the screen (Ctrl+A, d) and log out of the SSH session (Ctrl+d)
- Tar & zip the results and upload the results file to Storage once VEP
  finishes. From the VM:
    ```
    cd /home/dlcott2/data/
    tar czf rel2_100k-var-chr2-vep.tar.gz rel2_100k-var-chr2-vep*
    gsutil cp rel2_100k-var-chr2-vep.tar.gz gs://gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/var_vcfs/
    ```

I ended up running all 24 Compute VMs in this manner and uploaded the results to `gs://gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/var_vcfs`.

### Aside: How much does an image cost?
```
$ gcloud compute machine-images describe dcotter-vep-standalone-image

creationTimestamp: '2023-10-05T13:46:39.427-07:00'
description: |-
  Includes:
  - VEP Docker image
  - LOFTEE Git repo
  - homo sapiens cache file (20Gb) and extracted directory tree
id: '2888490450307685954'
...
name: dcotter-vep-standalone-image
...
totalStorageBytes: '44591179008'
```

44,591,179,008 bytes = ~45 GB
Machine image (per GB / month): $0.05 ([source](https://cloud.google.com/compute/disks-image-pricing#machine_image))
45 * 5 cents / month = $2.25 / month





