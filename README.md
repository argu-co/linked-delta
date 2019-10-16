# linked-delta

Extensible protocol for the transfer of delta updates in linked datasets.

## In short

To communicate changes to some (linked/RDF) dataset, special graph names can be used to convey the changes to that set.

## Example using N-Quads

Initial state:
``` turtle
<http://example.org/resource> <http://example.org/predicate> "Old value 🙈" .
```
Linked-Delta:
``` nquads
<http://example.org/resource> <http://example.org/predicate> "New value 🐵" <http://purl.org/linked-delta/replace> .
```
New state:
```nquads
<http://example.org/resource> <http://example.org/predicate> "New value 🐵" .
```

## Protocol

Status: draft

### Definitions

**delta**: A graph of quad statements

**Delta repository**: a set of one or more graphs, which SHOULD NOT contain statements in the default graph which should be processed according to this protocol.

**The underlying triple**: the [subject, predicate, object] combination (so without the graph) of the quad which is processed.

**Store** or **target store**: the target rdf graph or repository against which the update is executed. This generally is either the default graph or some main repository the application uses to run its logic against.

### Processing

To keep behaviour consitent across implementations, it is RECOMMENDED that, when recieving a delta repository, the processor processes the statements in order of recieving them. Though vocabularies MAY define to have idempotent behaviour regardless of processing order.

Processing all the delta repositories in order of some origin store SHOULD result in a duplicate store in the target store.

It is NOT RECOMMENDED to use blank nodes in a delta repository. When the underlying triple's object refers to a blank node, the blank node SHOULD be included in the delta repository, unless the graph name IRI specifies otherwise.

## Base vocabulary

Currently three different graph name IRI's have been (not yet fully) defined as to how the triple data in that quad should be processed:

### ld:add

Graph IRI: http://purl.org/linked-delta/add

The underlying triple SHOULD be added to the store. If the full [s,p,o] combination is already present, it SHOULD be omitted. If a triple with the same [s,p] combination is present, but has a different object, the triple SHOULD be added to the store in addition to the present statements.

### ld:remove

Graph IRI: http://purl.org/linked-delta/remove

All triples that match the same [s,p] combination as the underlying triple SHOULD be removed from the store. The underlying triple should match as well if it was added beforehand.

When the object of the underlying triple refers to a blank node, the origin MAY omit the inclusion of the referenced resource.

### ld:replace

Graph IRI: http://purl.org/linked-delta/replace

The [s,p] combination of the triple MUST be processed with the logic of ld:remove. After which all the underlying triples of the repository MUST be added according to the ld:add logic.

### Order

The processor MUST execute the logic of the above defined in the following order: 
1. ld:remove
2. ld:replace
3. ld:add

## When use linked-delta

- Communitate state changes / events of RDF data
- Replicate / sync datasets
- Record changes / Event Sourcing
- Alert / listener / notification systems
- A simpler alternative to SPARQL-Update
- No need for new parsers - many RDF parsers already support N-Quads.

## How we use it

- We use this in [Argu.co](https://argu.co) to communicate state changes between server and the front-end client app, and to describe actions. These are executed using [link-lib](https://github.com/fletcher91/link-lib/).
- We also use this in [ori_api](https://github.com/ontola/ori_api/) in a Kafka bus, as a large stream of events that can be played back (i.e. Event Sourcing). This application converts all linked-deltas into a filled NGINX server with variously serialized RDF files (Turtle, RDF/XML, N-Triples, etc.).

## Motivation

The current paradigm is to program logic on what to do after some request is done, based on the status code
and the body (display a message, redirect the user, update counters, etc.). Though this is supported in link with 
`execActionByIRI`, it is somewhat of an anti-pattern to duplicate such logic or to [change the language of the
back-end](https://en.wikipedia.org/wiki/Isomorphic_JavaScript) to share code. See [that repo](https://github.com/fletcher91/link-lib#usage) for information on the schema.org-based hypermedia action system in combination with this protocol to build full applications.

Link has a somewhat different approach to updating the client-side state. By viewing the back-end service as a basic
state-machine and the client application as a reflection of the application state, the back-end in essence can be viewed
as a function with some side-effects. We place a constraint on the server that it SHOULD send back the side-effects it
created in order to fulfill the request, the front-end can then process those side-effects causing the client to be in
sync with the server again.

So in short, the server should send a delta of the server state before and after the request was processed. The default
behaviour is to add all the statements returned by the server, so if a resource was created, its triples should be sent
back to the client. When data was changed or deleted*, the statements should be contained in a named graph to indicate to
the client that the matching statements should be removed (<ld:remove>).


## Related projects / alternatives

- [rdf-delta](https://afs.github.io/rdf-delta/).
- [json-patch](http://jsonpatch.com/)
- [JSON-LD-PATCH](https://github.com/digibib/ls.ext/wiki/JSON-LD-PATCH)
