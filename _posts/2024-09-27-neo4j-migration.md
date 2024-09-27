---
layout: post
title:  "#48 Neo4j migration from v3 to v4"
date:   2024-09-27 10:00:00 -0800
author: Daniel Cotter
categories: jekyll update
---

### Motivation

The goal of this migration is twofold: to upgrade our Neo4j database from
version 3 (current production) to version 4 and then to version 5 (the latest
version) and to move the DBMS from a standalone VM to a Kubernetes (K8s)
cluster. This migration will help modernize our infrastructure, prepare for
FedRAMP, improve scalability, and ensure better compatibility with new features
available in Neo4j v5.

#### Why Not Terraform or a Standalone VM?

One of our goals in preparing for FedRAMP is to automate as much of Trellis's
infrastructure as possible using modern software development tools and
methodologies such as Terraform (an Infrastructure-as-Code tool). However, this
migration is an exception:

- **No Terraform**: Since this is a one-time migration, using Terraform to
  automate everything felt unnecessary. While Terraform excels at
  infrastructure-as-code and automation for ongoing deployments, we needed a
  manual, hands-on approach here to handle complexities and adjustments
  throughout the migration.
- **No Standalone VM**: The goal was not just to upgrade the database but also
  to ensure it functions correctly in a cluster environment. Using Kubernetes
  directly for the migration allowed us to verify how Neo4j performs in this
  configuration, making the transition smoother for production deployment.

Migrating both the data and the architecture the database management system
(DBS) was deployed on made the migration more complex, but since Neo4j has
itself evolved from being intended for use on a standalone machine in v3 to a
cluster in v5, it made sense to do both at once so I could adjust the server
configuration in tandem with the data itself.

### Security Preliminaries

#### Creating a service account for dev

To maintain security and separate concerns during the migration process, I
created a dedicated service account, limiting its access to the development
project in GCP and assigning roles only as needed for specific tasks:

  ```bash
  % gcloud iam service-accounts create dev-service-account --display-name "Dev Service Account"
  Created service account [dev-service-account].

  % gcloud auth activate-service-account --key-file=/Users/dlcott2/Documents/mvp/fedramp/trellis-refactoring/mvp-fedramp-trellis-dev-babdc75eb753.json
  Activated service account credentials for: [dev-service-account@mvp-fedramp-trellis-dev.iam.gserviceaccount.com]
  #^^^uses the key created through the Cloud Console

  % gcloud projects add-iam-policy-binding mvp-fedramp-trellis-dev \
    --member="serviceAccount:dev-service-account@mvp-fedramp-trellis-dev.iam.gserviceaccount.com" \
    --role="roles/compute.admin"

  % gcloud projects add-iam-policy-binding mvp-fedramp-trellis-dev \
    --member="serviceAccount:dev-service-account@mvp-fedramp-trellis-dev.iam.gserviceaccount.com" \
    --role="roles/storage.admin"

  ...

  % gcloud projects get-iam-policy mvp-fedramp-trellis-dev --format="json(bindings)" | \
    jq -r '.bindings[] | select(.members[] | contains("serviceAccount:dev-service-account@mvp-fedramp-trellis-dev.iam.gserviceaccount.com")) | .role'

    roles/bigquery.admin
    roles/cloudbuild.builds.editor
    roles/cloudbuild.connectionAdmin
    roles/compute.admin
    roles/container.admin
    roles/iam.serviceAccountAdmin
    roles/iam.serviceAccountUser
    roles/monitoring.editor
    roles/notebooks.admin
    roles/pubsub.editor
    roles/serviceusage.serviceUsageAdmin
    roles/storage.admin
    roles/vpcaccess.admin

  % gcloud iam service-accounts list
  DISPLAY NAME         EMAIL                                                                DISABLED
  Dev Service Account  dev-service-account@mvp-fedramp-trellis-dev.iam.gserviceaccount.com  False
  ```

Then I tested creating a bucket in production under the service account, which
should fail if I set it up correctly:

  ```bash
  % gcloud storage buckets create --project=gbsc-gcp-project-mvp gs://daniels-test-bucket

  Creating gs://daniels-test-bucket/...
  ERROR: (gcloud.storage.buckets.create) HTTPError 403:
  dev-service-account@mvp-fedramp-trellis-dev.iam.gserviceaccount.com does not
  have storage.buckets.create access to the Google Cloud project. Permission
  'storage.buckets.create' denied on resource (or it may not exist). This
  command is authenticated as
  dev-service-account@mvp-fedramp-trellis-dev.iam.gserviceaccount.com which is
  the active account specified by the [core/account] property.
  ```

