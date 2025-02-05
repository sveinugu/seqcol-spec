---
title: Seqcol specification version 0.1.0
---

<!-- Table of contents: 
* The generated Toc will be an unordered list
{:toc} -->

<h1>Seqcol specification version 0.1.0</h1>

Table of contents:

[TOC]

## Specification version

This specification is in **DRAFT** form. This is **NOT YET AN APPROVED GA4GH specification**.

## Introduction

Reference sequences are fundamental to genomic analysis.
To make their analysis reproducible and efficient, we require tools that can identify, store, retrieve, and compare reference sequences.
The primary goal of the *Sequence Collections* (seqcol) project is **to standardize identifiers for collections of sequences**.
Seqcol can be used to identify genomes, transcriptomes, or proteomes -- anything that can be represented as a collection of sequences.
A common example and primary use case of sequence collections is for reference genome, so this documentation sometimes refers to reference genomes for convenience; really, it can be applied to any collection of sequences.

In brief, the project specifies several procedures:

1. **An algorithm for encoding sequence collection identifiers.**  The GA4GH standard [refget](http://samtools.github.io/hts-specs/refget.html) specifies a way to compute deterministic sequence identifiers from individual sequences. Seqcol uses refget identifiers and adds functionality to wrap them into collections of sequences. Seqcol also handles sequence attributes, such as their names, lengths, or topologies. Seqcol identifiers are defined by a hash algorithm, rather than an accession authority, and are thus decentralized and usable for private sequence collections, cases without connection to a central database, or validation of sequence collection content and provenance.
2. **An API describing lookup and comparison of sequence collections.** Seqcol specifies a RESTful API to retrieve the sequence collection given an identifier. A main use case is to reproduce the exact sequence collection (*e.g.* reference genome) used for analysis, instead of guessing based on a human-readable identifier. Seqcol also provides a standardized method of comparing the contents of two sequence collections. This comparison function can *e.g.* be used to determine if analysis results based on different references genomes are compatible. 
3. **Recommended ancillary, non-inherent attributes.** Finally, the protocol defines several recommended procedures that will improve the compatibility across Seqcol servers, and beyond.

## Use cases

Sequence collections represents fundamental concepts; therefore the specification can be used for many downstream use cases.
For example, we envision that seqcol identifiers could replace or live alongside the human-readable identifiers currently used to identify reference genomes (*e.g.* "hg38" or "GRCh38"). This would provide improved reproducibility.
Some other examples of common use cases where the use of seqcol is beneficial include:

1. Given a collection identifier, retrieve the list of refget sequence identifiers for the contained sequences.
2. Given a collection identifier, retrieve the contained sequences.
3. Given two collection identifiers, determine if downstream results are compatible.
4. Given a collection identifier, retrieve metadata about the collection. This may include human-readable aliases, author of the collection, links to other collections, or other metadata.
5. Given a sequence collection, compute its identifier.



## Definitions of key terms

- **Alias**: A human-readable identifier used to refer to a sequence collection.
- **Array**: An ordered list of elements.
- **Collated**: A qualifier applied to a seqcol attribute indicating that the values of the attribute matches 1-to-1 with the sequences in the collection and are represented in the same order.
- **Coordinate system**: An ordered list of named sequence lengths, but without actual sequences.
- **Digest**: An identifier resulting from a cryptographic hash function, such as `MD5` or `SHA512`, on input data.
- **Inherent**: A qualifier applied to a seqcol attribute indicating that the attribute is part of the definition of the sequence collection and therefore contributes to its digest.
- **Length**: The number of characters in a sequence.
- **Seqcol algorithm**: The set of instructions used to compute a digest from a sequence collection.
- **Seqcol API**: The set of endpoints defined in the *retrieval* and *comparison* components of the seqcol protocol.
- **Seqcol digest**: A digest for a sequence collection, computed according to the seqcol algorithm.
- **Seqcol protocol**: Collectively, the 3 operations outlined in this document, which include: 1. encoding of sequence collections; 2. API describing retrieval and comparison ; and 3. specifications for ancillary recommended attributes.
- **Sequence**: Seqcol uses refget to store actual sequences, so we generally use the term in the same way as refget. Refget was designed for nucleotide sequences; however, other sequences could be provided via the same mechanism, *e.g.*, cDNA, CDS, mRNA or proteins. Essentially any ordered list of refget-valid characters qualifies. Sequence collections also goes further, since sequence collections may contain sequences of non-specified characters, which therefore have a length but no actual sequence content.
- **Sequence digest** or **refget digest**: A digest for a sequence, computed according to the refget protocol.
- **Sequence collection**: A representation of 1 or more sequences that is structured according to the sequence collection schema
- **Sequence collection attribute**: A property or feature of a sequence collection (*e.g.* names, lengths, sequences, or topologies).

## Seqcol protocol functionality

The seqcol algorithm is based on the refget algorithm for individual sequences and should use refget servers to store the actual sequence data.
Seqcol servers therefore provide a lightweight organizational layer on top of refget servers.
To be fully compliant with the seqcol protocol an implementation must provide all `REQUIRED` capabilities as detailed below.

The seqcol protocol defines the following:

1. *Encoding* - An algorithm for computing an identifier given a collection of sequences.
2. *API* - A server RESTful API specification for retrieving and comparing sequence collections.
3. *Ancillary attribute management* - An optional specification for organizing non-inherent metadata as part of a sequence collection.

### 1. Encoding: Computing sequence digests from sequence collections

The encoding function specifies an algorithm that takes as input a set of annotated sequences and produces a unique digest. This function is generally expected to be provided by local software that operates on a local set of sequences. These steps of the encoding process are:

- **Step 1**. Organize the sequence collection data into *canonical seqcol object representation* and filter the non-inherent attributes.
- **Step 2**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) (JCS) to canonicalize the value associated with each attribute individually.
- **Step 3**. Digest each canonicalized attribute value using the GA4GH digest algorithm.
- **Step 4**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) again to canonicalize the JSON of the new seqcol object representation.
- **Step 5**. Digest the final canonical representation again using the GA4GH digest algorithm.

