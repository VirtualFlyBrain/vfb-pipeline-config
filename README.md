# Pipeline 2 Documentation

VFB Pipeline 2 comprises five servers and five data pipelines:

- Pipeline 2 _servers_:
  - VFB knowledge base (vfb-kb)
  - VFB triple store (vfb-triplestore) 
  - SOLr + preconfigured VFB SOLr core (vfb-solr)
  - owlery (vfb-owlery)
  - VFB Neo4J production instance (vfb-prod)
- Pipeline 2 _data pipelines_:
  - Transform KB1 to KB2 (vfb-kb2kb)
  - Validate KB (vfb-validate)
  - Data collection (vfb-collect-data) 
  - Triple store ingestion (vfb-updatetriplestore)
  - Data transformation and dumps for production instances (vfb-dumps)
  - VFB production instance ingestion (vfb-update-prod)

Server and data pipelines are combined into 6 general sub-pipelines which are configured as Jenkins jobs (currently located [here](https://jenkins.virtualflybrain.org/view/pip_pipeline2/)). This documentation describes all 6 sub-pipelines in detail, including which role the individual servers and data pipelines play. All high-level documentation including images can be found on the [vfb-pipeline-config repo](https://github.com/VirtualFlyBrain/vfb-pipeline-config). Note: There was once a pipeline server named `vfb-integration-api` which has since been discarded in favour of `vfb-dumps`.

<!--![Pipeline Overview](pipeline-overview.png)-->


## Sub-pipeline: Deploy KB (pip_vfb-kb)

- Summary: This pipeline loads the current KB from backup, applies a series of transformation steps and validates the resulting version of the KB for VFB Schema compliance. The finalised KB is backed up, and spun up, again from backup to clear caches.
- Jenkins: https://jenkins.virtualflybrain.org/view/pip_pipeline2/job/pip_vfb-kb/
- Dependents: pip-triplestore

1. vfb-kb
   * Image: virtualflybrain/docker-neo4j-knowledgebase:neo2owl ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/docker-neo4j-knowledgebase/builds))
   * Git: https://github.com/VirtualFlyBrain/docker-neo4j-knowledgebase
   * Summary: The VFB KB instance loads the [VFB KB Archive](http://data.virtualflybrain.org/archive/VFB-KB.tar.gz) and deploys it as a Neo4J instance that includes the [neo2owl plugin](https://github.com/VirtualFlyBrain/neo4j2owl). This plugin allows loading OWL ontologies into Neo4J according to a [specific schema](https://github.com/VirtualFlyBrain/neo4j2owl/blob/master/README.md), as well as serialising the (valid) Neo4J graph into OWL.
   * Access: http://kbl.p2.virtualflybrain.org/browser/ (post pipeline), http://kb.p2.virtualflybrain.org/browser/ (spin off from backup)
2. vfb-kb2kb [one-off]
   * Image: virtualflybrain/vfb-pipeline-kb2kb:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-kb2kb/builds))
   * Git: https://github.com/VirtualFlyBrain/vfb-pipeline-kb2kb
   * Summary: The image encapsulates a (python/cypher-based) pipeline to transform the original version of the KB into a [schema-compliant](https://github.com/VirtualFlyBrain/neo4j2owl/blob/master/README.md) version.
   * Notes
     * Currently, in order to perform the KB2KB migration, a [table](https://github.com/VirtualFlyBrain/VFB_neo4j/blob/kbold2new/src/uk/ac/ebi/vfb/neo4j/data_sig_vfb.csv) is required to decide what type an entity is. This is currently _on a branch in the pipeline repo_, which needs to be taken into account when the pipeline is merged all in. This can probably be merged in, but be done with some care.
3. vfb-validate [one-off]
   * Image: virtualflybrain/vfb-pipeline-validatekb:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-validatekb/builds))
   * Git: https://github.com/VirtualFlyBrain/vfb-pipeline-validatekb
   * Summary: The image encapsulates a (python/cypher-based) pipeline to check whether the current state of the KB is [schema-compliant](https://github.com/VirtualFlyBrain/neo4j2owl/blob/master/README.md).
   * Results of the validation can be read on the latest console output of the Jenkins `pip_vfb-kb` pipeline.

## Sub-pipeline: Deploy triplestore (pip_vfb-triplestore)

Summary: This pipeline deploys an empty triplestore, collects all VFB relevant data (including KB), and pre-processes and loads the collected data into the triplestore.
Depends on: pip-kb
Dependents: pip-dumps

1. vfb-triplestore
   * Image: yyz1989/rdf4j:latest ([dockerhub](https://hub.docker.com/repository/docker/yyz1989/rdf4j/builds))
   * Git: We do not maintain this, see [ticket](https://github.com/VirtualFlyBrain/vfb-pipeline-triplestore/issues/2)
   * Summary: The triplestore is currently an unspectacular default implementation of rdf4j-server. The container is maintained elsewhere (see docker-hub pages of image for details). 
   * Example access:
     * [Example](http://ts.p2.virtualflybrain.org/rdf4j-workbench/repositories/vfb/query?action=exec&queryLn=SPARQL&query=PREFIX%20%3A%20%3Chttp%3A%2F%2Fwww.test.com%2Fns%2Ftest2%23%3E%0A%0ACONSTRUCT%20%7B%20%3Fx%20%3Fp%20%3Fy%20.%20%7D%0A%0AWHERE%20%7B%3Fx%20%3Fp%20%3Fy%20.%7D%0ALIMIT%2010&limit_query=100&infer=true&) SPARQL query agains UI
     * [Repo summary](http://ts.p2.virtualflybrain.org/rdf4j-workbench/repositories/vfb/summary)
   * Notes:
     * We should probably migrate away from this particular image of rdf4j towards our own VFB one, because there is a danger that this container gets removed/updated causing problems for us (though not likely); .
2. vfb-collect-data [one-off]
   * Image: virtualflybrain/vfb-pipeline-collectdata:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-collectdata/builds))
   * Git: https://github.com/VirtualFlyBrain/vfb-pipeline-collectdata
   * Summary: This container encapsulates a process that downloads a number of source ontologies, obtains the OWL version of the VFB KB, and applies a number of ROBOT-based pre-processing steps, in particular: extracting modules/slices of external ontologies, running consistency checks and serialising as ttl for quicker ingest into triplestore. It also contains the _data embargo_ pipeline and has some provisions for _shacl validation_. 
3. vfb-update-triplestore [one-off]
   * Image: virtualflybrain/vfb-pipeline-updatetriplestore:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-updatetriplestore/builds))
   * Git: https://github.com/VirtualFlyBrain/vfb-pipeline-updatetriplestore
   * Summary: This container encapsulates a process that (1) sets up the triplestores vfb database and (2) loads all of the ttl files generated by vfb-collect-data into the vfb-triplestore. The image contains the configuration details of triplestore, like choice of triplestore engine.

## Sub-pipeline: Data transformation and dumps for production instances (pip_vfb-pipeline-dumps)

Summary: This pipeline deploys the REST integration layer, which contains methods for obtaining relevant VFB-specific subsets from the VFB triplestore.
Depends on: pip-triplestore
Dependents: pip-owlery, pip-prod

1. vfb-dumps
   * Image: virtualflybrain/vfb-pipeline-dumps:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-dumps/builds))
   * Git: https://github.com/VirtualFlyBrain/vfb-pipeline-dumps
   * Summary: The VFB dumps pipeline access the triple store to obtain data dumps that in mungs, transforms and enriches for various downstream purposes such as vfb-prod ingestion, owlery ingestion and vfb-solr ingestion.
   * Example access:
     * http://virtualflybrain.org/data/VFB/OWL contains all the data that is generated by this pipeline. This generated is loaded into the various downstream tools.
     