I created a "superuser" account similarly so that I could use my own account,
which has permissions on both projects, for operations that required them:

  ```bash
  % gcloud config configurations create trellis-dev-su
  Created [trellis-dev-su].

  % gcloud config set project mvp-fedramp-trellis-dev
  Updated property [core/project].

  % gcloud config set core/account dlcott2@stanford.edu
  Updated property [core/account].

  % gcloud config set compute/region us-west1
  Updated property [compute/region].

  % gcloud config set compute/zone us-west1-b
  Updated property [compute/zone].

  % gcloud config configurations describe trellis-dev-su
  is_active: true
  name: trellis-dev-su
  properties:
    compute:
      region: us-west1
      zone: us-west1-b
    core:
      account: dlcott2@stanford.edu
      project: mvp-fedramp-trellis-dev

#### Authorizing kubectl to Google Cloud Platform

Next, I authenticated the Kubernetes command-line utility, `kubectl`, to GCP so
that I could manually control the migration:

  ```bash
  % gcloud container clusters get-credentials --location us-west1 mvp-fedramp-trellis-dev-gke
  Fetching cluster endpoint and auth data.
  kubeconfig entry generated for mvp-fedramp-trellis-dev-gke.
  ```

`kubectl` runs under whichever `gcloud` account is active at the time, which you
can check with the following command:

  ```bash
  % gcloud config configurations list
  NAME            IS_ACTIVE  ACCOUNT                                                              PROJECT                    COMPUTE_DEFAULT_ZONE  COMPUTE_DEFAULT_REGION
  default         False
  mvp-dev         False      dlcott2@stanford.edu                                                 gbsc-gcp-project-mvp-dev   us-west1-b            us-west1
  mvp-prod        False      dlcott2@stanford.edu                                                 gbsc-gcp-project-mvp       us-west1-b            us-west1
  trellis-dev     True       dev-service-account@mvp-fedramp-trellis-dev.iam.gserviceaccount.com  mvp-fedramp-trellis-dev    us-west1-b            us-west1
  trellis-dev-su  False      dlcott2@stanford.edu                                                 mvp-fedramp-trellis-dev    us-west1-b            us-west1
  ```

### Migrating the data

#### First attempt: Backup in place

During the early stages of the migration, I investigated the option to back up
the production database in place, which would have been the simplest option. To
accomplish this, I logged into the production Neo4j instance to check the
available disk space and found the persistent disk housing the production data
was almost two-thirds full and wouldn't have room for a backup.

  ```bash
  % gcloud compute ssh trellis-neo4j-db

  $ docker exec -it b312b455f91f /bin/bash

  root@trellis-neo4j-db:/var/lib/neo4j# df -h
  Filesystem      Size  Used Avail Use% Mounted on
  overlay         5.7G  1.4G  4.4G  23% /
  tmpfs            64M     0   64M   0% /dev
  tmpfs            69G     0   69G   0% /sys/fs/cgroup
  shm              64M     0   64M   0% /dev/shm
  /dev/sdb        984G  596G  339G  64% /data
  /dev/sda1       5.7G  1.4G  4.4G  23% /logs
  ```

Since the production disk was too full, I decided to copy the database to a
second, larger disk in the development environment, allowing for a more
manageable backup process.

#### Second attempt: Cloning the production disk to a bigger disk in dev

After encountering issues with the persistent disk being too full, I decided to
clone the data disk currently in use by Neo4j in the production Trellis
environment to a disk twice as big in the development environment:

  ```bash
  % gcloud config configurations activate trellis-dev-su
  Activated [trellis-dev-su].

  % gcloud compute disks create trellis-neo4j-data-migration \
  --project=mvp-fedramp-trellis-dev \
  --size=2000
  --description="Neo4j data disk cloned from production Trellis on 9/12/2024 at 15:46 PDT" \
  --source-disk=projects/gbsc-gcp-project-mvp/zones/us-west1-b/disks/disk-trellis-neo4j-data

  Created [https://www.googleapis.com/compute/v1/projects/mvp-fedramp-trellis-dev/zones/us-west1-b/disks/disk-trellis-neo4j-migration].
  NAME                          ZONE        SIZE_GB  TYPE         STATUS
  disk-trellis-neo4j-migration  us-west1-b  2000     pd-standard  READY
  ```

This larger disk in the development environment would allow me to back up the
database without running into space issues.

It turns out the migration from Neo4j 3.5 to 4 is actually quite involved (to
say nothing of the migration from 4 to 5), or, as the [manual](https://neo4j.com/docs/upgrade-migration-guide/current/version-4/migration/) puts it, "A
migration to the next MAJOR version requires careful planning and
consideration."

From the manual:

  Follow the checklist to prepare for migrating your Neo4j deployment:
  - [ ] Complete all [prerequisites](https://neo4j.com/docs/upgrade-migration-guide/current/version-4/migration/migration-checklist/#migration-prerequisites) for the migration.
  - [ ] Reserve enough disk space for the pre-migration backup and the migration.
  - [ ] Shut down the Neo4j DBMS that you want to migrate.
  - [ ] Back up your current deployment to avoid losing data in case of a failure.
  - [ ] Download the new version of Neo4j. Make sure the migration path is supported.
  - [ ] Prepare a new neo4j.conf file to be used by the new installation.
  - [ ] Perform a test migration as per your Neo4j version and deployment type (Migrate a single instance (offline) or Migrate a Causal Cluster (offline)).
  - [ ] Monitor the logs.
  - [ ] Perform the migration.

### Migrating the architecture

#### Deploying Neo4j v3 to Google Kubernetes Engine (GKE)

[Helm](https://helm.sh) is a package manager for Kubernetes that simplifies the process of
deploying software to clusters by providing deployment files pre-configured
with the necessary settings (called "charts"); it was very helpful during the
proof-of-concept deployment of Neo4j version 5 to GKE.

Unfortunately, Helm only has charts going back to version 5, and I needed to
start from version 3.5.14, since that's what's running in production. This
meant I would have to install Neo4j on the cluster using a Dockerfile and
`kubectl`. Doing so involved coordinating several different pieces of software:
Docker, for the containers; Kubernetes for container orchestration; and `gcloud`
for Google's cloud services.

First, write the Dockerfile for the Neo4j image to be deployed:

  ```Dockerfile
  # Use an official openjdk base image
  FROM openjdk:8-jre-slim

  # Set environment variables
  ENV NEO4J_VERSION=3.5.14 \
      NEO4J_HOME=/var/lib/neo4j

  # Download and install Neo4j
  RUN apt-get update && apt-get install -y curl \
      && curl -SL "https://dist.neo4j.org/neo4j-community-${NEO4J_VERSION}-unix.tar.gz" \
      | tar -xz -C /var/lib/ \
      && mv /var/lib/neo4j-community-${NEO4J_VERSION} ${NEO4J_HOME} \
      && rm -rf /var/lib/neo4j-community-${NEO4J_VERSION}.tar.gz

  # Set file permissions
  RUN mkdir -p ${NEO4J_HOME}/data ${NEO4J_HOME}/logs && \
      chmod -R 777 ${NEO4J_HOME}

  # Expose Neo4j ports
  EXPOSE 7474 7687

  # Set the working directory
  WORKDIR ${NEO4J_HOME}

  # Start Neo4j
  CMD ["bin/neo4j", "console"]
  ```

The two exposed ports are for HTTP and Bolt, respectively, the protocols that
Neo4j uses to communicate with the web management console and the database
client.

Next, I had to create a Docker image repository in GCP's Artifact Registry to
store the Docker image in the cloud:

   ```bash
   % gcloud artifacts repositories create neo4j-old \
     --repository-format=docker \
     --location=us-west1 \
     --description="Docker repository for old Neo4j images"

  Create request issued for: [neo4j-old]
  Waiting for operation [projects/mvp-fedramp-trellis-dev/locations/us-west1/operations/55b67997-1731-449e-97b8-4d9b56d559ca] to complete...done.
  Created repository [neo4j-old].
  ```

Authorize Docker to push the image to GCP Artifact Registry:

  ```bash
  % gcloud auth configure-docker us-west1-docker.pkg.dev

  After update, the following will be written to your Docker config file located at [/Users/dlcott2/.docker/config.json]:
   {
    "credHelpers": {
      "asia.gcr.io": "gcloud",
      "eu.gcr.io": "gcloud",
      "gcr.io": "gcloud",
      "marketplace.gcr.io": "gcloud",
      "staging-k8s.gcr.io": "gcloud",
      "us.gcr.io": "gcloud",
      "us-west1-docker.pkg.dev": "gcloud"
    }
  }

  Docker configuration file updated.
  ```

Tag the image with the location of the remote repo and push it to the repo:

  ```bash
  % docker build -t us-west1-docker.pkg.dev/mvp-fedramp-trellis-dev/neo4j-old/neo4j-old-version:3.5.14 .
    [+] Building 1.2s (8/8) FINISHED                                                                                                                                                                                       docker:desktop-linux
     => [internal] load build definition from Dockerfile                                                                                                                                                                                   0.0s
     => => transferring dockerfile: 758B                                                                                                                                                                                                   0.0s
     => [internal] load metadata for docker.io/library/openjdk:8-jre-slim                                                                                                                                                                  1.2s
     => [internal] load .dockerignore                                                                                                                                                                                                      0.0s
     => => transferring context: 2B                                                                                                                                                                                                        0.0s
     => [1/4] FROM docker.io/library/openjdk:8-jre-slim@sha256:53186129237fbb8bc0a12dd36da6761f4c7a2a20233c20d4eb0d497e4045a4f5                                                                                                            0.0s
     => CACHED [2/4] RUN apt-get update && apt-get install -y curl     && curl -SL "https://dist.neo4j.org/neo4j-community-3.5.14-unix.tar.gz"     | tar -xz -C /var/lib/     && mv /var/lib/neo4j-community-3.5.14 /var/lib/neo4j     &&  0.0s
     => CACHED [3/4] RUN mkdir -p /var/lib/neo4j/data /var/lib/neo4j/logs &&     chmod -R 777 /var/lib/neo4j                                                                                                                               0.0s
     => CACHED [4/4] WORKDIR /var/lib/neo4j                                                                                                                                                                                                0.0s
     => exporting to image                                                                                                                                                                                                                 0.0s
     => => exporting layers                                                                                                                                                                                                                0.0s
     => => writing image sha256:b3a39d7fd71b6e0b3da31f154b5e46dd65823fa0f2e79fc2bfb5d4ec47a4c57a                                                                                                                                           0.0s
     => => naming to us-west1-docker.pkg.dev/mvp-fedramp-trellis-dev/neo4j-old/neo4j-old-version:3.5.14                                                                                                                                    0.0s

  % docker push us-west1-docker.pkg.dev/mvp-fedramp-trellis-dev/neo4j-old/neo4j-old-version:3.5.14
  The push refers to repository [us-west1-docker.pkg.dev/mvp-fedramp-trellis-dev/neo4j-old/neo4j-old-version]
  5f70bf18a086: Layer already exists 
  a4a3a44adfc5: Pushed 
  1e1f8b5f0236: Pushed 
  b66078cf4b41: Pushed 
  cd5a0a9f1e01: Pushed 
  eafe6e032dbd: Pushed 
  92a4e8a3140f: Pushed 
  3.5.14: digest: sha256:ee7f0c771a3ced9938c20621656bb8729882c2f3d50f40a87fea0c660ffbc208 size: 1791
  ```

Then write the YAML specifiying the Kubernetes resources to be created:

The first resources needed are Kubernetes abstractions for managing persistent
disks, the Persistent Volume and the Persistent Volume Claim:

  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: neo4j-data-migration-pv
  spec:
    capacity:
      storage: 2000Gi
    accessModes:
      - ReadWriteOnce
    storageClassName: standard-rwo
    gcePersistentDisk:
      pdName: trellis-neo4j-data-migration
      fsType: ext4
    persistentVolumeReclaimPolicy: Retain
  --
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: neo4j-data-migration-pvc
    namespace: neo4j-migration
    labels:
      app: neo4j
  spec:
    accessModes:
      - ReadWriteOnce
    storageClassName: standard-rwo
    resources:
      requests:
        storage: 2000Gi
  ```

Then, the Deployment, which specifies how to create the pod on which the
software will run in the Docker container we created earlier:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: neo4j-deployment
    labels:
      app: neo4j
    namespace: neo4j-migration
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: neo4j
    template:
      metadata:
        labels:
          app: neo4j
      spec:
        containers:
        - name: neo4j
          image: us-west1-docker.pkg.dev/mvp-fedramp-trellis-dev/neo4j-old/neo4j-old-version-bash:3.5.14
          command: ['bash', '-c', 'sleep infinity']
          ports:
          - containerPort: 7474
          - containerPort: 7687
          volumeMounts:
          - name: neo4j-config
            mountPath: /var/lib/neo4j/conf
          - name: neo4j-data-migration
            mountPath: /var/lib/neo4j/data
        initContainers:
        - name: configure-neo4j
          image: busybox
          command: ['sh', '-c', "echo 'dbms.connector.bolt.listen_address=0.0.0.0:7687' >> /var/lib/neo4j/conf/neo4j.conf && echo 'dbms.connector.http.listen_address=0.0.0.0:7474' >> /var/lib/neo4j/conf/neo4j.conf"]
          volumeMounts:
          - name: neo4j-config
            mountPath: /var/lib/neo4j/conf
        volumes:
        - name: neo4j-config
          emptyDir: {}
        - name: neo4j-data-migration
          persistentVolumeClaim:
            claimName: neo4j-data-migration-pvc
  ```

The Deployment has a few things to note:
- The Docker image is pulled from the Artifact Registry we pushed to previously
- The ports exposed are the same as those exposed in the Docker container
- A separate container called an "initializion container" is created when the
  main container is created and appends a couple of lines to Neo4j's config
  file telling it which ports to listen on (this file is accessed through the 
  empty-directory volume the init container mounts at `/var/lib/neo4j/conf`)
- The Persistent Volume Claim we created before, representing the persistent
  disk with the cloned data on it, is mounted at `/var/lib/neo4j/data`, Neo4j's
  default data directory
- The command `sleep infinity` keeps the container running indefinitely, unlike
  a command that exits immediately such as `sh`, without starting Neo4j, which
  would run indefinitely but lock the database and prevent dumping it to a
  file.
  
And finally, the Service, which exposes the DBMS to the network for clients:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: neo4j-service
    namespace: neo4j-migration
  spec:
    type: LoadBalancer  # Or ClusterIP for internal access
    selector:
      app: neo4j
    ports:
    - port: 7474
      name: http
      targetPort: 7474
      protocol: TCP
    - port: 7687
      name: bolt
      targetPort: 7687
      protocol: TCP
  ```

Then create the resources:

  ```bash
  % kubectl apply -f neo4j-pv.yaml
  persistentvolume/neo4j-pvc created
  
  % kubectl apply -f neo4j-pvc.yaml
  persistentvolumeclaim/neo4j-pvc created
  
  % kubectl apply -f neo4j-deployment.yaml
  deployment.apps/neo4j-deployment configured

  % kubectl apply -f neo4j-service.yaml
  service/neo4j-service created
  
  % kubectl logs neo4j-deployment-7579c454f9-kzkpv
  Defaulted container "neo4j" out of: neo4j, configure-neo4j (init)
  Starting Neo4j.
  2024-09-12 21:54:34.897+0000 INFO  ======== Neo4j 3.5.14 ========
  2024-09-12 21:54:34.909+0000 INFO  Starting...
  2024-09-12 21:54:37.447+0000 INFO  Bolt enabled on 0.0.0.0:7687.
  2024-09-12 21:54:38.899+0000 INFO  Started.
  2024-09-12 21:54:40.132+0000 INFO  Remote interface available at http://localhost:7474/
  ```

I was able to connect via the desktop client and see the ports with Nmap:
  
  ```bash
  % nmap -Pn -p 7474,7687 34.82.5.38

  Starting Nmap 7.95 ( https://nmap.org ) at 2024-09-12 15:06 PDT
  Nmap scan report for 38.5.82.34.bc.googleusercontent.com (34.82.5.38)
  Host is up (0.056s latency).

  PORT     STATE SERVICE
  7474/tcp open  neo4j
  7687/tcp open  bolt

  Nmap done: 1 IP address (1 host up) scanned in 0.13 seconds
  ```
  
#### Migrating the data and DBMS to Neo4j v4

Migrating to v4 required me first to back up several key items from v3:

- The `neo4j.conf` configuration file
- All the files used for encryption, such as private key, public
  certificate, and the contents of the trusted and revoked directories
  (located in `<neo4j-home>/certificates/`).
- The data store (the `graph.db` directory, in our case)

Once
I followed a similar procedure to deploy Neo4j v4.0.0 to GKE and mount the data
disk, with a few adjustments:

- I edited the YAML for the Kubernetes Deployment resource by removing the
  `emptyDir` at `/var/lib/neo4j/data` and replacing it with the mounted
  `migration-data` volume.
- I used the database password from the production MVP project successfully,
  which means the correct database was mounted.

To back up the data store, I checked first whether the PV and PVC had been
mounted correctly. Checking the free space on the deployment, I see:

  ```bash
  % kubectl exec -it neo4j-deployment-7c76b7ff97-td2xf -- df -h
  Defaulted container "neo4j" out of: neo4j, configure-neo4j (init)
  Filesystem      Size  Used Avail Use% Mounted on
  overlay          95G  5.9G   89G   7% /
  tmpfs            64M     0   64M   0% /dev
  /dev/sdc        2.0T  596G  1.3T  32% /data
  /dev/sda1        95G  5.9G   89G   7% /etc/hosts
  shm              64M     0   64M   0% /dev/shm
  tmpfs            13G   12K   13G   1% /run/secrets/kubernetes.io/serviceaccount
  tmpfs           7.9G     0  7.9G   0% /proc/acpi
  tmpfs           7.9G     0  7.9G   0% /proc/scsi
  tmpfs           7.9G     0  7.9G   0% /sys/firmware
  ```

That looks right to me – the `graph.db` database was around 600 GB when I
cloned the persistent disk. The directory listing looks right too (full of
data, that is):

  ```bash
  % kubectl -n neo4j-migration exec -it neo4j-deployment-7c76b7ff97-td2xf -- /bin/bash
  Defaulted container "neo4j" out of: neo4j, configure-neo4j (init)
  root@neo4j-deployment-7c76b7ff97-td2xf:/var/lib/neo4j# alias ls='ls -la'
  root@neo4j-deployment-7c76b7ff97-td2xf:/var/lib/neo4j# ls /data/databases/graph.db/
  total 408965296
  drwxrwxrwx 4 _apt 101         4096 Jul 14 15:10 .
  drwxrwxrwx 3 _apt 101         4096 Apr 15  2020 ..
  drwxrwxrwx 2 _apt 101         4096 Apr 15  2020 index
  -rwxrwxrwx 1 _apt 101         8192 Aug 16 16:45 neostore
  -rwxrwxrwx 1 _apt 101        16544 Aug 16 16:30 neostore.counts.db.a
  -rwxrwxrwx 1 _apt 101        16544 Aug 16 16:45 neostore.counts.db.b
  -rwxrwxrwx 1 _apt 101            9 Jul 18 22:07 neostore.id
  -rwxrwxrwx 1 _apt 101   1814085632 Aug 16 16:45 neostore.labelscanstore.db
  -rwxrwxrwx 1 _apt 101         8190 Feb 20  2023 neostore.labeltokenstore.db
  -rwxrwxrwx 1 _apt 101            9 Jul 18 22:07 neostore.labeltokenstore.db.id
  -rwxrwxrwx 1 _apt 101         8192 Feb 20  2023 neostore.labeltokenstore.db.names
  -rwxrwxrwx 1 _apt 101            9 Jul 18 22:07 neostore.labeltokenstore.db.names.id
  -rwxrwxrwx 1 _apt 101   2181611250 Aug 16 16:45 neostore.nodestore.db
  -rwxrwxrwx 1 _apt 101       382745 Jul 18 22:07 neostore.nodestore.db.id
  -rwxrwxrwx 1 _apt 101   1373880320 Jun 27  2022 neostore.nodestore.db.labels
  -rwxrwxrwx 1 _apt 101            9 Jul 18 22:07 neostore.nodestore.db.labels.id
  -rwxrwxrwx 1 _apt 101 115260577518 Aug 16 16:45 neostore.propertystore.db
  -rwxrwxrwx 1 _apt 101  19773677568 Aug 16 16:45 neostore.propertystore.db.arrays
  -rwxrwxrwx 1 _apt 101            9 Aug 13 14:00 neostore.propertystore.db.arrays.id
  -rwxrwxrwx 1 _apt 101            9 Aug 13 14:00 neostore.propertystore.db.id
  -rwxrwxrwx 1 _apt 101         8190 May 12  2023 neostore.propertystore.db.index
  -rwxrwxrwx 1 _apt 101            9 Jul 18 22:07 neostore.propertystore.db.index.id
  -rwxrwxrwx 1 _apt 101        16340 May 12  2023 neostore.propertystore.db.index.keys
  -rwxrwxrwx 1 _apt 101            9 Jul 18 22:07 neostore.propertystore.db.index.keys.id
  -rwxrwxrwx 1 _apt 101 271476342784 Aug 16 16:45 neostore.propertystore.db.strings
  -rwxrwxrwx 1 _apt 101            9 Aug 13 14:00 neostore.propertystore.db.strings.id
  -rwxrwxrwx 1 _apt 101     48445050 May 12  2023 neostore.relationshipgroupstore.db
  -rwxrwxrwx 1 _apt 101       328801 Jul 18 22:07 neostore.relationshipgroupstore.db.id
  -rwxrwxrwx 1 _apt 101   6148796640 Aug 16 16:45 neostore.relationshipstore.db
  -rwxrwxrwx 1 _apt 101    225432377 Jul 18 22:07 neostore.relationshipstore.db.id
  -rwxrwxrwx 1 _apt 101         8190 May 12  2023 neostore.relationshiptypestore.db
  -rwxrwxrwx 1 _apt 101            9 Jul 18 22:07 neostore.relationshiptypestore.db.id
  -rwxrwxrwx 1 _apt 101         8192 May 12  2023 neostore.relationshiptypestore.db.names
  -rwxrwxrwx 1 _apt 101            9 Jul 18 22:07 neostore.relationshiptypestore.db.names.id
  -rwxrwxrwx 1 _apt 101         8192 May 12  2023 neostore.schemastore.db
  -rwxrwxrwx 1 _apt 101           17 Jul 18 22:07 neostore.schemastore.db.id
  -rw-r--r-- 1  101 101    262146817 Jul 14 15:02 neostore.transaction.db.2971
  -rw-r--r-- 1  101 101    227505103 Aug 16 16:45 neostore.transaction.db.2972
  drwxrwxrwx 3 _apt 101         4096 Apr 16  2020 schema
```

At this point, I was able to backup the database using `neo4j-admin` at the
command line, which took ~2.5 hours and resulted in a file of 62 GB,
significantly smaller than the 596 GB directory on disk, meaning the backup was
compressed.

For the cryptographic elements, I copied the TLS certificate and private key
and created Kubernetes resources to represent them to the cluster:

  ```bash
  % gcloud compute ssh trellis-neo4j-db
    ########################[ Welcome ]########################
    #  You have logged in to the guest OS.                    #
    #  To access your containers use 'docker attach' command  #
    ###########################################################
                                                               
  dlcott2@trellis-neo4j-db ~ $ docker exec b312b455f91f ls /var/lib/neo4j/certificates
  neo4j.cert
  neo4j.key

  % kubectl create secret tls tls-cert-secret \
  > --cert=trellis-neo4jv4-db-files/certificates/neo4j.cert \
  > --key=trellis-neo4jv4-db-files/certificates/neo4j.key 
  secret/tls-cert-secret created
  ```

And for the configuration file, I copied it to my local machine and created a
Kubernetes ConfigMap resource to represent it to the cluster:

  ```bash
  kubectl create configmap neo4j-config --from-file=config.yaml=trellis-neo4jv4-db-files/conf/neo4j.conf
  ```

At this point, I started up the cluster and was met with several errors and
warnings, mostly as a result of configuration file changes from v3 to v4:

- Fixed out of memory error (#137) by changing heap settings in neo4j.conf:
  - before:
    - dbms.memory.pagecache.size=64G
    - dbms.memory.heap.max_size=64G
    - dbms.memory.heap.initial_size=64G
  - after:
    - dbms.memory.pagecache.size=8G
    - dbms.memory.heap.max_size=6G
    - dbms.memory.heap.initial_size=6G
- ERROR Failed to start Neo4j on 0.0.0.0:7474: HTTPS set to enabled, but no SSL policy provided
  - I changed `dbms.connector.http.enabled` to `false` since I only need Bolt
    at the moment
- WARN: Use of deprecated setting port propagation. port 7687 is migrated from dbms.connector.bolt.listen_address to dbms.connector.bolt.advertised_address.
- WARN: Use of deprecated setting port propagation. port 7474 is migrated from dbms.connector.http.listen_address to dbms.connector.http.advertised_address.
- WARN: Use of deprecated setting port propagation. port 7473 is migrated from dbms.connector.https.listen_address to dbms.connector.https.advertised_address.

Those were easy changes to make, since they only involved editing the config
file.

### Conclusion

At this point, the data, configuration, and cryptographic elements are backed
up, but not fully migrated. The next steps are to run the migration procedure
on the data and tweak the configuration so that the database can run on a
Kubernetes cluster.


