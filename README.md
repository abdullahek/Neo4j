# 🌐 Neo4j — RDF Sample Dataset

> **A minimal RDF / Linked-Data sample repository containing N-Triples about Erfurt municipal services, intended for experimentation with importing RDF graphs into Neo4j via the [neosemantics (`n10s`)](https://neo4j.com/labs/neosemantics/) plugin.**

[![Neo4j](https://img.shields.io/badge/Neo4j-Graph_DB-008CC1?logo=neo4j&logoColor=white)](https://neo4j.com/)
[![neosemantics](https://img.shields.io/badge/neosemantics-n10s-018BFF)](https://neo4j.com/labs/neosemantics/)
[![RDF](https://img.shields.io/badge/RDF-N--Triples-005A9C)](https://www.w3.org/TR/n-triples/)
[![SPARQL](https://img.shields.io/badge/SPARQL-1.1-E10098)](https://www.w3.org/TR/sparql11-overview/)
[![Linked Data](https://img.shields.io/badge/Linked_Data-W3C-FF6F00)](https://www.w3.org/standards/semanticweb/data)

---

## 🌟 Overview

This repository holds a tiny **RDF sample dataset** — a single N-Triples file (`test3.nt`) containing two statements about a municipal service offered by the city of **Erfurt, Germany** (specifically the *Abfallberatung Allgemein* — "General Waste Advisory" — service).

The dataset is intentionally minimal and is meant to be used as a **smoke-test fixture** when:

- Setting up a fresh **Neo4j** instance with the **[neosemantics (`n10s`)](https://neo4j.com/labs/neosemantics/)** plugin
- Verifying RDF → property-graph imports via `n10s.rdf.import.fetch` / `n10s.rdf.import.inline`
- Practising SPARQL-to-Cypher translations
- Demonstrating the `thurai/ontology#Service` ontology pattern from the [Thuringia Linked-Data project](https://www.purl.org/domain/thurai/ontology)

---

## 📦 Repository Contents

```
Neo4j/
├── test3.nt        # 2-line N-Triples sample (Erfurt waste-advisory service)
└── README.md
```

That's the entire repo. There is no application code, no database dump, and no Neo4j configuration shipped here — just the data fixture.

---

## 🗂️ The Dataset (`test3.nt`)

The file uses the **N-Triples** serialization (one `subject predicate object .` triple per line):

```turtle
<https://www.erfurt.de/ef/de/rathaus/bservice/leistungen/leistung-1081.htmc>
    <http://www.w3.org/2000/01/rdf-schema#label>
    "Abfallberatung Allgemein"@de .

<https://www.erfurt.de/ef/de/rathaus/bservice/leistungen/leistung-1081.htmc>
    <http://www.w3.org/1999/02/22-rdf-syntax-ns#type>
    <http://purl.org/domain/thurai/ontology#Service> .
```

### What it says

| # | Subject | Predicate | Object |
|---|---------|-----------|--------|
| 1 | The Erfurt service URL `…leistung-1081.htmc` | `rdfs:label` | `"Abfallberatung Allgemein"` (German) |
| 2 | The same service URL | `rdf:type` | `thurai:Service` |

In plain English: *the resource at the given erfurt.de URL is a `Service` (per the Thurai ontology), and its human-readable label in German is "Abfallberatung Allgemein".*

### Vocabularies used

| Prefix | URI | Used for |
|--------|-----|----------|
| `rdf` | `http://www.w3.org/1999/02/22-rdf-syntax-ns#` | `rdf:type` |
| `rdfs` | `http://www.w3.org/2000/01/rdf-schema#` | `rdfs:label` |
| `thurai` | `http://purl.org/domain/thurai/ontology#` | `Service` class |

---

## 🚀 Loading the Data into Neo4j

The intended use-case for this repo is to import the triples into Neo4j with the **neosemantics** plugin and explore them as a property graph.

### 1. Install neosemantics (`n10s`)
Download the plugin JAR matching your Neo4j version from the [official releases page](https://github.com/neo4j-labs/neosemantics/releases) and drop it into your Neo4j `plugins/` directory.

Add the following to `neo4j.conf`:

```properties
dbms.unmanaged_extension_classes=n10s.endpoint=/rdf
dbms.security.procedures.unrestricted=n10s.*
dbms.security.procedures.allowlist=n10s.*
```

Restart Neo4j.

### 2. Initialise the n10s graph configuration

```cypher
CREATE CONSTRAINT n10s_unique_uri
FOR (r:Resource) REQUIRE r.uri IS UNIQUE;

CALL n10s.graphconfig.init({
  handleVocabUris: 'MAP',
  handleMultival: 'ARRAY',
  keepLangTag:    true
});
```

### 3. Import `test3.nt`

**Option A — fetch from a public URL** (e.g. once you publish the file on GitHub Raw):

```cypher
CALL n10s.rdf.import.fetch(
  'https://raw.githubusercontent.com/abdullahek/Neo4j/main/test3.nt',
  'N-Triples'
);
```

**Option B — inline import** (paste the file content):

```cypher
CALL n10s.rdf.import.inline(
'<https://www.erfurt.de/ef/de/rathaus/bservice/leistungen/leistung-1081.htmc> <http://www.w3.org/2000/01/rdf-schema#label> "Abfallberatung Allgemein"@de .
<https://www.erfurt.de/ef/de/rathaus/bservice/leistungen/leistung-1081.htmc> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://purl.org/domain/thurai/ontology#Service> .',
  'N-Triples'
);
```

### 4. Verify the import

```cypher
MATCH (n:Resource)
RETURN n.uri AS uri, labels(n) AS labels, n.rdfs__label AS label;
```

Expected result:

```
┌──────────────────────────────────────────────────────────────────────┬──────────────────────┬──────────────────────────────┐
│ uri                                                                  │ labels               │ label                        │
├──────────────────────────────────────────────────────────────────────┼──────────────────────┼──────────────────────────────┤
│ https://www.erfurt.de/.../leistung-1081.htmc                         │ [Resource, Service]  │ ["Abfallberatung Allgemein"] │
└──────────────────────────────────────────────────────────────────────┴──────────────────────┴──────────────────────────────┘
```

---

## 🔍 Querying with Cypher

A few example queries against the imported graph:

```cypher
// All resources typed as a Service
MATCH (s:Service)
RETURN s.uri, s.rdfs__label;

// Count of triples (n10s materialises them as nodes/properties, not as edges by default)
MATCH (n:Resource) RETURN count(n) AS resources;

// Round-trip: export the imported subgraph back to RDF
CALL n10s.rdf.export.cypher(
  'MATCH (n:Resource) RETURN n'
);
```

---

## 🌐 SPARQL Equivalent

If you instead loaded `test3.nt` into a SPARQL endpoint (e.g. **Apache Jena Fuseki**, **Blazegraph**, **GraphDB**), the same data answers:

```sparql
PREFIX rdf:    <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs:   <http://www.w3.org/2000/01/rdf-schema#>
PREFIX thurai: <http://purl.org/domain/thurai/ontology#>

SELECT ?service ?label
WHERE {
  ?service a thurai:Service ;
           rdfs:label ?label .
}
```

---

## 🧰 Useful Tooling

| Tool | What for |
|------|----------|
| [Neo4j Desktop](https://neo4j.com/download/) | Local Neo4j with plugin manager |
| [neosemantics (`n10s`)](https://neo4j.com/labs/neosemantics/) | RDF ↔ Neo4j bridge |
| [Apache Jena `riot`](https://jena.apache.org/documentation/io/) | Validate / convert N-Triples ↔ Turtle ↔ JSON-LD |
| [rapper](https://librdf.org/raptor/rapper.html) | Quick CLI RDF parser/validator |
| [GraphDB](https://www.ontotext.com/products/graphdb/) | Full-featured triple store with SPARQL |

### Validate the file locally

```bash
# With Apache Jena's riot
riot --validate test3.nt

# With raptor
rapper -i ntriples -c test3.nt

# With Python rdflib
python -c "from rdflib import Graph; g = Graph().parse('test3.nt', format='nt'); print(len(g), 'triples')"
```

---

## 🧪 Extending the Dataset

The `test3.nt` file is a deliberate stub. If you want to grow it, follow the same N-Triples pattern (`subject predicate object .`) and consider adding:

- `dcterms:description` for service descriptions
- `schema:address` for office addresses
- `thurai:hasCategory` to group related services
- `owl:sameAs` links to DBpedia / Wikidata equivalents

Then re-run `n10s.rdf.import.fetch` to top up the graph.

---

## 📄 License

This repository hosts public-domain administrative data references (`erfurt.de` URIs) and the [Thurai ontology](http://purl.org/domain/thurai/ontology#) namespace. The repo itself is provided for educational / experimentation purposes.

---

## 👤 Author / Maintainer

**Abdullah EK** — [@abdullahek](https://github.com/abdullahek)

---

<p align="center">
  A pocket-sized RDF fixture for kicking the tyres on Neo4j + neosemantics
</p>
