---
title: "Pre-populating Neo4J using Kubernetes Init Containers and neo4j-admin import"
date: "2020-07-15"
categories: 
  - "devops-sysadmin"
  - "development"
  - "kubernetes"
tags: 
  - "bash"
  - "databases"
  - "docker"
  - "kubernetes"
  - "neo4j"
coverImage: "maxresdefault.jpg"
---

Recently there has been an uptake in the use of Neo4j by the Data Scientists. This is a good thing! they are wanting to use the right tool for the job. However we need to run it inside our k8s cluster as a portable readable data source that has been dynamically populated from a pile of data in a combination of PostgreSQL and MongoDB.

This isn't a problem for them working locally, they install and spin up a local copy of Neo4j and can interact with it quite happily. They even realised that they can generate CSV's from PostgreSQL and MongoDB and then import them, blindingly fast, into Neo4j using the `neo4j-admin` tool that comes bundled. Fantastic!

At least until they come to want to run their Neo instance inside our k8s cluster. That's where I step in and turn them aside from creating their own custom neo4j image with a bespoke entry point that loads all the data for them in some crazy threaded bash scripting!

"No, No, No!" I tell them. "It's far easier to just add an init container to your pod, that will preload the data before Neo starts up".

Init containers, if you haven't come across before, them are a special type of container that lives inside a k8s pod and are set to run **BEFORE** your main container runs. In this case it means we can easily sequence a bash script to run the `neo4j-admin import` before Neo4j is even started. And here is how we did it!

## The script

