---
layout: post
title:  "#8 Personalis Phase 3 WGS Data"
date:   2023-02-10 10:00:00 -0800
author: Paul Billing-Ross
categories: jekyll update
mathjax: false
---

# Personalis Phase 3 Whole Genome Sequencing Data
## Phase 3 Changes
Beginning in December of 2022, Personalis began delivering Fastqs for sample sequenced as part of their new phase 3 agreement with the Million Veteran Program. Changes from phase 2 to phase 3 have been summarized by Personalis in this [document](https://github.com/va-big-data-genomics/mvp-wgs-snp-indel-release/blob/main/Sequencing/ddt_va_mvp_phase3_change_summary_v2.pdf). The most relevant changes are the elimination of microarray data and an overhaul of the FASTQ naming conventions.

Aside from not having microarray data, we also lose the (2) sample metadata that were generated from them: predicted ethnicity and blood type. Additionally, the PredictedGender and Contamination are now estimated using only sequencing data. Not a huge deal, but we had used the ethnicity property to generate on-the-fly ethnic breakdowns of different sample groupings. In the future, we should look for ways to integrate our in-house ethnicity predictions back into the Trellis database.

The change in naming conventions has important functional consequences, because Trellis extracts essential metadata from object names and uses them to register new data objects in the database. Personalis began delivering phase 3 data before notifying us of any changes, so the initial batch of data [was not recognized](https://github.com/StanfordBioinformatics/trellis-mvp-functions/issues/41) and added to the database.

## Deploying a Hotfix for Phase 3
After recognizing the issue and reaching out to Personalis for clarification of the new naming structure, I started working on a [hotfix](https://nvie.com/posts/a-successful-git-branching-model/#hotfix-branches) of Trellis to deploy to our production environment. There were (2) main code elements that needed to updated, both in the node creation [configuration file](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/master/config/phase3/from-personalis/create-node-config.py) for data delivered from Personalis. The first was the regular expression used to recognize Fastqs and the second was the logic for parsing metadata from Fastq object names.

### Using groupdict() to get name elements
Looking at my old code, I saw that I had been treating recognizing the object and extracting metadata fields as two separate tasks, when a more elegant solution would be to do both tasks with regular expressions.

**Old method:**

Recognize a Fastq using a regular expression:

```
"Fastq": ["va_mvp_phase2/.*/.*/FASTQ/.*\\.fastq.gz$"]
```

Extract read group information using a separate method:

```
def read_group_name_1(db_dict, groupdict):
    """Get 2nd element of db_dict['name'] property & return as readGroup.
    """
    index = db_dict['name'].split('_')[1]
    return {'readGroup': int(index)}  
```

Another downside of the old approach is that the method for extracting metadata was static and naive to the naming structure: it would just pull out a fixed element of the name and assume that was the read group. So, if the names changed (which they did), I would have to change the regular expression and the helper methods. Bad all around.

**New method:**

Recognize a Fastq using a regular expression and get relevant metadata using the [Match.groupdict()](https://docs.python.org/3/library/re.html#re.Match.groupdict) function:

```
"Fastq": [
    r"^va_mvp_phase(?P<delivery_phase>\d)/\w+/\w+/FASTQ/(?P<shipping_id>[a-zA-Z0-9]+)_(?P<read_group>\d)_R(?P<mate_pair>\d)\.fastq\.gz$",
    r"^va_mvp_phase(?P<delivery_phase>\d)/\w+/\w+/(?P<flowcell_id>[a-zA-Z0-9]+)_(?P<shipping_id>[a-zA-Z0-9]+)_(?P<index_1>[ACGTU]+)-(?P<index_2>[ACGTU]+)_L(?P<flowcell_lane>[0-9]+)_R(?P<mate_pair>\d)_(?P<unknown>\d+)\.fastq\.gz$"
]
```

A little clunky to look at it, but now I've created separate match patterns for phase 2 and phase 3, and I've used the `(?P<subgroup_name>\w+)` elements to name different subgroups within the pattern. The groupdict() function will generate a dictionary with each subgroup name and the value.

So now, instead of different helper methods to extract values like read group and mate pair, I have one helper with logic to determine which delivery phase the Fastq comes from and get the appropriate metadata from the dictionary.

```
def get_fastq_metadata(db_dict, groupdict):
    if groupdict['delivery_phase'] == "2":
        return {
            'shippingId': groupdict['shipping_id'],
            'readGroup': groupdict['read_group'],
            'matePair': groupdict['mate_pair']
        }
    elif groupdict['delivery_phase'] == "3":
        return {
            'flowcellId': groupdict['flowcell_id'],
            'shippingId': groupdict['shipping_id'],
            'index1': groupdict['index_1'],
            'index2': groupdict['index_2'],
            'flowcellLane': int(groupdict['flowcell_lane']),
            'matePair': groupdict['mate_pair'],
            'readGroup': int(groupdict['flowcell_lane'])
            #'unknown': groupdict['unknown']
        }
    else:
        raise ValueError(f"Delivery phase {groupdict['delivery_phase']} is not valid. Supported phases: [2,3].")
```

### Writing Tests with ChatGPT

Before deploying my new code I wanted to test my new regular expression code locally, but writing tests can be kind of tedious. It's the kind of thing I'd rather to hand off to a computer. So I tried getting ChatGPT to write the test code for me. My code assigns a label to newly created objects based on which regular expressions they match. I wanted the bot to write me code to test which labels were being applied to differen test inputs.

**Prompt:**

> "There is a dictionary named 'match_patterns' where each key is a label and each value is a list of regular expression patterns. Create a dictionary called "pattern_matches" and populate it with key-value pairs. The keys should be a copy of the keys in pattern_tests and each value should be an empty list. For each value in a dictionary named 'pattern_tests', check to see whether the value is a full match for any of the regular expressions in match_patterns. If any of the patterns match, update the pattern_matches dictionary entry for the corresponding pattern tests key. Update the value of that entry to include they key of the match_pattern regular expression that matched the pattern_tests value."

**ChatGPT output:**

```
import re

match_patterns = {"label1": [r"pattern1", r"pattern2"], "label2": [r"pattern3"]}
pattern_tests = {"key1": "test1", "key2": "test2", "key3": "test3"}

pattern_matches = {key:	 [] for key in pattern_tests.keys()}

for test_key, test_value in pattern_tests.items():
    for label, patterns in match_patterns.items():
        for pattern in patterns:
            if re.fullmatch(pattern, test_value):
                pattern_matches[test_key].append(label)

for test_input, labels in pattern_matches.items():
    print(f"{test_input}: {labels}")
```

I took the code it spit out and, provided my expected inputs and outputs in the context of a [test case](https://github.com/StanfordBioinformatics/trellis-mvp-functions/blob/6bbaf8bbda7d1cec40621d69471db737c7ebcd95/config/phase3/from-personalis/test_create_node_config.py), gave it a whirl, and got the result I expected. It still took me some time and brain power to articulate the code I wanted, but I think it was still easier than trying to write the code itself. Overall, worth the effort.

## Where are we now?

**Good news:** Hotfix v1.2.9 has been deployed to production and seems to be picking up the new phase 3 data properly. Hooray.

**Bad news:** For some reason, after converting Fastqs to Ubams, the variant calling workflow seems to stop. No errors in the logs, it just stops. I've only observed (1) sample, and I added it to the database in a way that is a little different than how data would be organically be added as Personalis uploads it, so maybe there is some artifact I didn't account for that is causing an issue with this one case, or maybe there is a more systemic one. Once Personalis uploads a few more genomes, I should have a better idea.