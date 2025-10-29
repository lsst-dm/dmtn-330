# Publication flow for Butler prompt data products

```{abstract}
Describes the system supporting the end-user prompt data products Butler repository and the process of making data available in it.
```

## System Overview
The following diagram shows an overview of the dataflow for Prompt Processing
outputs. Items shown with a dotted border are not currently deployed.

```{figure} system_diagram.png
:figclass: technote-wide-content

System diagram
```

Prompt Processing outputs originate inside Prompt Processing worker pods in the
embargo rack.  The workers transfer the outputs to a central Ceph S3 file
store.  The workers then send a Kafka message to notify the [Prompt Processing
Butler Writer](https://dmtn-310.lsst.io/) service that the datasets are
available for ingestion.  The Prompt Processing Butler Writer batches up the
messages from the workers, then inserts records for the datasets into the
Butler database.

This tech note proposes modifying the Prompt Processing Butler Writer to log
ingested datasets to another Kafka topic. A new service, the Prompt Publication
Service, would use that Kafka topic as the source of truth to generate the list
of datasets available for unembargo and publication.

The Prompt Publication Service will be responsible for transferring datasets
from the `embargo` repository to two USDF Butler repositories: `/repo/main` and
`prompt_prep`.  It will publish the datasets to repositories in Google Cloud
for delivery to end users, and expire them when they are no longer needed.

Butler server at Google Cloud will serve a Butler repository to end users.
The Butler database for this repository will be hosted in AlloyDB in the
cloud.  The artifact files will be hosted at USDF in Weka, and served to
end users via the Weka S3 gateway.  The Butler server signs URLs to grant users
access to files in S3.


### Files that will be published

An initial list of Butler dataset types that will be published is currently
being tracked [in Confluence](https://rubinobs.atlassian.net/wiki/spaces/DM/pages/1000767506/Draft+prompt+product+types).

Most of the items on that list are outputs of Prompt Processing, and this tech
note is primarily concerned with those.  There are two major types of dataset
that are not Prompt Processing outputs:

1. Raws
2. Calibrations and ancillaries

Raws will eventually be integrated into the Prompt Publication Service, but
they have a slightly different lifecycle (originating outside of Prompt
Processing, and being retained forever.)  They are not discussed in detail here.

Calibrations and ancillaries are currently updated on an ad-hoc basis by the
calibration team.  The calibration team has scripts that are used to
synchronize the calibration collections between different Butler repositories.
In the short term, these scripts could be extended to also synchronize to the
end-user repositories at Google.  We may eventually need a more regimented
system for publication of these datasets, but it is outside the scope of this
tech note.

### Existing unembargo tooling

The [transfer_embargo](https://github.com/lsst-dm/transfer_embargo) repository 
contains scripts that are currently used to move raws and other outputs from
the `embargo` repository to other repositories at USDF.  However, do not
track any state about which files have been transferred (other than by querying
the Butler database).  This will make it difficult to reliably implement the
full lifecycle of these data products across multiple repositories.

(lifecycle)=
## Lifecycle of Prompt Processing Outputs

Prompt processing pods:

1. Generate outputs in a local Butler repository
2. Copy files to central `embargo` Butler repository S3
3. Write file metadata to Kafka for consumption by writer service

Prompt Processing Butler Writer

4. Read file metadata from Kafka and write it to central `embargo` Butler repository Postgres
5. Write list of output datasets to Kafka for consumption by publication service

Prompt Publication Service:

6. Read list of datasets to unembargo from Kafka
7. Copy files and metadata from `embargo` to `prompt_prep` Butler repository
8. Copy metadata from `embargo` to `/repo/main` Butler repository (files shared with `prompt_prep`)
9. Delete datasets from `embargo` Butler repository
10. Copy metadata from `prompt_prep` to end-user `prompt` repository at Google
11. Mark datasets as expired in end-user `prompt` repository at Google
12. Delete files from `prompt_prep` Butler repository
13. Delete files from `/repo/main` Butler repository

### Batched vs incremental release of datasets

## Prompt Publication Service

Most of the work in the publication process would be handled by a new service: the Prompt Publication Service ("PPS").

### State Database

PPS will maintain a database tracking the state of each dataset as it moves
through its lifetime.  Conceptually, the state will look something like the
following:

```{rst-class} technote-wide-content
```

| Dataset ID                           | Dataset Type            | Exposure Time   | embargo status | prompt_prep status | repo_main status | Google status | Google publication time |
| ------------------------------------ | ----------------------- | --------------- | -------------- | ------------------ | ---------------- | ------------- | ----------------------- |
| b8e9d6b8-d961-4eea-bfa0-2dae310df6cf | preliminary_visit_image | 2026-03-15 1am  | PRESENT        |                    |                  |               |                         |
| bdc5e4c9-077e-4eb4-86ff-5cf1b2f0a328 | preliminary_visit_image | 2026-03-11 11pm | DELETED        | PRESENT            | PRESENT          | PRESENT       | 2026-03-14 11pm         |
| b587b20d-0d1a-4b15-9502-24cc61856642 | difference_image        | 2026-02-05 12pm | DELETED        | DELETED            | DELETED          | EXPIRED       | 2026-02-08 1am          |

This is sufficient information to allow the service to efficiently find
datasets that are ready to move to the next state. 

The insertion of new datasets into this database is triggered by a Kafka
message received from the Prompt Processing Butler Writer on the
`prompt-output` topic.  Kafka's transactions can be used to guarantee
at-least-once processing of these messages.

After a task runs to completion, the database is updated with the new state. 
The underlying Butler methods can be made idempotent, so we only need to record
completion of tasks and do not need to persist an "in progress" state in the
database.  In the event of a crash, the service can pick up where it left off
without any special retry logic.

### Task logic

Each of the steps described in [Lifecycle of Prompt Processing
Outputs](#lifecycle) above will be handled as an independent task within the
service.  The general structure of each task is simple:

1. Query the state database to identify datasets ready for the task
2. Execute code to transfer/delete/etc datasets
3. Update the state database

Step #1 is where we would apply policies like "dataset type x should be expired
90 days after publication."

The actual work of Step #2 may be fanned out across multiple workers,
but there will be a single central service that is responsible for enqueueing
work and collecting results from the workers.

### User Interface
A small web app will be written for displaying service status and running
administrative commands.  The Data Curation team will be monitoring and
managing the publication process.  Most of the time, publication will be
hands-off and fully automatic, but unusual situations may arise that need
manual intervention.  The web app will be developed in response to the needs of
Data Curation as we gain experience with this process.

## `prompt` Butler Repository

### Butler Collection Structure

Although many datasets will be deleted from disk after 30 days, we need to
retain a record of all the datasets that have existed in the Butler repository.
This allows users to look up the dataset ID and other information needed to
re-create them.  The following collection structure would allow users to
distinguish datasets that are available for download from those that have been
deleted:

```
Prompt/All                        CHAINED
    Prompt/Available              CHAINED
        LSSTCam/raw/all           RUN
        Prompt/Outputs/Available  TAGGED
        Prompt/Calibs             CHAINED
            ... replicate calibration chain structure from embargo ...
    Prompt/Outputs/Expired        TAGGED
```

PPS would manage the `Prompt/Outputs/Available` and `Prompt/Outputs/Expired`
tagged collections as part of the publication process.

Most tutorials and common use cases would point to `Prompt/Available` as the
default top-level collection.