Example Python code for computing a seqcol digest can be found in the [tutorial for computing seqcol digests](digest_from_collection.md). These steps are described in more detail below:

#### Step 1: Organize the sequence collection data into *canonical seqcol object representation*.

We first create an object representation of the attributes of the sequence collection.
The structure of this object is critical, and is strictly controlled by the seqcol protocol.
It must be defined in a JSON Schema.
This is the general, minimal schema:


```YAML
description: "A collection of biological sequences."
type: object
properties:
  lengths:
    type: array
    collated: true
    description: "Number of elements, such as nucleotides or amino acids, in each sequence."
    items:
      type: integer
  names:
    type: array
    collated: true
    description: "Human-readable identifiers of each sequence (chromosome names)."
    items:
      type: string
  sequences:
    type: array
    collated: true
    items:
      type: string
      description: "Refget v2 identifiers of sequences."
required:
  - names
  - lengths
inherent:
  - lengths
  - names
  - sequences
```

This schema is the *seqcol schema*, and sequence collection objects in this structure are said to be the *canonical seqcol object representation*.
Here's an example of a sequence collection organized into the canonical seqcol object representation:

```json
{
  "lengths": [
    248956422,
    133797422,
    135086622
  ],
  "names": [
    "chr1",
    "chr2",
    "chr3"
  ],
  "sequences": [
    "SQ.2648ae1bacce4ec4b6cf337dcae37816",
    "SQ.907112d17fcb73bcab1ed1c72b97ce68",
    "SQ.1511375dc2dd1b633af8cf439ae90cec"
  ]
}
```

