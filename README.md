# HOOT Specification

**HOOT** (Hierarchical Ontology-Optimized Tokens) is a compact RDF serialization format optimized for OWL ontology data.

Inspired by [TOON](https://github.com/toon-format/toon) (Token-Oriented Object Notation).

## Why HOOT?

Turtle is human-readable but verbose. HOOT encodes the same RDF graph in **~1/3 the characters** by exploiting recurring OWL patterns.

**Turtle (42 lines, ~1,900 chars):**

```turtle
@prefix ex: <http://example.org/> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .

ex:Person a owl:Class ; rdfs:label "Person" ; rdfs:subClassOf owl:Thing .
ex:Politician a owl:Class ; rdfs:label "Politician" ; rdfs:subClassOf ex:Person .
ex:Organization a owl:Class ; rdfs:label "Organization" ; rdfs:subClassOf owl:Thing .
ex:Company a owl:Class ; rdfs:label "Company" ; rdfs:subClassOf ex:Organization .

ex:partOf a owl:ObjectProperty, owl:TransitiveProperty ;
    rdfs:label "part of" ; owl:inverseOf ex:hasPart .

[] a owl:AllDisjointClasses ; owl:members ( ex:Person ex:Organization ) .
```

**HOOT Compact (~600 chars):**

```
@ex //example.org

class owl:Thing
 Person
  Politician
 Organization
  Company

->{iri,inverse,characteristics}:
 partOf,hasPart,transitive

! (Person Organization)
```

## Encoding Modes

| Mode | Purpose | Guarantee |
|------|---------|-----------|
| **Lossless** | Round-trip with Turtle | All triples preserved exactly |
| **Compact** | LLM prompt optimization | Derivable information omitted |

## Syntax Overview

```
@ex //example.org                       # Prefix: // = http://, trailing / or # auto-appended

class owl:Thing                          # Class hierarchy via indentation
 Person
  Politician

->{iri,inverse,characteristics}:         # Tabular property declarations
 partOf,hasPart,transitive

! (Person Organization),(Person Place)   # Disjoint class sets

ex:Toyota:                               # Subject block: general-purpose triples
 a ex:Company
 rdfs:label "Toyota"
 ex:startDate "1937-08-28"^^xsd:date
```

## Specification

See [SPEC.md](SPEC.md) for the full format specification.

## Implementations

| Language | Repository |
|----------|-----------|
| Swift | [hoot-format/swift-hoot](https://github.com/hoot-format/swift-hoot) |

## License

MIT
