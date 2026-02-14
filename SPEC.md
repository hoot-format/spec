# HOOT Specification

**HOOT** (Hierarchical Ontology-Optimized Tokens) is a compact RDF serialization format optimized for OWL ontology data. It can represent any RDF graph and round-trips losslessly with Turtle.

Inspired by [TOON](https://github.com/toon-format/toon) (Token-Oriented Object Notation).

## Design Principles

1. **Tabular arrays** -- Field names declared once; values listed per row
2. **Minimal quoting** -- Strings unquoted unless ambiguous
3. **Indentation-based structure** -- Whitespace replaces delimiters
4. **Pattern compression** -- Common OWL patterns get compact syntax
5. **Full RDF coverage** -- Any RDF graph is representable

## Encoding Modes

HOOT defines two encoding modes:

| Mode | Purpose | Guarantee |
|------|---------|-----------|
| **Lossless** | Round-trip with Turtle | All triples preserved exactly |
| **Compact** | LLM prompt optimization | Derivable information omitted for minimum tokens |

### Compact Mode Omissions

The compact encoder MAY omit:

| Information | Condition for omission |
|-------------|----------------------|
| `rdfs:label` | Label equals the IRI local name (e.g., `ex:Person` -> `"Person"`) |
| Prefix on names | A declared prefix matches (e.g., `ex:Person` -> `Person`) |
| `a owl:Class` | Class appears in a `class` section |
| `a owl:ObjectProperty` | Property appears in an `ObjectProperty` tabular section |
| `a owl:DatatypeProperty` | Property appears in a `DataProperty` tabular section |
| Standard prefix declarations | `owl:`, `rdfs:`, `rdf:`, `xsd:` are implicitly available |
| `rdfs:domain owl:Thing` | Domain is the universal class |

The compact encoder MUST NOT omit:

- Class hierarchy structure (rdfs:subClassOf)
- Property inverse / characteristics / domain / range (except owl:Thing domain)
- Disjoint, EquivalentClass, Restriction axioms
- Labels that differ from the IRI local name
- All ABox assertions (individuals)

---

## 1. Primitives

### 1.1 IRIs

Prefixed form or full IRI in angle brackets.

```
ex:Person
<http://example.org/Person>
```

In compact mode, the prefix may be omitted when unambiguous:

```
Person
```

### 1.2 Literals

```
"plain string"
"typed"^^xsd:integer
"tagged"@en
42
3.14
true
false
```

- Unquoted integers and decimals are numeric literals
- `true` / `false` are boolean literals
- All other values must be quoted strings

### 1.3 Blank Nodes

Named blank node with document-scoped identity:

```
_:label
```

Inline anonymous blank node:

```
[
 rdf:type owl:Restriction
 owl:onProperty ex:hasChild
 owl:someValuesFrom owl:Thing
]
```

Properties of the blank node are indented inside the brackets.

### 1.4 Collections

RDF lists use parentheses:

```
(ex:Person ex:Organization ex:Place)
```

Expands to `rdf:first` / `rdf:rest` / `rdf:nil` chain.

---

## 2. Document Structure

A HOOT document consists of:

1. **Prefix declarations**
2. **Sections** (in any order, separated by blank lines):
   - `class` -- Class hierarchy (compact OWL pattern)
   - Tabular sections -- `ObjectProperty`, `DataProperty`, etc.
   - `disjoint` -- Disjoint class sets (compact OWL pattern)
   - Subject blocks -- General-purpose triples

---

## 3. Prefix Declaration

```
prefix <name>: <full-iri>
```

- Keyword `prefix` (lowercase, no `@`, no trailing `.`)
- Standard prefixes (`owl:`, `rdfs:`, `rdf:`, `xsd:`) are implicitly available
- In lossless mode, all prefixes used in the source Turtle are declared
- In compact mode, standard prefixes MAY be omitted

**Example:**
```
prefix ex: <http://example.org/>
```

---

## 4. Class Hierarchy

Compact syntax for the triple pattern:
```
?c a owl:Class ; rdfs:label ?label ; rdfs:subClassOf ?parent .
```

### Syntax

```
class <root>
 <iri>
 <iri> <label>
  <iri>
```

### Rules

- Keyword `class` followed by root class IRI
- One space per hierarchy level (depth 1 = 1 space, depth 2 = 2 spaces)
- Each line: class IRI, optionally followed by a quoted label
- A class at depth N is `rdfs:subClassOf` the nearest preceding class at depth N-1

### Label derivation

- **Lossless mode**: Always include the label (e.g., `ex:Person "Person"`)
- **Compact mode**: Omit label when it equals the IRI local name (e.g., `ex:Person`)

### Example (Lossless)

```
class owl:Thing
 ex:Person "Person"
  ex:Politician "Politician"
  ex:Ruler "Ruler"
 ex:Organization "Organization"
  ex:GovernmentAgency "Government Agency"
   ex:Municipality "Municipality"
```

### Example (Compact)

```
class owl:Thing
 Person
  Politician
  Ruler
 Organization
  GovernmentAgency "Government Agency"
   Municipality
```

Note: `GovernmentAgency` retains its label because `"Government Agency"` differs from the local name `"GovernmentAgency"`.

### Equivalent Turtle (per class)

```turtle
ex:Person a owl:Class ; rdfs:label "Person" ; rdfs:subClassOf owl:Thing .
ex:GovernmentAgency a owl:Class ; rdfs:label "Government Agency" ; rdfs:subClassOf ex:Organization .
```

---

## 5. Tabular Section

Compact syntax for uniform property declarations.

### Syntax

```
<SectionType>{<field1>,<field2>,...}:
 <value1>,<value2>,...
```

### Rules

- Header: `SectionType{field1,field2,...}:`
- Data rows: 1-space indent, comma-separated values
- First field is always the subject IRI
- Empty field: omitted between commas (`a,,c`)
- Trailing empty fields may be omitted

### 5.1 ObjectProperty

Fields and their predicate mapping:

| Field | Predicate |
|-------|-----------|
| `iri` | Subject IRI (gets `a owl:ObjectProperty`) |
| `label` | `rdfs:label` |
| `inverse` | `owl:inverseOf` |
| `characteristics` | Additional type assertion (see table below) |
| `domain` | `rdfs:domain` |
| `range` | `rdfs:range` |

Characteristic values:

| Value | Expands to |
|-------|-----------|
| `transitive` | `a owl:TransitiveProperty` |
| `symmetric` | `a owl:SymmetricProperty` |
| `asymmetric` | `a owl:AsymmetricProperty` |
| `reflexive` | `a owl:ReflexiveProperty` |
| `irreflexive` | `a owl:IrreflexiveProperty` |
| `functional` | `a owl:FunctionalProperty` |
| `inverseFunctional` | `a owl:InverseFunctionalProperty` |

**Example (Lossless):**
```
ObjectProperty{iri,label,inverse,characteristics,domain,range}:
 ex:partOf,part of,ex:hasPart,transitive,,
 ex:locatedIn,located in,,,ex:Place
 ex:hasFounder,has founder,ex:founderOf
 ex:hasParticipant,has participant,ex:participatesIn,,ex:Event,
 ex:causes,causes,ex:causedBy,,ex:Event,ex:Event
```

**Example (Compact):**
```
ObjectProperty{iri,inverse,characteristics,domain,range}:
 partOf,hasPart,transitive
 locatedIn,,,Place
 hasFounder,founderOf
 hasParticipant,participatesIn,,Event
 causes,causedBy,,Event,Event
```

In compact mode, the `label` field is omitted entirely from the header since all labels are derivable.

### 5.2 DataProperty

| Field | Predicate |
|-------|-----------|
| `iri` | Subject IRI (gets `a owl:DatatypeProperty`) |
| `label` | `rdfs:label` |
| `domain` | `rdfs:domain` |
| `range` | `rdfs:range` |

**Example (Lossless):**
```
DataProperty{iri,label,domain,range}:
 ex:occurredOnDate,occurred on date,ex:Occurrent,xsd:date
 ex:wikipediaURL,Wikipedia URL,,xsd:anyURI
```

### 5.3 AnnotationProperty

| Field | Predicate |
|-------|-----------|
| `iri` | Subject IRI (gets `a owl:AnnotationProperty`) |
| `label` | `rdfs:label` |

---

## 6. Disjoint Section

Compact syntax for `owl:AllDisjointClasses`.

### Syntax

```
disjoint (<class> <class> ...),(<class> <class> ...)
```

### Rules

- Keyword `disjoint` (lowercase)
- Each `(...)` group is a set of mutually disjoint classes
- Groups separated by commas

### Example

```
disjoint (ex:Person ex:Organization),(ex:Person ex:Place),(ex:Event ex:Era)
```

### Equivalent Turtle (per group)

```turtle
[] a owl:AllDisjointClasses ; owl:members ( ex:Person ex:Organization ) .
```

---

## 7. Subject Block

General-purpose triple representation for anything not covered by compact patterns.

### Syntax

```
<subject>:
 <predicate> <object>
 <predicate> <object>,<object>
```

### Rules

- Subject IRI (or blank node label) followed by colon
- Each predicate-object pair on an indented line (1 space)
- Multiple objects for the same predicate: comma-separated
- `a` is shorthand for `rdf:type`

### Inline Blank Nodes

```
ex:Parent:
 owl:equivalentClass [
  a owl:Restriction
  owl:onProperty ex:hasChild
  owl:someValuesFrom owl:Thing
 ]
```

- Opening `[` on the same line as the predicate
- Properties indented inside
- Closing `]` at the same indent level as opening

### Nested Blank Nodes

```
ex:WorkingParent:
 owl:equivalentClass [
  owl:intersectionOf (
   ex:Parent
   [
    a owl:Restriction
    owl:onProperty ex:worksAt
    owl:someValuesFrom ex:Organization
   ]
  )
 ]
```

### Collections in Subject Blocks

```
_:disj1:
 a owl:AllDisjointClasses
 owl:members (ex:Person ex:Organization)
```

---

## 8. Quoting Rules

### In Tabular Rows

Values are unquoted by default. Quote with `"..."` when the value:

- Contains: `,` `"` `\` `(` `)` `[` `]` `{` `}`
- Has leading or trailing whitespace
- Is an empty string that must be preserved (distinct from omitted field)
- Matches a keyword: `true` `false` `null` `a` `class` `prefix` `disjoint`

### In Subject Blocks

Predicate and object values follow IRI / literal / blank node syntax (no additional quoting needed).

### Escape Sequences

| Sequence | Character |
|----------|-----------|
| `\\` | Backslash |
| `\"` | Double quote |
| `\n` | Newline |
| `\t` | Tab |

---

## 9. Comments

Lines starting with `#` (after optional whitespace) are comments.

```
# Ontology definition
class owl:Thing
 # Person subtree
 ex:Person "Person"
```

---

## 10. Complete Example (Lossless)

```
prefix ex: <http://example.org/>

class owl:Thing
 ex:Person "Person"
  ex:Politician "Politician"
  ex:Ruler "Ruler"
  ex:Scientist "Scientist"
 ex:Organization "Organization"
  ex:Company "Company"
  ex:GovernmentAgency "Government Agency"
   ex:Municipality "Municipality"
 ex:Place "Place"
  ex:Country "Country"
  ex:City "City"
 ex:Occurrent "Occurrent"
  ex:Event "Event"
  ex:Era "Era"

ObjectProperty{iri,label,inverse,characteristics,domain,range}:
 ex:partOf,part of,ex:hasPart,transitive,,
 ex:locatedIn,located in,,transitive,,ex:Place
 ex:hasFounder,has founder,ex:founderOf
 ex:memberOf,member of,ex:hasMember
 ex:produces,produces,ex:producedBy
 ex:hasParticipant,has participant,ex:participatesIn,,ex:Event,
 ex:causes,causes,ex:causedBy,,ex:Event,ex:Event

DataProperty{iri,label,domain,range}:
 ex:occurredOnDate,occurred on date,ex:Occurrent,xsd:date
 ex:startDate,start date,ex:Occurrent,xsd:date
 ex:endDate,end date,ex:Occurrent,xsd:date
 ex:wikipediaURL,Wikipedia URL,,xsd:anyURI
 ex:officialURL,official URL,,xsd:anyURI

disjoint (ex:Person ex:Organization),(ex:Person ex:Place),(ex:Occurrent ex:Person),(ex:Occurrent ex:Organization),(ex:Occurrent ex:Place),(ex:Product ex:Service),(ex:Event ex:Era)
```

## 11. Complete Example (Compact)

```
prefix ex: <http://example.org/>

class owl:Thing
 Person
  Politician
  Ruler
  Scientist
 Organization
  Company
  GovernmentAgency "Government Agency"
   Municipality
 Place
  Country
  City
 Occurrent
  Event
  Era

ObjectProperty{iri,inverse,characteristics,domain,range}:
 partOf,hasPart,transitive
 locatedIn,,transitive,,Place
 hasFounder,founderOf
 memberOf,hasMember
 produces,producedBy
 hasParticipant,participatesIn,,Event
 causes,causedBy,,Event,Event

DataProperty{iri,domain,range}:
 occurredOnDate,Occurrent,xsd:date
 startDate,Occurrent,xsd:date
 endDate,Occurrent,xsd:date
 wikipediaURL,,xsd:anyURI
 officialURL,,xsd:anyURI

disjoint (Person Organization),(Person Place),(Occurrent Person),(Occurrent Organization),(Occurrent Place),(Product Service),(Event Era)
```

---

## Round-trip Guarantee

### Turtle -> HOOT (Lossless)

1. Parse Turtle into RDF triples
2. Identify compressible patterns:
   - Classes with `a owl:Class` + `rdfs:label` + `rdfs:subClassOf` -> `class` section
   - Properties with uniform predicates -> tabular section
   - `owl:AllDisjointClasses` -> `disjoint` section
3. Remaining triples -> subject blocks
4. All information preserved

### HOOT (Lossless) -> Turtle

1. Expand `class` section: each entry -> `a owl:Class`, `rdfs:label`, `rdfs:subClassOf` triples
2. Expand tabular rows: each row -> property declaration triples
3. Expand `disjoint` section: each group -> `owl:AllDisjointClasses` triples
4. Emit subject block triples directly
5. Produce valid Turtle with prefix declarations

**Guarantee**: The RDF graphs are identical (same set of triples, modulo blank node renaming and triple ordering).

### Turtle -> HOOT (Compact) -> Turtle

Not guaranteed to produce identical Turtle. The compact encoder omits derivable information. However, a compact-mode decoder reconstructs the omitted triples using derivation rules:

- Missing label -> derive from IRI local name
- Missing prefix -> resolve using declared prefixes
- Missing `a owl:Class` -> infer from `class` section membership

The reconstructed RDF graph is semantically equivalent to the original.