This object would validate against the JSON Schema above.
The object is a series of arrays with matching length (`3`), with the corresponding entries collated such that the first element of each array corresponds to the first element of each other array.
For the rationale why this structure was chosen instead of an array of annotated sequences, see [*Footnote F1*](#f1-why-use-an-array-oriented-structure-instead-of-a-sequence-oriented-structure).
Implementations `MUST` provide at least the structure specified in this schema.
Implementations `MAY` choose to extend this schema by adding additional attributes.
This schema extends vanilla JSON Schema in two ways; first, it provides the `collated` qualifier.
For further details about the rationale behind collated attributes, see [*Footnote F2*](#f2-collated-attributes).
Second, it specifies the `inherent` qualifier. For further details about the rationale and examples of non-inherent attributes, see [*Footnote F3*](#f3-details-of-inherent-and-non-inherent-attributes).
Finally, another detail that may be unintuitive at first is that the `sequences` attribute is optional; for an explanation of why, see [*Footnote F4*](#f4-sequence-collections-without-sequences).

##### Filter non-inherent attributes

The `inherent` section in the seqcol schema is an extension of the basic JSON Schema format that adds specific functionality.
Inherent attributes are those that contribute to the identifier; *non-inherent* attributes are not considered when computing the top-level digest.
Attributes of a seqcol that are *not* listed as `inherent` `MUST NOT` contribute to the identifier; they are therefore excluded from the digest calculation.
Therefore, if the canonical seqcol representation includes any non-inherent attributes, these must be removed before proceeding to step 2.
In the simple example, there are no non-inherent attributes.

#### Step 2: Apply RFC-8785 to canonicalize the value associated with each attribute individually.

The [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) (JCS) standardizes whitespace, character encodings, and other details that would cause inconsequential variations to yield different digests.
For most use cases*, the following Python function suffices:

```python
import json

def canonical_str(item: [list, dict]) -> bytes:
    """Convert a list or dict into a canonicalized UTF8-encoded bytestring representation"""
    return json.dumps(
        item, separators=(",", ":"), ensure_ascii=False, allow_nan=False, sort_keys=True
    ).encode('utf8')
```

This will turn the values into canonicalized UTF8-encoded bytestring representations of the list objects. Using Python notation, the value of the lengths attribute becomes `b'[248956422,133797422,135086622]'`, the value of the names attribute becomes `b'["chr1","chr2","chr3"]'`, and the value of the sequences attribute becomes 

```
b'["SQ.2648ae1bacce4ec4b6cf337dcae37816","SQ.907112d17fcb73bcab1ed1c72b97ce68","SQ.1511375dc2dd1b633af8cf439ae90cec"]'
```

_* The above Python function suffices if (1) attribute keys are restricted to ASCII, (2) there are no floating point values, and (3) for all integer values `i`:  `-2**63 < i < 2**63`_

#### Step 3: Digest each canonicalized attribute value using the GA4GH digest algorithm.

Apply the GA4GH digest algorithm to each attribute value.
The GA4GH digest algorithm is described in detail in [*Footnote F5*](#f5-the-ga4gh-digest-algorithm).
This converts the value of each attribute in the seqcol into a digest string.
Applying this to each value will produce a structure that looks like this:

```json
{
  "lengths": "20e95aade8e72d399dbf7f82a9e84ba5cc4047dc8d791d62",
  "names": "834e2529dc6262d1b774e19e502e4074a1227f0eb91b45a9",
  "sequences": "78f45f5aa3b36a2a8fe1eec415258a036b3753f69acf05df"
}
```

#### Step 4: Apply RFC-8785 again to canonicalize the JSON of the new seqcol object representation.

Here, we repeat step 2, except instead of applying RFC-8785 to each value separately, we apply to the entire object.
This will result in a canonical bytestring representation of the object, shown here using Python notation:

```
b'{"lengths":"20e95aade8e72d399dbf7f82a9e84ba5cc4047dc8d791d62","names":"834e2529dc6262d1b774e19e502e4074a1227f0eb91b45a9","sequences":"78f45f5aa3b36a2a8fe1eec415258a036b3753f69acf05df"}'
```

#### Step 5: Digest the final canonical representation again using the GA4GH digest algorithm.

Again using the same approach as in step 3, we now apply the GA4GH digest algorithm to the canonicalized bytestring.
The result is the final unique identifier for this sequence collection:

```
64ff00b85402a4dc821752e1e8d56d3ecc4e29b55a930748
```

---

### 2. API: A server RESTful API specification for retrieving and comparing sequence collections.

The API has 3 top-level endpoints, for 3 functions:

1. `/service-info`, for describing information about the service;
2. `/collection`, for retrieving sequence collections; and
3. `/comparison`, for comparing two sequence collections.

In addition, a RECOMMENDED endpoint at `/openapi.json` SHOULD provide OpenAPI documentation.

Under these umbrella endpoints are a few more specific sub-endpoints, described in detail below:

#### 2.1 Service info 
- *Endpoint*: `GET /service-info` (`REQUIRED`)
- *Description*: The service info endpoint provides information about the service
- *Return value*: Must include the Seqcol JSON Schema that is used by this server

The `/service-info` endpoint should follow the [GA4GH-wide specification for service info](https://github.com/ga4gh-discovery/ga4gh-service-info/) for general description of the service.
Then, it should also add a few specific pieces of information under a `seqcol` property:
 - `schema`: MUST return the JSON Schema implemented by the server.

##### The service-info JSON-schema document

The `schema` attribute of `service-info` return value MUST provide a single schema, this MUST include all possible attributes your service may provide. In other words, you cannot have a collection with an attribute that is not defined in your schema.

We RECOMMEND the schema only define terms actually used in at least one collection served; however, it is allowed for the schema to contain extra terms that are not used in any collections in the server.

We RECOMMEND your schema use property-level refs to point to terms defined by a central, approved seqcol schema. However, it is also allowed for the schema to embed all definitions locally. The central, approved seqcol schema will be made available as the spec is finalized.

For example, here's a JSON schema that uses a `ref` to reference the approved seqcol schema:

```yaml
description: "A collection of biological sequences."
type: object
"$id": "https://example.com/sequence_collection",  # URI to main seqcol definition
properties:
  lengths:
    "$ref": "/lengths"
  names:
    "$ref": "/names"
  sequences:
    "$ref": "/sequences"
required:
  - names
  - lengths
inherent:
  - lengths
  - names
  - sequences
```

#### 2.2 Collection

- *Endpoint*: `GET /collection/:digest?level=:level` (`REQUIRED`)
- *Description*: The retrieval function specifies an API endpoint that retrieves original sequences from a database keyed by the unique digest. Here `:digest` is the seqcol digest computed above.  The level corresponds to the "expansion level" of the returned sequence collection returned. The default is `?level=2`, which returns the canonical structure.
- *Return value*: The sequence collection identified by the `:digest` variable. The structure of the data `MUST` be modulated by the `:level` query parameter.  Specifying `?level=2` returns the canonical structure, and `?level=1` returns the collection with digested attributes.

Non-inherent attributes `MUST` be stored and returned by the collection endpoint alongside inherent attributes.

#### 2.3 Comparison

- *Endpoint variant 1*: Two-digest comparison `GET /comparison/{digest1}/{digest2}` (`REQUIRED`)  
- *Endpoint variant 2*: POST comparison with one digest  `POST /comparison/{digest1}` (`REQUIRED`)  
- *Description*: The comparison function specifies an API endpoint that allows a user to compare two sequence collections. The `POST` version compares one database collection to a local user-provided collection.  
- *Return value*: The output is an assessment of compatibility between those sequence collections. Both variants of the `/comparison` endpoint must `MUST` return an object in JSON format with these 3 keys: "digests", "arrays", and "elements", as described below:
    - `digests`: an object with 2 elements, with keys *a* and *b*, and values either the level 0 seqcol digests for the compared collections, or *null* (undefined). The value MUST be the level 0 seqcol digest for any digests provided by the user for the comparison. However, it is OPTIONAL for the server to provide digests if the user provided the sequence collection contents, rather than a digest. In this case, the server MAY compute and return the level 0 seqcol digest, or it MAY return *null* (undefined) in this element for any corresponding sequence collection.
    - `attributes`: an object with 3 elements, with keys *a_only*, *b_only*, and *a_and_b*. The value of each element is a list of array names corresponding to arrays only present in a, only present in b, or present in both a and b.
    - `array_elements`: An object with 4 elements: *a_count*, *b_count*, *a_and_b_count*, and *a_and_b_same_order*. The 3 attributes with *_count* are objects with names corresponding to each array present in the collection, or in both  collections (for *a_and_b_count*), with values as the number of elements present either in one collection, or in both collections for the given array. *a_and_b_same_order* is also an object with names corresponding to arrays, and the values a boolean following the same-order specification below.


Example `/comparison` return value: 
```
{
  "digests": {
    "a": "514c871928a74885ce981faa61ccbb1a",
    "b": "c345e091cce0b1df78bfc124b03fba1c"
  },
  "attributes": {
    "a_only": [],
    "b_only": [],
    "a_and_b": [
      "lengths",
      "names",
      "sequences"
    ]
  },
  "array_elements": {
    "a_count": {
      "lengths": 195,
      "names": 195,
      "sequences: 195
    },
    "b_count": {
      "lengths": 25,
      "names": 25,
      "sequences: 25
    }
    "a_and_b_count": {
      "lengths": 25,
      "names": 25,
      "sequences": 0
    },
    "a_and_b_same_order": {
      "lengths": false,
      "names": false,
      "sequences": null
    }
  }
}
```

##### Same-order specification

The comparison return includes an *a_and_b_same_order* boolean value for each array that is present in both collections. The defined value of this attribute is:

- undefined (*null*) if there are fewer than 2 overlapping elements
- undefined (*null*) if there are unbalanced duplicates present (see definition below)
- *true* if all matching elements are in the same order in the two arrays
- *false* otherwise.

An *unbalanced duplicate* is used in contrast with a *balanced duplicate*. Balanced means the duplicates are the same in both arrays.
When the duplicates are balanced, order is still defined; but if duplicates are unbalanced, this means an array has duplicates not present in the other, and in that case, order is not defined.

##### Interpreting the result of the compare function

The output of the comparison function provides information-rich feedback about the two collections.
These details can be used to make a variety of inferences comparing two collections, but it can take some thought to interpret.
For more details about how to interpret the results of the comparison function to determine different types of compatibility, please see the [howto guide on comparing sequencing collections](compare_collections.md).

### 2.4 OpenAPI documentation

In addition to the primary top-level endpoints, it is RECOMMENDED that the service provide `/openapi.json`, an OpenAPI-compatible description of the endpoints.

---
### 3. Ancillary attribute management: recommended non-inherent attributes

In *Section 1: Encoding*, we distinguished between *inherent* and *non-inherent* attributes.
Non-inherent attributes provide a standardized way for implementations to store and serve additional, third-party attributes that do not contribute to the digest.
As long as separate implementations keep such information in non-inherent attributes, the identifiers will remain compatible.
Furthermore, the structure for how such non-inherent metadata is retrieved will be standardized.
Here, we specify standardized, useful non-inherent attributes that we recommend.

#### 3.2 The `sorted_name_length_pairs` attribute (`RECOMMENDED`)

The `sorted_name_length_pairs` attribute is a *non-inherent* attribute of a sequence collection with a formal definition, provided here.
It is `RECOMMENDED` that all seqcol implementations add this attribute to all sequence collections.
When digested, this attribute provides an identifier for an order-invariant coordinate system for a sequence collection.
Because it is *non-inherent*, it does not affect the identity (digest) of the collection.
It is created deterministically from the `names` and `lengths` attributes in the collection; it *does not* depend on the actual sequence content, so it is consistent across two collections with different sequence content if they have the same `names` and `lengths`, which are correctly collated, but with pairs not necessarily in the same order.
For rationale and use cases of `sorted_name_length_pairs`, see [*Footnote F6*](#f6-use-cases-for-the-sortednamelengthpairs-non-inherent-attribute).

Algorithm: 

1. Lump together each name-length pair from the primary collated `names` and `lengths` into an object, like `{"length":123,"name":"chr1"}`.
2. Canonicalize JSON according to the seqcol spec (using RFC-8785).
3. Digest each name-length pair string individually.
4. Sort the digests lexicographically.
5. Add as a non-inherent, non-collated attribute to the sequence collection object.

## Footnotes

### F1. Why use an array-oriented structure instead of a sequence-oriented structure?

In the canonical seqcol object structure, we first organize the sequence collection into what we called an "array-oriented" data structure, which is a list of collated arrays (names, lengths, sequences, *etc.*).
An alternative we considered was a "sequence-oriented" structure, which would group each sequence with some attributes, like `{name, length, sequence}`, and structure the collection as an array of such objects.
While the latter is intuitive, as it captures each sequence object with some accompanying attributes as independent entities, there are several reasons we settled on the array-oriented structure instead: 

  1. Flexibility and backwards compatibility of sequence attributes. What happens for an implementation that adds a new attribute? For example, if an implementation adds a `topology` attribute to the sequences, in the sequence-oriented structure, this would alter the sequence objects and thereby change their digests. In the array-based structure, since we digest the arrays individually, the digests of the other arrays are not changed. Thus, the array-oriented structure emphasizes flexibility of attributes, where the sequence-oriented structure would emphasize flexibility of sequences. In other words, the array-based structure makes it straightforward to mix-and-match *attributes* of the collection. Because each attribute is independent and not integrated into individual sequence objects, it is simpler to select and build subsets and permutations of attributes. We reasoned that flexibility of attributes was desirable.

  2. Conciseness. Sequence collections may be used for tens of thousands or even millions of sequences. For example, a transcriptome may contain a million transcripts. The array-oriented data structure is a more concise representation for digesting collections with many elements because the attribute names are specified once *per collection* instead of once *per element*. Furthermore, the level 1 representation of the sequence collection is more concise for large collections, since we only need one digest *per attribute*, rather than one digest *per sequence*. 

  3. Utility of intermediate digests. The array-oriented approach provides useful intermediate digests for each attribute. This digest can be used to test for matching sets of sequences, or matching coordinate systems, using the individual component digests. With a sequence-oriented framework, this would require traversing down a layer deeper, to the individual elements, to establish identity of individual components. The alternative advantage we would have from a sequence-oriented structure would be identifiers for *annotated sequences*. We can gain the advantages of these digests through adding a custom non-inherent, but collated attribute that calculates a unique digest for each element based on the selected attributes of interest, *e.g.* `named_sequences` (digest of *e.g.* `b'{"name":"chr1","sequence":"SQ.2648ae1bacce4ec4b6cf337dcae37816"}'`).

See [ADR on 2021-06-30 on array-oriented structure](/decision_record/#2021-06-30-use-array-based-data-structure-and-multi-tiered-digests)


### F2. Collated attributes

In JSON Schema, there are 2 ways to qualify properties: 1) a local qualifier, using a key under a property; or 2) an object-level qualifier, which is specified with a keyed list of properties up one level.
For example, you annotate a property's `type` with a local qualifier, underneath the property, like this:

```console
properties:
  names:
    type: array
```

However, you specify that a property is `required` by adding it to an object-level `required` list that's parallel to the `properties` keyword:

```console
properties:
  names:
    type: array
required:
  - names
```

In sequence collections, we chose to define `collated` as a local qualifier. Local qualifiers fit better for qualifiers independent of the object as a whole.
They are qualities of a property that persist if the property were moved onto a different object.
For example, the `type` of an attribute is consistent, regardless of what object that attribute were defined on.
In contrast, object-level qualifier lists fit better for qualifiers that depend on the object as a whole.
They are qualities of a property that depend on the object context in which the property is defined.
For example, the `required` modifier is not really meaningful except in the context of the object as a whole. A particular property could be required for one object type, but not for another, and it's really the object that induces the requirement, not the property itself.

We reasoned that `inherent`, like `required`, describes the role of an attribute in the context of the whole object; an attribute that is inherent to one type of object need not be inherent to another.
Therefore, it makes sense to treat this concept the same way JSON schema treats `required`.
In contrast, the idea of `collated` describes a property independently: Whether an attribute is collated is part of the definition of the attribute; if the attribute were moved to a different object, it would still be collated.


### F3. Details of inherent and non-inherent attributes

The specification in section 1, *Encoding*, described how to structure a sequence collection and then apply an algorithm to compute a digest for it.
What if you have ancillary information that goes with a collection, but shouldn't contribute to the digest?
We have found a lot of useful use cases for information that should go along with a seqcol, but should not contribute to the *identity* of that seqcol.
This is a useful construct as it allows us to include information in a collection that does not affect the identifier that is computed for that collection.
One simple example is the "author" or "uploader" of a reference sequence; this is useful information to store alongside this collection, but we wouldn't want the same collection with two different authors to have a different identifier! Seqcol refers to these as *non-inherent attributes*, meaning they are not part of the core identity of the sequence collection.
Non-inherent attributes are defined in the seqcol schema, but excluded from the `inherent` list. 

See: [ADR on 2023-03-22 regarding inherent attributes](/decision_record/#2023-03-22-seqcol-schemas-must-specify-inherent-attributes)

### F4. Sequence collections without sequences

Typically, we think of a sequence collection as consisting of real sequences, but in fact, sequence collections can also be used to specify collections where the actual sequence content is irrelevant.
Since this concept can be a bit abstract for those not familiar, we'll try here to explain the rationale and benefit of this.
First, consider that in a sequence comparison, for some use cases, we may be primarily concerned only with the *length* of the sequence, and not the actual sequence of characters.
For example, BED files provide start and end coordinates of genomic regions of interest, which are defined on a particular sequence.
On the surface, it seems that two genomic regions are only comparable if they are defined on the same sequence. However, this not *strictly* true; in fact, really, as long as the underlying sequences are homologous, and the position in one sequence references an equivalent position in the other, then it makes sense to compare the coordinates.
In other words, even if the underlying sequences aren't *exactly* the same, as long as they represent something equivalent, then the coordinates can be compared.
A prerequisite for this is that the *lengths* of the sequence must match; it wouldn't make sense to compare position 5,673 on a sequence of length 8,000 against the same position on a sequence of length 9,000 because those positions don't clearly represent the same thing; but if the sequences have the same length and represent a homology statement, then it may be meaningful to compare the positions. 

We realized that we could gain a lot of power from the seqcol comparison function by comparing just the name and length vectors, which typically correspond to a coordinate system.
Thus, actual sequence content is optional for sequence collections.
We still think it's correct to refer to a sequence-content-less sequence collection as a "sequence collection" -- because it is still an abstract concept that *is* representing a collection of sequences: we know their names, and their lengths, we just don't care about the actual characters in the sequence in this case.
Thus, we can think of these as a sequence collection without sequence characters.


### F5. The GA4GH digest algorithm

The GA4GH digest algorithm, `sha512t24u`, was created as part of the [Variation Representation Specification standard](https://vrs.ga4gh.org/en/stable/impl-guide/computed_identifiers.html). 
This procedure is described as ([Hart _et al_. 2020](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0239883)):

- performing a SHA-512 digest on a binary blob of data
- truncate the resulting digest to 24 bytes
- encodes the 24 bytes using `base64url` ([RFC 4648](https://datatracker.ietf.org/doc/html/rfc4648#section-5)) resulting in a 32 character string

In Python, the digest can be computed with this function:

```python
def sha512t24u_digest(seq):
    """ GA4GH digest function """
    offset = 24
    digest = hashlib.sha512(seq.encode()).digest()
    tdigest_b64us = base64.urlsafe_b64encode(digest[:offset])
    return tdigest_b64us.decode("ascii")
```

See: [ADR from 2023-01-25 on digest algorithm](/decision_record/#2023-01-25-digest-algorithm)

### F6. Use cases for the `sorted_name_length_pairs` non-inherent attribute

One motivation for this attribute comes from genome browsers, which may display genomic loci of interest (*e.g.* BED files).
The genome browser should only show BED files if they annotate the same coordinate system as the reference genome.
This is looser than strict identity, since we don't really care what the underlying sequence characters are, as long as the positions are comparable.
We also don't care about the order of the sequences.
Instead, we need them to match level 1 digest of the `sorted_name_length_pairs` attribute.
Thus, to assert that a BED file can be viewed for a particular genome, we compare the `sorted_name_length_pairs` digest of our reference genome with the sequence collection used to generate the file.
There are only two possibilities for compatibility: 1) If the digests are equal, then the data file is directly compatible; 2) If not, we must check the `comparison` endpoint to see whether the `sorted_name_length_pairs` attribute of the sequence collection is a direct subset of the same array in the sequence collection attached to the genome browser instance.
If so, the data file is still compatible. 

For efficiency, if the second case is true, we may cache the `sorted_name_length_pairs` digest in a list of known compatible reference genomes.
In practice, this list will be short.
Thus, in a production setting, the full compatibility check can be reduced to a lookup into a short, pre-generated list of `sorted_name_length_pairs` digests.

See: [ADR from 2023-07-12 on sorted name-length pairs](/decision_record/#2023-07-12-implementations-should-provide-sorted_name_length_pairs-and-comparison-endpoint)
