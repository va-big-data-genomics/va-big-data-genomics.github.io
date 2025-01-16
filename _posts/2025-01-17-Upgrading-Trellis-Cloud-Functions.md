---
layout: post
title:  "#51 Upgrading Trellis Cloud Functions"
date:   2025-01-16 00:00:00 -0800
author: Daniel Cotter
categories: jekyll update
---

# What are Cloud Functions?

Cloud Functions are part of Google Cloud Platform's suite of services. The
generic term for these is 'serverless functions,' and all major cloud providers
offer similar services: on Amazon Web Services (AWS), they are called 'Lambda
Functions'; on Azure, 'Azure Functions'; and so on.

Serverless functions differ from traditional functions in several important
ways. Traditionally, functions existed within a larger body of code, such as a
script, library, or class. They referenced state (variables) within the context
of that larger body of code. These functions were called by other functions or
directly within the script.

Serverless functions, by contrast, exist independently, outside any larger body
of code. They are not part of a class or library and do not reference state
within such a context. Nor are they called by another function, class, or
script. Instead, they operate as standalone functions in practice—for example,
responding to external triggers such as HTTP requests.

Serverless functions are typically triggered by events. When defining a function
on a cloud platform, it is associated with a specific event. When this event
occurs, the function is executed, with details about the triggering event passed
as input. Examples of triggering events include HTTP requests, files written to
Cloud Storage, or messages added to a Publish/Subscribe queue or topic. While
serverless functions are not stateless, any state they use is derived from the
triggering event.

The term 'serverless' is somewhat misleading; it means the functions are not
tied to a specific server. Instead, a temporary, ephemeral server is created to
host the function whenever it is executed.

The difference between traditional and serverless functions parallels the
difference between monolithic and microservices architectures. In a monolithic
architecture, all code resides in one or several executables hosted on a single
server or a small cluster. Scaling is achieved vertically, by upgrading
machines, or horizontally, by adding servers, but neither process is easily
automated. In contrast, a microservices architecture allows automatic scaling of
individual services or functions in response to demand, leveraging the cloud's
extensive commodity hardware.

# What do Cloud Functions do in Trellis?