The data scientists had been using Neo4j 3.5.x locally because they had a need for the graph algorithms plugin ([https://github.com/neo4j-contrib/neo4j-graph-algorithms](https://github.com/neo4j-contrib/neo4j-graph-algorithms)) which at the time they were looking didn't support Neo4j 4.x. The plugin is now deprecated and its replacement (https://github.com/neo4j/graph-data-science) thankfully supports 3.5.x and 4.x.

As Neo4j 4.x introduces a lot of new features and improves performance so I recommended we switch to using that. This meant a refactor of their bash script for neo4j-admin there some very subtle differences and a few caveats to work with. This is what they came up with

```bash
#!/bin/bash
DBNAME="neo4j"
if [ "$#" -eq 1 ]; then
    DBNAME=$1
fi

# extract data from SQL
python3 extract_data.py

# remove old db for rebuild
rm -rf "/data/databases/$DNBAME"

neo4j-admin import \
    --database=$DBNAME \
    --delimiter="|" \
    --nodes=Protein=${NODE_DIR}/nodes_protein_header.csv,${DATA_DIR}/nodes_proteins.csv \
    --nodes=UniProtKB=${NODE_DIR}/nodes_uniprot_header.csv,${DATA_DIR}/nodes_uniprot.csv \
    --relationships=HAS_AMINO_ACID_SEQUENCE=${EDGE_DIR}/edges_protein_sequence_header.csv,${DATA_DIR}/edges_protein_sequence.csv \    
    --relationships=HAS_AMINO_ACID_SEQUENCE=${EDGE_DIR}/edges_chembl_protein_biotherapeutic_molregno_header.csv,${DATA_DIR}/edges_chembl_protein_biotherapeutic_molregno.csv \
    --skip-bad-relationships=true \
    --skip-duplicate-nodes=true
```

The `import` command here is significantly shorter for example purposes, as the original is about 120 lines long. As you can see it's pretty straight forward, they had another script in `extract_data.py`, that I wont bore you with suffice to say that it pulled out all the data they wanted from PostgreSQL and MongoDB, which got saved to disk as CSV files in the relevant directories.

Great, it worked on their local version!

## The Dockerfile

```
ROM neo4j:latest
ENV NEO4JLABS_PLUGINS ["graph-data-science"]
RUN apt update && apt install -y python3
WORKDIR /srv
COPY src /srv/src
COPY headers /srv/headers
```

The plan is always to keep it simple. We have one image that we can run for both the init container and the main container. This docker file gives a vanilla neo4j instance with python and our scripts for extracting the data loaded into it

## The k8s Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: neo4j
spec:
  containers:
  - name: neo4j
    env:
    - name: NEO4J_AUTH
      value: neo4j/password
    image: registry.example.com/phpboyscout/rnd_graph:latest
    imagePullPolicy: Always
    volumeMounts:
    - mountPath: /data
      name: neo4j
      subPath: data
  initContainers:
  - name: importer
    args:
    - neo4j_import.sh
    command:
    - /bin/bash
    env:
    - name: DATA_DIR
      value: /import/data
    - name: HEADER_DIR
      value: /srv/headers
    image: registry.example.com/phpboyscout/rnd_graph:latest
    imagePullPolicy: Always
    stdin: true
    workingDir: /srv/src
    volumeMounts:
    - mountPath: /data
      name: neo4j
      subPath: data
    - mountPath: /import
      name: neo4j
      subPath: import
  - name: neo4j
    persistentVolumeClaim:
      claimName: neo4j
```

Now we can pull it all together with our k8s manifest. From here you can see that we have our default neo4j container that we pass in our default authentication details to and an init container that runs our `import.sh` script. Both containers have access to a shared volume for the `/import` and `/data` folders.

And now we get to...

## Troubleshooting

So right off the bat it didn't work! No surprises there but here are a few things that caused us some issues and how we resolved them.

### Database offline

At first glance everything seemed to work. Until we tried to connect to the `neo4j` database with the default UI, at which point we were presented with the error message

```
Database "neo4j" is unavailable, its status is "offline."
```

This took a little sleuthing and shelling into the neo4j container to take a look at the `/var/debug.log` file which gives significantly more useful information about whats going on with the server. First we were getting stack traces that contained messages like

```
Component 'org.neo4j.kernel.impl.transaction.log.files.TransactionLogFiles@59d6a4d1'
was successfully initialized, but failed to start. Please see the attached cause 
exception "/data/transactions/neo4j/neostore.transaction.db.0"
```

From experience this sounded like a permissions issue and lo and behold, checking the files on the filesystem showed that because the import script was run as root the database files were owned by root. We resolved this by adding:-

```bash
chown -R neo4j:neo4j /data/
```

to the bottom of the import script. Next we were then presented with an error that looked like

```
2020-07-14 16:56:33.919+0000 WARN [o.n.k.d.Database] [neo4j] Exception occurred while
starting the database. Trying to stop already started components. Mismatching store id.
```

This one seems like it would be an obvious one to google and I did come up with few pages that seemed to describe what was happening to me but gave some varied solutions, from starting and stopping the sever and running `neo4j-admin unbind` in between to deleting various files. It seemed very strange because we did test this with the 3.5.17 version of Neo and it worked fine.

The solution we needed was to wipe the slate clean properly. The line in our script to remove the previous build of the db

```bash
# remove old db for rebuild
rm -rf "/data/databases/$DNBAME"
```

just didn't cut it. It turns out that because the 4.x version of Neo4j supports multiple databases the `import` command writes additional information to the system database and transactions database in the form of some identifiers for each database, BUT if you don't do something to clear that value for the database your are building it wont match up when the server starts and you get a declaration of `Mismatching store id`

I'm not sure if the developers are aware of this flaw, so in the mean time we have to expand our cleanup to:

```bash
# clean up for fresh import
rm -rf /data/databases/*
rm -rf /data/transactions/*
```

removing the neoj4, system and store\_lock databases and transaction logs from the data store. This solved the problem and the server was able to start and we could connect to neo4j database successful.

Its not an ideal solution, I can foresee definite situations we will have to work around when we get to a point where multiple databases may be needed and are built separately and independently from each other. but it will suffice for now.

### Malloc(): Error message goes here

Once it was up and running we noticed that we were getting lots of restarts on the main neo4j container a quick look at the `stdout` log and we could see each restart ending with something that looked like

```
malloc(): corrupted top size
```

instantly this looks like an issue with memory sizing inside the container for the JVM. Thankfully the team at Neo4j have accounted for this and give you a nice tool in the form of

```
neo4j-admin memrec
```

which interrogates the databases and gives some sensible values you can set in the output which in our case looked like

```

# Memory settings recommendation from neo4j-admin memrec:
#
# Assuming the system is dedicated to running Neo4j and has 376.6GiB of memory,
# we recommend a heap size of around 31g, and a page cache of around 331500m,
# and that about 22400m is left for the operating system, and the native memory
# needed by Lucene and Netty.
#
# Tip: If the indexing storage use is high, e.g. there are many indexes or most
# data indexed, then it might advantageous to leave more memory for the
# operating system.
#
# Tip: Depending on the workload type you may want to increase the amount
# of off-heap memory available for storing transaction state.
# For instance, in case of large write-intensive transactions
# increasing it can lower GC overhead and thus improve performance.
# On the other hand, if vast majority of transactions are small or read-only
# then you can decrease it and increase page cache instead.
#
# Tip: The more concurrent transactions your workload has and the more updates
# they do, the more heap memory you will need. However, don't allocate more
# than 31g of heap, since this will disable pointer compression, also known as
# "compressed oops", in the JVM and make less effective use of the heap.
#
# Tip: Setting the initial and the max heap size to the same value means the
# JVM will never need to change the heap size. Changing the heap size otherwise
# involves a full GC, which is desirable to avoid.
#
# Based on the above, the following memory settings are recommended:
dbms.memory.heap.initial_size=31g
dbms.memory.heap.max_size=31g
dbms.memory.pagecache.size=331500m
#
# It is also recommended turning out-of-memory errors into full crashes,
# instead of allowing a partially crashed database to continue running:
#dbms.jvm.additional=-XX:+ExitOnOutOfMemoryError
#
# The numbers below have been derived based on your current databases located at: '/var/lib/neo4j/data/databases'.
# They can be used as an input into more detailed memory analysis.
# Total size of lucene indexes in all databases: 0k
# Total size of data and native indexes in all databases: 17300m
```

So how to get these values into the container... Thankfully this is handled for you in the form of Environment Variables you can pass into the docker image. A bit of a google and i found this little snippet which is a goldmine for telling us how to translate settings into environment variables.

```
# Env variable naming convention:
# - prefix NEO4J_
# - double underscore char '__' instead of single underscore '_' char in the setting name
# - underscore char '_' instead of dot '.' char in the setting name
# Example:
# NEO4J_dbms_tx__log_rotation_retention__policy env variable to set
#       dbms.tx_log.rotation.retention_policy setting
```

As for getting the variables into the container, you could do this from the pod and inject it in. I this case because the data we are going to be using is reasonably stable and tested we decided to stick them into the Docker file with the `ENV` directive.

```
ENV NEO4J_dbms_memory_heap_initial__size 31g
ENV NEO4J_dbms_memory_heap_max__size 31g
ENV NEO4J_dbms_memory_pagecache_size 331500m
```

And so far we haven't had a restart yet!