## Sub-pipeline: Deploy Owlery (pip_vfb-owlery)

Summary: This pipeline deploys the Owlery webservice which is used by VFB to answer ontology queries.
Depends on: pip-integration
Dependents: None (gepetto)

1. vfb-owlery
   * Image: virtualflybrain/owlery-vfb:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/owlery-vfb/builds))
   * Git: https://github.com/VirtualFlyBrain/owlery-vfb
   * Summery deployment of [Owlery](https://owlery.docs.apiary.io/#), a web-service for accessing basic reasoning methods of an ontology. 
   * Example access:
     * Get [subclasses](http://owl.ps2.virtualflybrain.org/kbs/vfb/superclasses?object=%3Chttp://purl.obolibrary.org/obo/FBbt_00005774%3E&direct=true) of a term


## Sub-pipeline: VFB prod (pip_vfb-prod)

Summary: This pipeline deploys the production instance of the VFB neo4j database and loads all the relevant data.
Depends on: pip-integration
Dependents: None (gepetto)

1. vfb-prod
   * Image: virtualflybrain/vfb-prod:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-prod/builds))
   * Git: https://github.com/VirtualFlyBrain/vfb-prod
   * Summary: Deploys an empty, configured instance of a Neo4J database with the [neo2owl plugin](https://github.com/VirtualFlyBrain/neo4j2owl).
   * Access: http://pdb.p2.virtualflybrain.org/browser/
2. vfb-update-prod [one-off]
   * Image: virtualflybrain/vfb-pipeline-update-prod:latest ([dockerhub](https://hub.docker.com/repository/docker/virtualflybrain/vfb-pipeline-update-prod/builds))
   * Git: https://github.com/VirtualFlyBrain/vfb-pipeline-update-prod
   * Summary: The update-prod container currently takes an ontology (from the integration layer) and loads it into the the Neo4J instance (vfb-prod) using the neo2owl plugin.
   
## Sub-pipeline VFB SOLr (pip_vfb-solr)


# Deployment during development phase:

1. The pipeline is currently deployed as a series of connected [Jenkis jobs](https://jenkins.virtualflybrain.org/view/pip_pipeline2/). 
1. Every sub-pipeline has a Jenkins job that can be restarted manually. Every sub-pipeline will trigger all of its dependents. So if the pip_vfb-integration pipeline is started, it will automatically trigger the pip_vfb-prod and pip_vfb-owlery pipelines to redeploy as well.  
1. The whole pipeline can be restarted by simply triggering the pip_vfb_kb pipeline to be re-run. This will trigger all downstream sub-pipelines.
1. The whole pipeline is re-run every night at 4am.
1. The pipeline is exposed at p2.virtualflybrain.org
