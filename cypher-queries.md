# Cypher queries for BioSamples Attribute Analysis

## Set up indexes

Initialise indexes over nodes and properties we're going to use

### Samples

~~~~
CREATE CONSTRAINT ON (s:Sample) ASSERT s.accession IS UNIQUE
~~~~

### Attributes

~~~~
CREATE INDEX on :Attribute(type)
CREATE INDEX on :Attribute(value)
~~~~

### Ontology Terms

~~~~
CREATE CONSTRAINT ON (ot:OntologyTerm) ASSERT ot.iri IS UNIQUE
~~~~

## Data loading

The following cypher query handles the contents of the CSV file produced by the file [collate-attributes.py](/blob/master/collate-attributes.py).

~~~~
USING PERIODIC COMMIT 1000 LOAD CSV WITH HEADERS FROM "file:///biosamples-annotations.csv" AS line WITH line WHERE line.ATTRIBUTE_TYPE IS NOT NULL MERGE (s:Sample {accession: line.ACCESSION}) MERGE (a:Attribute {type: line.ATTRIBUTE_TYPE, value: line.ATTRIBUTE_VALUE}) MERGE (s)-[:has_attribute]->(a)
~~~~

Note that this excludes rows where the attribute type is null to prevent errors; in theory this should not occur but may incidentally occur where a sample has zero attributes.  Also note that this query does not handle ontology terms; so far this will only populate sample to attribute nodes, and does not create nodes for ontology terms.  This is a useful future enhancement that might guide cleanup.

This data loading takes a long time - it might be better to optimise by splitting the input file into several smaller chunks instead of one single file of ~40M lines

## Get most frequently used attributes

The following query extracts the top 100 mostly commonly used attribute type/value pairs and counts their usage.

~~~~
MATCH (a:Attribute)<-[:has_attribute]-(s:Sample) WITH a, COUNT(s) AS usage_count RETURN a.type, a.value, usage_count ORDER BY usage_count DESC LIMIT 100
~~~~