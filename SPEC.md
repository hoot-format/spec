# HOOT Specification v1.0

**HOOT** (Hierarchical Ontology-Optimized Tokens) is a compact, human-readable serialization format for OWL ontology data, designed for reduced token usage when passing structured ontology information to Large Language Models.

Inspired by [TOON](https://github.com/toon-format/toon) (Token-Oriented Object Notation).

## Design Principles

1. **Tabular arrays** -- Field names declared once; values listed per row
2. **Minimal quoting** -- Strings unquoted unless ambiguous
3. **Indentation-based hierarchy** -- Whitespace replaces verbose markup

## Document Structure

A HOOT document consists of sections separated by blank lines, in this order:

1. **Prefix declarations** (optional)
2. **Class hierarchy** (indentation tree)
3. **Tabular sections** (properties, etc.)
4. **Inline sections** (disjoints, etc.)

## 1. Prefix Declaration

```
@prefix <short>: <full-iri>
```

Declares a namespace prefix. After declaration, IRIs using that prefix may omit it in class and property names.

**Example:**
```
@prefix ex: <http://example.org/>
```

After this declaration, `Person` is equivalent to `ex:Person`.

**Rules:**
- Zero or more `@prefix` lines at the start of the document
- Each line declares exactly one prefix
- Prefix names follow the pattern `[a-zA-Z][a-zA-Z0-9]*`
- Full IRIs are enclosed in angle brackets

## 2. Class Hierarchy

```
<root-class>
 <child-class>
  <grandchild-class>
 <another-child>
```

Represents `rdfs:subClassOf` relationships via indentation.

**Rules:**
- One space per hierarchy level (depth 0 = root, depth 1 = 1 space, depth 2 = 2 spaces, ...)
- Root class is typically `owl:Thing`
- Class names use the short form (prefix omitted) when a matching `@prefix` is declared
- Each line contains exactly one class name
- A class at depth N is a subclass of the nearest preceding class at depth N-1

**Example:**
```
owl:Thing
 Person
  Politician
  Ruler
  Scientist
 Organization
  Company
  GovernmentAgency
   Municipality
 Place
  Country
  City
```

This encodes:
- `Person rdfs:subClassOf owl:Thing`
- `Politician rdfs:subClassOf Person`
- `Municipality rdfs:subClassOf GovernmentAgency`
- etc.

## 3. Tabular Section

```
<SectionName>{<field1>,<field2>,...}:
 <value1>,<value2>,...
 <value3>,<value4>,...
```

Declares a table of uniform records. The header specifies the section name and field names; each subsequent indented line is a data row.

**Rules:**
- Header format: `Name{field1,field2,...}:` (no spaces around commas in header)
- Data rows: 1-space indent, values separated by commas
- Empty field: omitted between commas (e.g., `a,,c` means field2 is empty)
- Trailing empty fields may be omitted (e.g., `a,b` is valid when 3 fields are declared -- field3 is empty)
- Field count per row must not exceed the declared field count

**Example:**
```
ObjectProperty{iri,inverse,characteristics}:
 partOf,hasPart,transitive
 locatedIn,,transitive
 hasFounder,founderOf
 memberOf,hasMember
 produces,producedBy
```

This encodes 5 object properties. `locatedIn` has no inverse (empty field). `hasFounder` and below have no characteristics (trailing field omitted).

## 4. Inline Section

```
<SectionName>: <value>
```

A single-line section for compact data.

**Set notation:**
```
Disjoint: {A,B},{C,D},{E,F}
```

Each `{...}` group represents a set of mutually exclusive classes.

**Rules:**
- Section name followed by colon and space
- Set groups use `{member1,member2,...}` notation
- Groups separated by commas (no spaces)
- Member names use short form (prefix omitted)

## 5. Quoting Rules

Values are **unquoted by default**. A value MUST be quoted with double quotes (`"..."`) when it:

- Is an empty string
- Contains: comma (`,`), colon (`:`), double quote (`"`), backslash (`\`), braces (`{`, `}`), brackets (`[`, `]`)
- Has leading or trailing whitespace
- Looks like a reserved token: `true`, `false`, `null`

**Escape sequences** (inside quoted strings):

| Sequence | Character |
|----------|-----------|
| `\\` | Backslash |
| `\"` | Double quote |
| `\n` | Newline |
| `\t` | Tab |

## 6. Comments

Lines starting with `#` (after optional leading whitespace) are comments and are ignored by parsers.

```
# This is a comment
owl:Thing
 # Person hierarchy
 Person
  Politician
```

## Complete Example

```
@prefix ex: <http://example.org/>

owl:Thing
 Person
  Politician
  Ruler
  MilitaryLeader
  Executive
  Scientist
  Artist
  Athlete
  ReligiousLeader
 Organization
  Company
  GovernmentAgency
   Municipality
  AcademicInstitution
   University
   ResearchInstitute
 Place
  Country
  Region
  City

ObjectProperty{iri,inverse,characteristics}:
 partOf,hasPart,transitive
 locatedIn,,transitive
 hasFounder,founderOf
 memberOf,hasMember
 produces,producedBy
 hasParticipant,participatesIn
 causes,causedBy

DataProperty{iri,domain,range}:
 occurredOnDate,Occurrent,xsd:date
 startDate,Occurrent,xsd:date
 endDate,Occurrent,xsd:date
 wikipediaURL,,xsd:anyURI
 officialURL,,xsd:anyURI

Disjoint: {Person,Organization},{Person,Place},{Occurrent,Person},{Occurrent,Organization},{Occurrent,Place},{Product,Service}
```

## Token Efficiency

Compared to equivalent OWL Turtle serialization:

| Format | Approx. tokens (200 classes + 15 properties) |
|--------|----------------------------------------------|
| Turtle | ~700+ lines |
| HOOT | ~230 lines |
| **Reduction** | **~60-70%** |

The savings come from:
- No repeated `a owl:Class ; rdfs:label "..." ;` per class
- No prefix declarations repeated per triple
- No ` .` statement terminators
- Tabular properties vs. multi-line property blocks
