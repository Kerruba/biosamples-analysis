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

The following cypher queries handle the contents of the CSV file produced by the file [collate-attributes.py](/blob/master/collate-attributes.py).

### Samples and attributes, with type and value properties

~~~~
USING PERIODIC COMMIT 1000 LOAD CSV WITH HEADERS FROM "file:///biosamples-annotations.csv" AS line WITH line WHERE line.ATTRIBUTE_TYPE IS NOT NULL MERGE (s:Sample {accession: line.ACCESSION}) MERGE (a:Attribute {type: line.ATTRIBUTE_TYPE, value: line.ATTRIBUTE_VALUE}) MERGE (s)-[:has_attribute]->(a)
~~~~

Note that this excludes rows where the attribute type is null to prevent errors; in theory this should not occur but may incidentally occur where a sample has zero attributes.  Also note that this query does not handle ontology terms; so far this will only populate sample to attribute nodes, and does not create nodes for ontology terms.  This is a useful future enhancement that might guide cleanup.

This data loading takes a long time - it might be better to optimise by splitting the input file into several smaller chunks instead of one single file of ~40M lines

### Samples, attributes, attribute types and values

~~~~
USING PERIODIC COMMIT 10000 LOAD CSV WITH HEADERS FROM "file:///biosamples-annotations-1.csv" AS line WITH line WHERE line.ATTRIBUTE_TYPE IS NOT NULL MERGE (s:Sample {accession: line.ACCESSION}) MERGE (a:Attribute {type: line.ATTRIBUTE_TYPE, value: line.ATTRIBUTE_VALUE}) MERGE (at:AttributType {name: line.ATTRIBUTE_TYPE}) MERGE (av:AttributeValue {name: line.ATTRIBUTE_VALUE}) MERGE (s)-[:has_attribute]->(a) MERGE (a)-[:has_type]->(at) MERGE (a)-[:has_value]->(av)
~~~~

Excludes rows with attribute type is null.

Still doesn't use ontology terms

Adds dedicated attribute type and value nodes

## Queries

### Get most frequently used attributes

The following query extracts the top 100 mostly commonly used attribute type/value pairs and counts their usage.

~~~~
MATCH (a:Attribute)<-[:has_attribute]-(s:Sample) WITH a, COUNT(s) AS usage_count RETURN a.type, a.value, usage_count ORDER BY usage_count DESC LIMIT 100
~~~~

### Get a list of attribute values where attribute type is "Disease State"

~~~~
MATCH (at:AttributeType {name: "Disease State"})<-[:has_type]-(a:Attribute)-[:has_value]->(av:AttributeValue) RETURN av.name
~~~~

### Get a list of attribute values where attribute type is "Disease State" with counts

~~~~
MATCH (at:AttributeType {name: "Disease State"})<-[:has_type]-(a:Attribute)-[:has_value]->(av:AttributeValue) WITH a, av MATCH (s:Sample)-[:has_attribute]->(a)-[:has_value]->(av) WITH av, COUNT(s) AS usage_count RETURN av.name, usage_count ORDER BY usage_count DESC
~~~~

### Get a list of attribute values which have more than one attribute type, sorted by most frequently used

~~~~
MATCH (av:AttributeValue)<-[:has_value]-(:Attribute)-[:has_type]->(at:AttributeType) WITH av, COUNT(at) AS num_of_types WHERE num_of_types > 1 MATCH (s:Sample)-[:has_attribute]->(a:Attribute)-[:has_value]->(av) WITH av, COUNT(s) AS usage_count RETURN av ORDER BY usage_count DESC LIMIT 10
~~~~