# Pipeline 2 Documentation

VFB Pipeline 2 comprises five servers/services and six data pipelines:

- Pipeline 2 _servers_:
  - VFB knowledge base (`vfb-kb`)
  - VFB triple store (`vfb-triplestore`) 
  - SOLr + preconfigured VFB SOLr core (`vfb-solr`)
  - owlery (`vfb-owlery`)
  - VFB Neo4J production instance (`vfb-prod`)
- Pipeline 2 _data pipelines_:
  - Transform KB1 to KB2 (`vfb-kb2kb`) [to be obsoleted]
  - Validate KB (`vfb-validate`)
  - Data collection (`vfb-collect-data`) 
  - Triple store ingestion (`vfb-updatetriplestore`)
  - Data transformation and dumps for production instances (`vfb-dumps`)
  - VFB production instance ingestion (`vfb-update-prod`)

Server and data pipelines are combined into 6 general sub-pipelines which are configured as Jenkins jobs (currently located [here](https://jenkins.virtualflybrain.org/view/pip_pipeline2/)). This documentation describes all 6 sub-pipelines in detail, including which role the individual servers and data pipelines play. All high-level documentation including images can be found on the [vfb-pipeline-config repo](https://github.com/VirtualFlyBrain/vfb-pipeline-config). Note: There was once a pipeline server named `vfb-integration-api` which has since been discarded in favour of `vfb-dumps`.

![Pipeline Overview](pipeline-overview.png)

## Sub-pipeline: Deploy KB (pip_vfb-kb)

- _Summary_: This pipeline loads the current KB from backup, applies a series of transformation steps and validates the resulting version of the KB for VFB Schema compliance. The finalised KB is backed up, and spun up, again from backup to clear caches. Components:
  - `vfb-kb` (deployment of the VFB knowledge base)
  - `vfb-kb2kb` (provisional data pipeline managing the migration from KB1 to KB2)
  - `vfb-validate` (validation pipeline to check if KB2 is in the correct basic shape for neo4j2owl)
- [Jenkins job](https://jenkins.virtualflybrain.org/view/pip_pipeline2/job/pip_vfb-kb/)
- _Dependents_: pip-triplestore

### Service: vfb-kb
- _Image_: virtualflybrain/docker-neo4j-knowledgebase:neo2owl ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/docker-neo4j-knowledgebase/builds))
- _Git_: https://github.com/VirtualFlyBrain/docker-neo4j-knowledgebase
- [Dockerfile](https://github.com/VirtualFlyBrain/docker-neo4j-knowledgebase/blob/neo2owl/Dockerfile)
- [Jenkins job](https://jenkins.virtualflybrain.org/view/pip_pipeline2/job/pip_vfb-kb)
- _Summary_: The VFB KB instance loads the [VFB KB Archive](http://data.virtualflybrain.org/archive/VFB-KB.tar.gz) and deploys it as a Neo4J instance that includes the [neo2owl plugin](https://github.com/VirtualFlyBrain/neo4j2owl). This plugin allows loading OWL ontologies into Neo4J according to a [specific schema](https://github.com/VirtualFlyBrain/neo4j2owl/blob/master/README.md), as well as serialising the (valid) Neo4J graph into OWL.
- _Access_: http://kbl.p2.virtualflybrain.org/browser/ (post pipeline), http://kb.p2.virtualflybrain.org/browser/ (spin off from backup)

#### Detailed notes on vfb-kb

- There is nothing specifically important about `vfb-kb`, other than that it comes in two flavours. The pipeline spins of a KB instance which gets a tiny bit of pre-processing in `vfb-collectdata` (basically setting the labels correctly). This instance is spun up from backup, only for the pipeline, run, and thrown away after the pipeline is finished. It is important to note that this system means that `vfb-kb` _pipeline edition_ is not necessarily the exact same as `vfb-kb` _curation edition_ - `vfb-kb` _pipeline edition_ corresponds to `vfb-kb` _curation edition_ at the _time of the last backup_. So, if you want them to correspond exactly, you need to make sure the backup step is run right before vfb-kb is spun up.
- Currently the Dockerfile with the neo4j2owl plugin (and APOC!) is on a [branch](https://github.com/VirtualFlyBrain/docker-neo4j-knowledgebase/blob/neo2owl/Dockerfile)! So be careful when you merge this in!

### Data pipeline: vfb-kb2kb [provisional]

- _Image_: virtualflybrain/vfb-pipeline-kb2kb:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-kb2kb/builds))
- _Git_: https://github.com/VirtualFlyBrain/vfb-pipeline-kb2kb
- [Dockerfile](https://github.com/VirtualFlyBrain/vfb-pipeline-kb2kb/blob/master/Dockerfile)
- [Jenkins job](https://jenkins.virtualflybrain.org/view/pip_pipeline2/job/pip_vfb-kb)
- _Summary_: The image encapsulates a (python/cypher-based) pipeline to transform the original version of the KB into a [schema-compliant](https://github.com/VirtualFlyBrain/neo4j2owl/blob/master/README.md) version.
 

#### Detailed notes on vfb-kb2kb

- Currently, in order to perform the KB2KB migration, a [table](https://github.com/VirtualFlyBrain/VFB_neo4j/blob/kbold2new/src/uk/ac/ebi/vfb/neo4j/data_sig_vfb.csv) is required to decide what type an entity is. This is currently _on a branch in the pipeline repo_, which needs to be taken into account when the pipeline is merged all in. This can probably be merged in, but sould be done with the usual care (pull request, review that nothing important has changed accidentally - this branch has been created years ago).
- The script that performs the change is on an [unmerged branch](https://github.com/VirtualFlyBrain/VFB_neo4j/blob/kbold2new/src/uk/ac/ebi/vfb/neo4j/neo4j_kb_old2new.py) in the VirtualFlyBrain/VFB_neo4j repo.
- The script _should be obsoleted_ once the migration to KB2 is completed


### Data pipeline: vfb-validate

- _Image_: virtualflybrain/vfb-pipeline-validatekb:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-validatekb/builds))
- [Dockerfile](https://github.com/VirtualFlyBrain/vfb-pipeline-validatekb/blob/master/Dockerfile)
- _Git_: https://github.com/VirtualFlyBrain/vfb-pipeline-validatekb
- [Jenkins job](https://jenkins.virtualflybrain.org/view/pip_pipeline2/job/pip_vfb-kb)
- _Summary_: The image encapsulates a (python/cypher-based) pipeline to check whether the current state of the KB is [schema-compliant](https://github.com/VirtualFlyBrain/neo4j2owl/blob/master/README.md).
- Results of the validation can be read on the latest console output of the Jenkins `pip_vfb-kb` pipeline.

#### Detailed notes on vfb-validate

- The actual validation process is implemented as a [python script](https://github.com/VirtualFlyBrain/VFB_neo4j/blob/kbold2new/src/uk/ac/ebi/vfb/neo4j/neo4j_kb_validateschema.py). It is not complete in terms of validation, but it is doing a few things, like checking that every node has at least one owl base type, and iri and so on. Each test is a single function in the script, so it should be fairly easy to read them over.
- The tests are implemented as a report that is printed as part of the Jenkins job - currently the _pipeline does not break_ if there is a validation error!

## Sub-pipeline: Deploy triplestore (pip_vfb-triplestore)

- _Summary_: This pipeline deploys an empty triplestore, collects all VFB relevant data (including KB and ontologies), and pre-processes and loads the collected data into the triplestore. Components:
  - `vfb-triplestore` (deploying triplestore)
  - `vfb-collect-data` (data collection and preprocessing pipeline for all VFB data)
  - `vfb-update-triplestore` (loading collected data into the triplestore)
- [Jenkins job](https://jenkins.virtualflybrain.org/view/pip_pipeline2/job/pip_vfb-triplestore/)
- _Depends on_: `pip-kb`
- _Dependents_: `pip-dumps`

### Service: vfb-triplestore

- _Image_: yyz1989/rdf4j:latest ([dockerhub](https://hub.docker.com/repository/docker/yyz1989/rdf4j/builds))
- _Git_: We do not maintain this, see [ticket](https://github.com/VirtualFlyBrain/vfb-pipeline-triplestore/issues/2)
- _Summary_: The triplestore is currently an unspectacular default implementation of rdf4j-server. We make use of a simple in-memory store that is configured [here](https://github.com/VirtualFlyBrain/vfb-pipeline-triplestore/blob/master/rdf4j_vfb.txt). The container is maintained elsewhere (see docker-hub pages of image for details).

#### Detailed notes on vfb-triplestore

- _Triplestore access_:
  - [Example](http://ts.p2.virtualflybrain.org/rdf4j-workbench/repositories/vfb/query?action=exec&queryLn=SPARQL&query=PREFIX%20%3A%20%3Chttp%3A%2F%2Fwww.test.com%2Fns%2Ftest2%23%3E%0A%0ACONSTRUCT%20%7B%20%3Fx%20%3Fp%20%3Fy%20.%20%7D%0A%0AWHERE%20%7B%3Fx%20%3Fp%20%3Fy%20.%7D%0ALIMIT%2010&limit_query=100&infer=true&) SPARQL query agains UI
  - [Repo summary](http://ts.p2.virtualflybrain.org/rdf4j-workbench/repositories/vfb/summary)
- We should probably migrate away from this particular image of rdf4j towards our own VFB one, because there is a danger that this container gets removed/updated causing problems for us (though not likely);

### Data pipeline: vfb-collect-data

- _Image_: virtualflybrain/vfb-pipeline-collectdata:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-collectdata/builds))
- _Git_: https://github.com/VirtualFlyBrain/vfb-pipeline-collectdata
- [Dockerfile](https://github.com/VirtualFlyBrain/vfb-pipeline-collectdata/blob/master/Dockerfile)
- _Summary_: This container encapsulates a process that downloads a number of source ontologies, obtains the OWL version of the VFB KB, and applies a number of ROBOT-based pre-processing steps, in particular: extracting modules/slices of external ontologies, running consistency checks and serialising as ttl for quicker ingest into triplestore. It also contains the _data embargo_ pipeline and has some provisions for _shacl validation_.

#### neo4j2owl:exportOWL()

- Exporting the KB into OWL2 is managed through a custom procedure (`exportOWL()`) implemented in the [neo4j2owl plugin](https://github.com/VirtualFlyBrain/neo4j2owl).
- The plugin is documented in detail in the [repos readme](https://github.com/VirtualFlyBrain/neo4j2owl/blob/master/README.md)

#### Detailed notes on vfb-collect-data

- The process is encoded [here](https://github.com/VirtualFlyBrain/vfb-pipeline-collectdata/blob/master/process.sh). It performs the following steps:
  1. Exporting KB to OWL using the above `neo4j2owl:exportOWL()` procedure.
  1. Removing embargoed data. The technique applied here is based on using ROBOT query and encoding the embargo logic as SPARQL queries (combined with `ROBOT remove`).
  1. Downloading external ontologies.
  1. Ontologies in [vfb_fullontologies.txt](https://github.com/VirtualFlyBrain/vfb-pipeline-collectdata/blob/master/vfb_fullontologies.txt) are imported in their entirety.
  1. Ontologies in [vfb_slices.txt](https://github.com/VirtualFlyBrain/vfb-pipeline-collectdata/blob/master/vfb_slices.txt) are sliced. The slice corresponds to a BOTTOM module that has the combined signature of all ontologies in the fullontologies section with the signature of the KB. 
     - Note: there is an annoying [hack](https://github.com/VirtualFlyBrain/vfb-pipeline-collectdata/blob/1afcd5860c0cfbe3b7f2471b2a1545a1ed8bafbd/process.sh#L96) in there that should be fixed, simply by removing the if/else in this code block. First though, we need to understand why this process is so slow (ROBOT memory?).
  1. All ontologies are converted to turtle.
  1. The KB is checked using a SHACL validation engine.
  1. All ontologies ready to be imported into the triplestore are gzipped.
 
### Data pipeline: vfb-update-triplestore

- _Image_: virtualflybrain/vfb-pipeline-updatetriplestore:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-updatetriplestore/builds))
- [Dockerfile](https://github.com/VirtualFlyBrain/vfb-pipeline-updatetriplestore/blob/master/Dockerfile)
- _Git_: https://github.com/VirtualFlyBrain/vfb-pipeline-updatetriplestore
- _Summary_: This container encapsulates a process that (1) sets up the triplestores vfb database and (2) loads all of the ttl files generated by vfb-collect-data into the vfb-triplestore. The image contains the configuration details of triplestore, like choice of triplestore engine.

#### Detailed notes on vfb-update-triplestore:

- The [process](https://github.com/VirtualFlyBrain/vfb-pipeline-updatetriplestore/blob/master/process.sh) really does nothing more other than loading the ontologies and data collected in the previous step into the triple store.

## Sub-pipeline: Data transformation and dumps for production instances (pip_vfb-pipeline-dumps)

- _Summary_: This pipeline transforms the knowledge graph in the triplestore into various custom dumps used by downstream services such as the VFB Neo4J production instance, owlery and solr.
- [Jenkins pipeline](https://jenkins.virtualflybrain.org/view/pip_pipeline2/job/pip_vfb-pipeline-dumps/)
- _Depends on_: pip-triplestore
- _Dependents_: pip-owlery, pip-prod

### Data pipeline: vfb-dumps

- _Image_: virtualflybrain/vfb-pipeline-dumps:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-dumps/builds))
- _Git_: https://github.com/VirtualFlyBrain/vfb-pipeline-dumps
- _Summary_: The VFB dumps pipeline access the triple store to obtain data dumps that in mungs, transforms and enriches for various downstream purposes such as vfb-prod ingestion, owlery ingestion and vfb-solr ingestion.
- [Dockerfile](https://github.com/VirtualFlyBrain/vfb-pipeline-dumps/blob/master/Dockerfile)
- _Example access_: http://virtualflybrain.org/data/VFB/OWL contains all the data that is generated by this pipeline. This generated is loaded into the various downstream tools.

#### Detailed notes on vfb-dumps

- The [process](https://github.com/VirtualFlyBrain/vfb-pipeline-dumps/blob/master/process.sh) performs the following steps (all encoded in the [Makefile](https://github.com/VirtualFlyBrain/vfb-pipeline-dumps/blob/master/dumps.Makefile)):
  1. Build dump for `vfb-owlery` (all logical axioms in triplestore)
  1. Build dump for `vfb-prod` (VFB production instance)
  1. Build dump for `vfb-solr` (special json file, created using python)
- There is a new section in the [config file](https://github.com/VirtualFlyBrain/vfb-prod/blob/6427bf8c41401d9978f76d601e020943536006f0/neo4j2owl-config.yaml#L159) called filters which should be pretty self explanatory. The main thing to know is that the ['iri_prefix'] [filter](https://github.com/VirtualFlyBrain/vfb-pipeline-dumps/blob/5aa5d27e89fd442b3b9635bb3e851c24590906eb/scripts/obographs-solr.py#L86) actually checks whether the listed string is contained somewhere in the IRI - so in our case, VFBc_ would have worked as well. The ['neo4j_node_label'] simply filters out every entity that also has a particular node label associated with it.
- The vfb-solr pipeline is a bit more involved and also relies on the general pipeline [config file](https://github.com/VirtualFlyBrain/vfb-prod/blob/master/neo4j2owl-config.yaml).
- There is a new section in the dumps.Makefile (around line 66) that allows adding arbitrary sparql construct queries to the produced dumps. This can be useful, for example, to materialise ad hoc neo labels. Add a new dump:
  1. pick name, add to the correct DUMPS variable (DUMPS_SOLR, DUMPS_PDB, DUMPS_OWLERY)
  2. create new sparql query in sparql/, naming it 'construct_name.sparql', e.g. sparql/construct_image_names.sparql
  Note that non-sparql goals, like 'inferred_annotation', need to be added separately.

## Sub-pipeline: Deploy Owlery (pip_vfb-owlery, Service)

- _Summary_: This pipeline deploys the Owlery webservice which is used by VFB to answer ontology queries (no special config).
- _Depends on_: `vfb-dumps`
- _Dependents_: None (gepetto)

### Service: vfb-owlery

- _Image_: virtualflybrain/owlery-vfb:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/owlery-vfb/builds))
- _Git_: https://github.com/VirtualFlyBrain/owlery-vfb
- [Dockerfile](https://github.com/VirtualFlyBrain/owlery-vfb/blob/master/Dockerfile)
- _Summary_: Deployment of [Owlery](https://owlery.docs.apiary.io/#), a web-service for accessing basic reasoning methods of an ontology. 
- _Example access_: Get [subclasses](http://owl.ps2.virtualflybrain.org/kbs/vfb/superclasses?object=%3Chttp://purl.obolibrary.org/obo/FBbt_00005774%3E&direct=true) of a term


## Sub-pipeline: VFB prod (pip_vfb-prod)

- _Summary_: This pipeline deploys the production instance of the VFB neo4j database and loads all the relevant data.
- _Depends on_: pip-integratio
- [Jenkins pipeline](https://jenkins.virtualflybrain.org/view/pip_pipeline2/job/pip_vfb-prod/)
- _Dependents_: None (gepetto)


### Service: vfb-prod

- _Image_: virtualflybrain/vfb-prod:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-prod/builds))
- _Git_: https://github.com/VirtualFlyBrain/vfb-prod
- [Dockerfile](https://github.com/VirtualFlyBrain/vfb-prod/blob/master/Dockerfile)
- _Summary_: Deploys an empty, configured instance of a Neo4J database with the [neo2owl plugin](https://github.com/VirtualFlyBrain/neo4j2owl), and APOC tools.
- _Access_: http://pdb.p2.virtualflybrain.org/browser/
- Note that this image is _used for all runtime deployments of neo4j databases_ across the VFB ecosystem (check branches of the vfb-prod container)

### Data pipeline: vfb-update-prod

- _Image_: virtualflybrain/vfb-pipeline-update-prod:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-update-prod/builds))
- _Git_: https://github.com/VirtualFlyBrain/vfb-pipeline-update-prod
- [Dockerfile](https://github.com/VirtualFlyBrain/vfb-pipeline-update-prod/blob/master/Dockerfile)
- _Summary_: The update-prod container currently takes an ontology (from the integration layer) and loads it into the the Neo4J instance (vfb-prod) using the neo2owl plugin. Process"
  1. Loading the ontology using the `neo4j2owl:owl2Import()` procedure
  1. Setting a number of indices (see detailed notes below). 

#### Detailed notes about vfb-update-prod

- You can set additional Pipeline post-processing steps like indices by editing [this file](https://github.com/VirtualFlyBrain/vfb-pipeline-update-prod/blob/master/pdb_set_indices.neo4j). Note that this file can be used to set arbitrary post-processing cypher queries, not just indices (contrary to the file name). Essentially, all list cypher queries are executed in order right after PDB import is completed.
- The possible configuration settings for the `neo4j2owl:owl2Import()` procedure are described [here](https://github.com/VirtualFlyBrain/neo4j2owl#configuration-of-neo4j2owl). The configuration is stored [here](https://github.com/VirtualFlyBrain/vfb-prod/blob/master/neo4j2owl-config.yaml).

## Sub-pipeline VFB SOLr (pip_vfb-solr, Service)

- _Image_: virtualflybrain/vfb-solr ([dockerhub](https://hub.docker.com/r/virtualflybrain/vfb-solr))
- _Git_: https://github.com/VirtualFlyBrain/vfb-solr
- [Dockerfile](https://github.com/VirtualFlyBrain/vfb-solr/blob/master/Dockerfile)
- _Summary_: an essentially unchanged solr 8 image instance
- [Jenkins pipeline](https://jenkins.virtualflybrain.org/view/pip_pipeline2/job/pip_vfb-solr/)

# Deployment during development phase:

1. The pipeline is currently deployed as a series of connected [Jenkis jobs](https://jenkins.virtualflybrain.org/view/pip_pipeline2/). 
1. Every sub-pipeline has a Jenkins job that can be restarted manually. Every sub-pipeline will trigger all of its dependents. So if the `pip_vfb-dumps` pipeline is started, it will automatically trigger the `pip_vfb-prod` and `pip_vfb-owlery` pipelines to redeploy as well.  
1. The whole pipeline can be restarted by simply triggering the `pip_vfb_kb` pipeline to be re-run. This will trigger all downstream sub-pipelines.
1. The whole pipeline is re-run every night at 4am.