The Cloud Functions are the core of Trellis's business logic. As the Trellis
[paper](https://doi.org/10.1038/s41598-021-02569-5) puts it, "The application follows a microservice architecture where each
service is implemented as a serverless function and the state of the system is
tracked using metadata stored in a Neo4j property graph database."

They coordinate interactions between the major subsystems of Trellis: the file
system (Cloud Storage), the database (Neo4j), and the bioinformatics pipelines,
orchestrated by `dsub` and in the case of the GATK $5 pipeline, Cromwell.

# What needs to be updated?

## The Python runtime

When you look at the Cloud Functions in the Cloud Console, you will notice a
yellow warning flag by the Runtime column for those still using Python 3.7.
Hovering over the flag, a notice appears:

> Python 3.7 will be deprecated on Jan 30, 2024. Please update this function
> using the latest runtime version available.

### What happens if we don't upgrade?

Given the looming deprecation when I began working on upgrading Trellis in early
January 2024, I asked what exactly would happen when the runtime was deprecated:
Would the function stop working? Or would the runtime just not continue to get
security patches? According to our technical account manager at Google:

> After the depreciation date, any functions you have running Python 3.7 will
> continue to exist and function as intended. However, deprecation means that
> the Python 3.7 runtime will no longer be actively maintained, and so it will
> not receive any important security or maintenance updates after the
> depreciation date. While the effects of this will probably not be immediate,
> it is highly advised you begin upgrading your functions to a supported runtime
> as soon as you are able to.
    
It is a moot point in any case what the exact effects of the deprecation would
be, since FedRAMP authorization requires Trellis to use a non-deprecated runtime
in order to ensure it receives important security updates.

### Which runtime should we upgrade to?

The latest version of Python was v3.13, but I had reservations about being on
the bleeding edge, so I checked which runtimes Google Cloud Platform would
support:

  ```bash
  % gcloud functions runtimes list --region us-west1
  NAME       STAGE       ENVIRONMENTS
  ...
  python37   DEPRECATED  1st gen
  python38   DEPRECATED  1st gen, 2nd gen
  python39   GA          1st gen, 2nd gen
  python310  GA          1st gen, 2nd gen
  python311  GA          1st gen, 2nd gen
  python312  GA          1st gen, 2nd gen
  ...
  ```

So the latest runtime wasn't even supported yet by GCP, which definitely ruled
it out. The second-to-latest runtime was supported; however, I chose to use the
one before that, v3.11, based on [ChatGPT's advice](https://chatgpt.com/c/6753420e-01d0-800f-90c0-4dc9b16416ba) when I ran into an error
importing the Python libraries `six` and `pkg_resources` (more on that later):

> > If you’re using Python 3.13, note that many libraries might not yet fully
> > support this version. Downgrade to Python 3.10 or 3.11 for better
> > compatibility.
>   
> Why use Python 3.11 instead of 3.12?
> 
> > The recommendation to use Python 3.10 or 3.11 instead of 3.12 (or 3.13 in your
> > case) is due to compatibility concerns with third-party libraries. Here's why:
> >
> > **Why Not Python 3.12?**
> > 1. **New Release Cycle**:
> >    - Python 3.12 was officially released recently (October 2023), and many
> >      libraries might not yet be fully tested or updated to support it.
> >    - Some dependencies, particularly older or less actively maintained
> >      libraries, might still specify compatibility with versions up to 3.11.
> >
> > 2. **Library Ecosystem Lag**:
> >    - Popular libraries like `google-cloud-*` and others often take a few
> >      months to update their packages to ensure compatibility with new Python
> >      versions. The error involving `pkg_resources` and `six` suggests the
> >      library ecosystem for Google Cloud libraries isn't fully ready for Python
> >      3.12+.
> >
> > 3. **Stability**:
> >    - If you're working on a project where reliability is critical, sticking to
> >      an older, well-supported version of Python (such as 3.10 or 3.11) ensures
> >      stability.
> >    - These versions are in their "long tail" phase, meaning they receive only
> >      critical bug and security fixes, making them highly stable.
> >
> > **Why Use Python 3.10 or 3.11?**
> > 1. **Wide Compatibility**:
> >    - Almost all actively maintained libraries support Python 3.10 and 3.11,
> >      making them safe choices.
> >    - Python 3.10 introduced structural pattern matching and other useful
> >      features, while 3.11 brought performance improvements, so you’re not
> >      missing out on modern capabilities.
> >
> > 2. **Google Cloud Support**:
> >    - Google Cloud libraries officially support Python 3.10 and 3.11. While
> >      they may work on 3.12, it's less certain and could lead to subtle issues.
> >
> > 3. **Tested Environment**:
> >    - These versions have been rigorously tested in production environments,
> >      ensuring your cloud functions work seamlessly.

That seemed like sensible advice, so I stuck with 3.11.

## Imported libraries

Trellis's Cloud Functions utilize a variety of Python libraries to perform their
tasks. The specific version of each library is defined in a file named
`requirements.txt`, which Python's package manager, `pip`, uses as a list of
dependencies to install.

Why is it necessary to upgrade libraries in Trellis's Cloud Functions if the
functions' code hasn't changed? If the libraries worked when the functions were
last updated (circa 2021), why might they no longer work today? After all, we
are not utilizing any new features introduced in later versions of these
libraries.

The answer lies in the dual purpose of library upgrades: they are both forward-
and backward-looking. Upgrades introduce new features while also refining or
fixing existing code. For Trellis, particularly in the context of FedRAMP
compliance, the latter is our primary concern. Library upgrades are often
published in response to security vulnerabilities and bugs, to ensure
compatibility with other libraries, and to address other issues. In some cases,
a library may no longer be maintained and must be replaced by a different one.
For Trellis, upgrading to newer library versions is primarily about securing the
system's code and maintaining its integrity.

# The upgrade

## Upgrading the Cloud Functions' runtimes

Trellis uses a Google Cloud Platform service called Cloud Build to automatically
build and deploy Cloud Functions when the code for the function is updated in
the GitHub repository.

In Trellis's Cloud Functions repository, each function is in an eponymouse
subdirectory under `/functions/`, along with a file named `cloudbuild.yaml`
specifying the steps required to build the function, including the Python
runtime to use.

As an example, here is the `cloudbuild.yaml` file for `create-blob-node`:

  ```yaml
  steps:
  - name: 'ubuntu'
    args: ['cp', '-r', 'config/${_DATA_GROUP}', 'functions/create-blob-node/']
  - name: 'gcr.io/cloud-builders/gcloud'
    args: [
           'beta',
           'functions',
           'deploy',
           'trellis-create-blob-node-${_BUCKET_SHORT_NAME}-${_OPERATION_SHORT_NAME}',
           '--project=${PROJECT_ID}',
           '--source=functions/create-blob-node',
           '--memory=128Mi',
           '--max-instances=100',
           '--entry-point=create_node_query',
           '--runtime=python311',
           '--trigger-resource=${_TRIGGER_RESOURCE}',
           '--trigger-event=google.storage.object.${_TRIGGER_OPERATION}',
           '--trigger-location=us-west1',
           '--update-env-vars=CREDENTIALS_BUCKET=${_CREDENTIALS_BUCKET}',
           '--update-env-vars=CREDENTIALS_BLOB=${_CREDENTIALS_BLOB}',
           '--update-env-vars=ENVIRONMENT=${_ENVIRONMENT}',
           '--update-env-vars=TRIGGER_OPERATION=${_TRIGGER_OPERATION}',
           '--update-env-vars=GIT_COMMIT_HASH=${SHORT_SHA}',
           '--update-env-vars=GIT_VERSION_TAG=${TAG_NAME}',
           '--update-labels=trigger-operation=${_OPERATION_SHORT_NAME}',
           '--update-labels=trigger-resource=${_TRIGGER_RESOURCE}',
           # Fix for logging issue: https://issuetracker.google.com/issues/155215191#comment112
           '--update-env-vars=USE_WORKER_V2=true',
           '--update-labels=user=trellis',
    ]
  options: 
    logging: CLOUD_LOGGING_ONLY
  ```

This means that changing the Python runtime is as simple as replacing
`--runtime=python37` in `cloudbuild.yaml` with `--runtime=python311`, then committing
and pushing the code to the repo of origin.

### Troubleshooting

I started troubleshooting one function at a time, even though I suspected many
of the functions would suffer from the same problems, since they repeated a lot
of the same code. I figured once I had fixed the problem in one function, I
could apply the fix in bulk to all the functions that suffered from the same
problem.

#### Error #1: KeyError: 'FUNCTION_NAME'

This error occurred in the following section of code in `main.py` of the
function `create-blob-node`. Identical code was found in many of the functions:

  ```python
  if ENVIRONMENT == 'google-cloud':
      FUNCTION_NAME = os.environ['FUNCTION_NAME']
  ```

The code above is trying to retrieve a value from a key-value dictionary indexed
by the key. `KeyError` means the key provided was not found in the dictionary.
Normally, this might indicate a programming error - either the key was
misspelled, the key-value pair hadn't been set yet or had been deleted from the
dictionary, or something along those lines.

However, several things weighed against that conclusion. One, the function had
been building and running successfully before the runtime change. Two,
`os.environ` is part of the `os` (operating system) library in Python, and the
environment in this case was the GCP runtime environment, meaning this was not a
key-value pair defined by the user.

There are two ways environment variables can be defined in the GCP runtime
environment: by the user and by GCP. In the cloud function definition, it is
possible to set environment variables (in fact, there is an example of this
being done from the command line in the `cloudbuild.yaml` code above). But
`FUNCTION_NAME` was not and had not previously been set by the user, so it must
have been set at one time by GCP itself. Also, common sense would seem to
indicate that setting the function name is something the platform itself could
easily and automatically do, rather than making the user do extra work by
creating an environment variable in each function's definition just to specify
the function's name.

In that line of thinking, I started looking into how these values were set by
GCP, and this was harder than I expected. I expected to find a section of the
user's manual saying which environment variables were set by GCP, and I did.
However, it took a bit more digging to find an [archived version](https://web.archive.org/web/20200811142831/https://cloud.google.com/functions/docs/env-var#nodejs_6_nodejs_8_python_37_and_go_111) of the Cloud
Function documentation saying which environment variables *used* to be set by
GCP but were no longer, and what they had been replaced by. It turns out the
environment variable FUNCTION_NAME was provided by Google in Python runtime 3.7,
but not in subsequent runtimes. The equivalent environment variable in later
runtimes is `FUNCTION_TARGET`, according to [current documentation](https://cloud.google.com/functions/docs/configuring/env-var#runtime_environment_variables_set_automatically).

Once I replaced `FUNCTION_NAME` with `FUNCTION_TARGET`, the error stopped. This
same error recurred with the key `GCP_PROJECT`, which had been replaced by
`GOOGLE_CLOUD_PROJECT`, but this time the solution came much faster. Since this
problem affected many of the functions, I scripted a bulk replacement of the
legacy keys:

  ```bash
  # From the repository root:
  find -mindepth 2 -maxdepth 2 -type f -name 'main.py' |\
  xargs sed -i -e 's/FUNCTION_NAME/FUNCTION_TARGET/g' -e 's/GCP_PROJECT/GOOGLE_CLOUD_PROJECT/g'
  ```

#### Error #2: ModuleNotFoundError: No module named 'yaml'

The offending line of code was at the very beginning of the function:

  ```python
  import yaml
  ```
  
The library was used a bit farther down in loading the `trellis.yaml` file
containing the database credentials:
  
  ```python
  vars_blob = storage.Client() \
            .get_bucket(os.environ['CREDENTIALS_BUCKET']) \
            .get_blob(os.environ['CREDENTIALS_BLOB']) \
            .download_as_string()
  parsed_vars = yaml.load(vars_blob, Loader=yaml.Loader)
  ```

I first checked the `requirements.txt` file to see what YAML library was specified
for Cloud Build to install, but there was no YAML library specified. However,
the code had worked previously, and this led me to believe that, similarly to
the `KeyError` above, this error was caused by something implicitly provided by
GCP in Python 3.7 that was no longer being provided.

Then the question became *which* YAML library the code had been using. Since the
library wasn't named in the code, all I had to go on was the members of the
library actually called in the code. I had a hunch it was [PyYAML](https://github.com/yaml/pyyaml), since this is
one of the most popular YAML libraries, so I opened up a Python shell and
checked for a few of the fields and method's I had seen in the cloud functions'
code:

  ```bash
  % python3.11 -m pip install pyyaml
  Requirement already satisfied: pyyaml in /usr/local/lib/python3.11/site-packages (6.0.2)
  
  % python3.11
  Python 3.11.11 (main, Dec  3 2024, 17:20:40) [Clang 16.0.0 (clang-1600.0.26.4)] on darwin
  Type "help", "copyright", "credits" or "license" for more information.
  >>> import yaml
  >>> yaml.Loader
  <class 'yaml.loader.Loader'>
  >>> yaml.load
  <function load at 0x105164220>
  ```
  
That was enough to satisfy my doubts, so I added `pyyaml` to `requirements.txt`
as an explicit requirement.

#### Error #3: Invalid value specified for container memory

This error was caused by the parameter `'--memory=128MB'` in the original
`cloudbuild.yaml`:

  ```plaintext
  ERROR: (gcloud.beta.functions.deploy) ResponseError: status=[400], code=[Ok],
  message=[Could not create Cloud Run service
  trellis-create-blob-node-from-personalis-final.
  spec.template.spec.containers[0].resources.limits.memory: Invalid value
  specified for container memory. For 0.083 CPU, memory must be between 128Mi
  and 512Mi inclusive.
  ```
 
The difference between 128 MB and 128 Mi may not be obvious at first, even to
people in technology. Megabytes (MB) and mibibytes (Mi) are similar quantities,
but they are defined slightly differently: 

  1MB = 10^6 bytes = 1,000,000 bytes
  1MiB = 2^20 bytes = 1,048,576 bytes 
 
Megabytes are by far the more commonly used unit and have the nice property of
being easily remembered round numbers, but mibibytes are powers of 2 and
therefore more accurate representations of storage and memory sizes. In the case
of our cloud functions, 128MB < 128Mi < 512Mi, therefore not just in the
wrong units but also outside the acceptable range. I changed `128MB` to `128Mi`,
and the error cleared.

## Upgrading the required libraries

Compiling a list of specific library versions that are mutually compatible is a
non-trivial task. Fortunately, there are specialized tools designed to do it
quickly and accurately. The challenge, broadly speaking, lies in the fact that
the libraries required by the code at hand often depend not only on other
libraries but on specific versions of those libraries. Sometimes, these version
requirements are mutually incompatible or only compatible within certain version
ranges.

For example, library A might require version 2.0 or higher of library B, while
library C requires version 1.9 or lower of library B. In this case, the
requirements of library A and library C are mutually incompatible. More
commonly—and less problematically—library A may require library B version 2.0 or
higher, while library C requires version 2.2 or higher. Here, the resolved
requirement would be the range of versions acceptable to both, which is version
2.2 or higher.

You can imagine how tedious and error-prone it would be to work this out
manually, especially since each library often depends on lower-level libraries,
which in turn depend on even lower-level libraries, cascading all the way down
to the Python standard libraries included with the Python runtime. That’s where
a tool like `pip-compile` comes in. It resolves these dependencies for you and
provides a mutually compatible list of libraries with specified (or "pinned")
versions.

This is, incidentally, why developing in a virtual environment is a good idea. A
virtual environment allows you to fulfill the requirements of a specific project
in isolation, preventing your local system from becoming cluttered with
different library versions that are likely to clash and cause broken code and
confusion—commonly referred to as "dependency hell."

Here's how to create a virtual environment for testing library installations:

  ```bash
  % python3.11 -m venv venv
  % source venv/bin/activate
  % pip install -r requirements.txt
  ...
  % deactivate
  ```

This creates a new virtual environment using the Python 3.11 runtime in a
directory named `venv` within the project folder. The virtual environment includes
only the libraries specified in `requirements.txt` along with their dependencies.
Virtual environments are isolated, per-project environments, separate from those
of other projects and your local computer's Python environment. Activating the
virtual environment temporarily "enters" it, while deactivating it "exits" it.

I started out with `create-blob-node`, as before. Its `requirements.txt` was
originally as follows:
 
  ```python
  pytz==2018.7
  iso8601==0.1.12
  google-cloud-storage==1.15.0
  google-cloud-pubsub==0.40.0
  ```

I removed the version numbers, which causes `pip` to default to the latest
available version, and then ran `pip-compile`:

  ```bash
  (create-blob-node) % pip-compile --output-file requirements.harmonized.txt requirements.txt 
  #
  # This file is autogenerated by pip-compile with Python 3.11
  # by the following command:
  #
  #    pip-compile --output-file=requirements.harmonized.txt requirements.txt
  #
  cachetools==5.5.0
      # via google-auth
  certifi==2024.8.30
      # via requests
  charset-normalizer==3.4.0
      # via requests
  deprecated==1.2.15
      # via
      #   opentelemetry-api
      #   opentelemetry-semantic-conventions
  google-api-core[grpc]==2.23.0
      # via
      #   google-cloud-core
      #   google-cloud-pubsub
      #   google-cloud-storage
  google-auth==2.36.0
      # via
      #   google-api-core
      #   google-cloud-core
      #   google-cloud-pubsub
      #   google-cloud-storage
  google-cloud-core==2.4.1
      # via google-cloud-storage
  google-cloud-pubsub==2.27.1
      # via -r requirements.txt
  google-cloud-storage==2.19.0
      # via -r requirements.txt
  google-crc32c==1.6.0
      # via
      #   google-cloud-storage
      #   google-resumable-media
  google-resumable-media==2.7.2
      # via google-cloud-storage
  googleapis-common-protos[grpc]==1.66.0
      # via
      #   google-api-core
      #   grpc-google-iam-v1
      #   grpcio-status
  grpc-google-iam-v1==0.13.1
      # via google-cloud-pubsub
  grpcio==1.68.1
      # via
      #   google-api-core
      #   google-cloud-pubsub
      #   googleapis-common-protos
      #   grpc-google-iam-v1
      #   grpcio-status
  grpcio-status==1.68.1
      # via
      #   google-api-core
      #   google-cloud-pubsub
  idna==3.10
      # via requests
  importlib-metadata==8.5.0
      # via opentelemetry-api
  iso8601==2.1.0
      # via -r requirements.txt
  opentelemetry-api==1.28.2
      # via
      #   google-cloud-pubsub
      #   opentelemetry-sdk
      #   opentelemetry-semantic-conventions
  opentelemetry-sdk==1.28.2
      # via google-cloud-pubsub
  opentelemetry-semantic-conventions==0.49b2
      # via opentelemetry-sdk
  proto-plus==1.25.0
      # via
      #   google-api-core
      #   google-cloud-pubsub
  protobuf==5.29.1
      # via
      #   google-api-core
      #   google-cloud-pubsub
      #   googleapis-common-protos
      #   grpc-google-iam-v1
      #   grpcio-status
      #   proto-plus
  pyasn1==0.6.1
      # via
      #   pyasn1-modules
      #   rsa
  pyasn1-modules==0.4.1
      # via google-auth
  pytz==2024.2
      # via -r requirements.txt
  pyyaml==6.0.2
      # via -r requirements.txt
  requests==2.32.3
      # via
      #   google-api-core
      #   google-cloud-storage
  rsa==4.9
      # via google-auth
  typing-extensions==4.12.2
      # via opentelemetry-sdk
  urllib3==2.2.3
      # via requests
  wrapt==1.17.0
      # via deprecated
  zipp==3.21.0
      # via importlib-metadata
  ```

As you can see, a handful of top-level requirements cascades down to quite a
number of lower-level requirements (the ones with comments like
`# via -r requirements.txt` are the top-level requirements; the others are
libraries they themselves require). Installing the required packages and their
dependencies looked like this:

  ```bash
  (create-blob-node) % pip install -r requirements.harmonized.txt
  ...
  Installing collected packages: pytz, zipp, wrapt, urllib3, typing-extensions, pyyaml, pyasn1, protobuf, iso8601, idna, grpcio, google-crc32c, charset-normalizer, certifi, cachetools, rsa, requests, pyasn1-modules, proto-plus, importlib-metadata, googleapis-common-protos, google-resumable-media, deprecated, opentelemetry-api, grpcio-status, google-auth, opentelemetry-semantic-conventions, grpc-google-iam-v1, google-api-core, opentelemetry-sdk, google-cloud-core, google-cloud-storage, google-cloud-pubsub
  Successfully installed cachetools-5.5.0 certifi-2024.8.30 charset-normalizer-3.4.0 deprecated-1.2.15 google-api-core-2.23.0 google-auth-2.36.0 google-cloud-core-2.4.1 google-cloud-pubsub-2.27.1 google-cloud-storage-2.19.0 google-crc32c-1.6.0 google-resumable-media-2.7.2 googleapis-common-protos-1.66.0 grpc-google-iam-v1-0.13.1 grpcio-1.68.1 grpcio-status-1.68.1 idna-3.10 importlib-metadata-8.5.0 iso8601-2.1.0 opentelemetry-api-1.28.2 opentelemetry-sdk-1.28.2 opentelemetry-semantic-conventions-0.49b2 proto-plus-1.25.0 protobuf-5.29.1 pyasn1-0.6.1 pyasn1-modules-0.4.1 pytz-2024.2 pyyaml-6.0.2 requests-2.32.3 rsa-4.9 typing-extensions-4.12.2 urllib3-2.2.3 wrapt-1.17.0 zipp-3.21.0
  ```
  
## Applying the upgrades in bulk

Now that I've figured out how to make the changes common to all the cloud
functions using `create-blob-node`, I can make those same changes to all the
cloud funtions at once, in a mass fix:

- In `cloudbuild.yaml`, replace Python 3.7 with 3.11, get rid of `--no-gen2`,
  and change `128MB` to `128Mi`:
  ```bash
  find -mindepth 2 -maxdepth 2 -type f -name 'cloudbuild.yaml' |\
  xargs sed -i '.bk' -e 's/python37/python311/' -e '/--no-gen2/d' -e 's/128MB/128Mi/'
  ```
- In `requirements.txt`, replace pinned versions with unpinned and append `pyyaml`:
  ```bash
  find -mindepth 2 -maxdepth 2 -type f -name 'requirements.txt' |\
  xargs sed -i '.bk' -E -e 's/[<>=]+.*//'
  
  find -type f -name requirements.txt -print0 |\
  xargs -0 -I {} sh -c 'echo "pyyaml" >> "{}"'   
  ```
- Harmonize the libraries version requirements using `pip-compile`:
  ```bash
  % for dir in *(/);
    do
      rm $dir/*bk
      mv $dir/requirements.txt $dir/requirements.txt.bk
      pip-compile $dir/requirements.txt.bk --output-file $dir/requirements.txt
    done
  ```

After committing them to GitHub and letting Cloud Build build and deploy them,
most of them succeeded, and at this point there are only 5 functions out of 27
not upgraded.

