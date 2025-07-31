---
layout: post
title:  "Querying Bitcoin data using RDF and AllegroGraph"
date:   2019-10-10 22:34:00 +0300
categories: jekyll update
---

## Introduction
Bitcoin operates by maintaining the data about all value transfers in a single
database, whose integrity and consistency is ensured by a set of cryptographic
protocols, the most important of which is called **PoW** (*Proof-of-Work*). This
data forms a graph with two general types of relations:
- *block membership relations* introduce ordering into the transaction data by
  building a timestamped sequence (chain) of groups (blocks) of transactions,
  and are merely used to verify the integrity and consistency of the database
  according to the network consensus rules;
- *transaction chaining relations* provide monetary properties for Bitcoin
  system by creating links between chunks of value, tracking their movement from
  the moment of generation during mining to point in time when they are used.

By definition, Bitcoin chain data is an immutable set of public records that
represent its complete transaction history, which makes it an important global
resource, while its transparent graph structure makes it an easy subject of
various analysis methods. These methods are actively researched from two
opposite points of view: as means of undermining privacy of the transaction data
(see Bitfury's [Crystal]) and as means of protecting against that (see
[CoinJoin] and Samourai's [Whirlpool] technologies).

One of the simplest means of graph analysis is graph querying with specialized
querying languages like [SPARQL]. Moreover, there exists a set of powerful
standard tools for representing and exploring graph data with such languages:
[RDF - Resource Description Framework][RDF].

In order to be able to use SPARQL for loading and querying Bitcoin database, we
need to build an RDF model which defines the properties of Bitcoin entities as
well as relations between them. To make our approach as general as possible, we
will model Bitcoin data directly from its native representation. This model can
then be used as a guideline for converting the Bitcoin data into a set of
triples suitable for storing in a graph database. We will be using AllegroGraph
for the purpose of storing the triples and as a querying engine.

## Block data model
In all Turtle examples we assume the following standard namespaces are available:
- `RDF` (http://www.w3.org/1999/02/22-rdf-syntax-ns#);
- `RDFS` (http://www.w3.org/2000/01/rdf-schema#);
- `XSD` (http://www.w3.org/2001/XMLSchema#);
- `OWL` (http://www.w3.org/2002/07/owl#).

These resources together form a powerful standard language for describing RDF
models and in particular provide all necessary definitions to build the Bitcoin
model.

Bitcoin block is a container for transactions, whose main consensus property is
a hash of the previous block (some properties omitted for simplicy):
```turtle
<Block>
        rdf:type     rdfs:Class;

<hash>
        rdf:type     rdfs:Property;
        rdfs:comment "Hex-encoded double-hash of the block header that is used in PoW to chain blocks.";
        rdfs:domain  <Block>;
        rdfs:range   xsd:string.

<height>
        rdf:type     rdfs:Property;
        rdfs:comment "Height of the block is an index of the block in the chain, starting from 0.";
        rdfs:domain  <Block>;
        rdfs:range   xsd:integer.

<transaction>
        rdf:type     rdfs:Property;
        rdfs:comment "Transaction-Block membership relation.";
        rdfs:domain  <Block>;
        rdfs:range   <Transaction>.

<prevBlock>
        rdf:type      rdfs:Property;
        rdfs:comment  "Introduce ordering into set of Bitcoin blocks.";
        rdfs:domain   <Block>;
        rdfs:range    <Block>.

<nextBlock>
        rdf:type      rdfs:Property;
        rdfs:comment  "Helper inverse of prevBlock property.";
        rdfs:domain   <Block>;
        rdfs:range    <Block>;
        owl:inverseOf <prevBlock>.
```

Value in Bitcoin is represented by a set of entities called *transaction
outputs* - pairs `(<amount> <lock script>)` where `<lock script>` is a certain
condition expressed in a simple stack language called Bitcoin script:
```turtle
<TxOutput>
        rdf:type     rdfs:Class;

<amount>
        rdf:type     rdfs:Property;
        rdfs:comment "Amount of atomic units locked in this UTXO.";
        rdfs:domain  <TxOutput>;
        rdfs:range   xsd:integer.

<lockScript>
        rdf:type     rdfs:Property;
        rdfs:comment "Script that describes an unlocking condition of the given output.";
        rdfs:domain  <TxOutput>;
        rdfs:range   <Script>.
```

Value from an output can be transfered by providing an entity called
*transaction input* - a triple `(<transaction id> <output index> <unlock
script>)` such that `<lock script>` and `<unlock script>` when combined, must
evaluate to 1, forming a proof of ownership over the given value (some
properties omitted for brevity):
```turtle
<TxInput>
        rdf:type     rdfs:Class;

<outputHash>
        rdf:type     rdfs:Property;
        rdfs:comment "Hash of the transaction whose output is spent by this input.";
        rdfs:domain  <TxInput>;
        rdfs:range   xsd:string.

<outputIndex>
        rdf:type     rdfs:Property;
        rdfs:comment "Index of the transcation output which is spent by this input.";
        rdfs:domain  <TxInput>;
        rdfs:range   xsd:integer.

<unlockScript>
        rdf:type     rdfs:Property;
        rdfs:comment "Script that is appended to locking script in order to unlock the UTXO.";
        rdfs:domain  <TxInput>;
        rdfs:range   <Script>.
```

All the value in the Bitcoin system is contained in a *UTXO* (*Unspent
Transaction Output*) set - a set of transaction outputs, for which no inputs
exist within the system so far.

Finally, Bitcoin transaction is just a special record that represents a
destruction of a subset of existing UTXOs and creation of new UTXOs under the
condition that the sum of the amounts in the destroyed UTXOs is larger than or
equal to the sum of created ones (some properties omitted for brevity):
```turtle
<Transaction>
        rdf:type     rdfs:Class;

<txid>
        rdf:type     rdfs:Property;
        rdfs:comment "Hash of the transaction which is used to uniquely identify it.";
        rdfs:domain  <Transaction>;
        rdfs:range   xsd:string.

<input>
        rdf:type     rdfs:Property;
        rdfs:comment "Input-Transaction membership relation.";
        rdfs:domain  <Transaction>;
        rdfs:range   <Input>.

<output>
        rdf:type     rdfs:Property;
        rdfs:comment "Output-Transaction membership relation.";
        rdfs:domain  <Transaction>;
        rdfs:range   <Output>.
```

The following Turtle example demonstrates how this RDF model can be used to
represent complete chain database (block given is a *genesis* block - the first
block in the mainnet Bitcoin chain; script strings omitted for brevity):
```turtle
@prefix : <https://raw.githubusercontent.com/franzinc/agraph-examples/master/data/bitcoin/model.ttl#>
@prefix btc: <bitcoin://>

btc:blk0
    :height 0;
    :hash "000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f";
    :time 1231006505;
    :version 1;
    :transaction btc:4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b.

btc:4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
    :lockTime 0;
    :input [:unlockScript "...".];
    :output [:amount 5000000000; :lockScript "...".].
```


## Loading Bitcoin data
In order to be able to run queries, we first need to prepare a repository with
chain data. We will be using a simple Python tool for loading the chain data
into the AllegroGraph instance, which can be found in the
[`data/bitcoin`][agraph-bitcoin] directory of the
[`agraph-examples`][agraph-examples] repository on GitHub along with the model
we described above.

The following examples assume AllegroGraph triple store and assume it is already
installed and running on the target machine. The following AG instance settings
settings are assumed as well:
- *host*: `localhost` (default);
- *port*: `10035` (default);
- *username*: `aguser`;
- *password*: `agpassword`.

We need to make sure we have an access to a running `bitcoind` instance with RPC
port open. We assume following `bitcoind` settings:
- *host*: `localhost` (default);
- *port*: `8332` (default);
- *username*: `btcuser`;
- *password*: `btcpassword`.

First, we will install the tool by cloning the
[`agraph-examples`][agraph-examples] repository, setting up a virtual
environment and installing the dependencies:
``` bash
git clone https://github.com/franzinc/agraph-examples
cd agraph-examples/data/bitcoin
python3 -m venv .
source ./bin/activate
pip3 install -r requirements.txt
```

Now we are good to go. The following command starts the process of loading
bitcoin chain data into a clean AG repository named `bitcoin` using 4 loader
processes:
``` bash
./convert.py \
    --source=http://btcuser:btcpassword@localhost:8332 \
    --destination=http://aguser:agpassword@localhost:10035 \
    --name=bitcoin \
    --workers=4 \
    --clear
```

Note that loading the whole Bitcoin chain takes quite a lot of time and
space. Using the following setup:

- machine running the loader - 2 x 4-core Intel(R) Xeon(R) L5420 @ 2.50GHz, 32
  Gb of memory,

- machine running the AllegroGraph instance - 2 x 6-core AMD Opteron(tm) 2439 @
  2.80GHz SE, 64 Gb of memory,

we were able to load 80% of all chain data in approximately 14 days. The disk
space required for the 80% chain database, which contains 11.5 billion triples,
is 1.1 Tb, around 4 times more than for a raw binary Bitcoin database maintained
by the `bitcoind` node, which at the moment of writing takes around 270
Gb. Given these numbers, for experimenting purposes, it might make sense to load
only a subset of blocks we are interested in by using the
`--start-height`/`--end-height` parameters for the `convert` tool:

``` bash
./convert.py \
    --source=http://btcuser:btcpassword@localhost:8332 \
    --destination=http://aguser:agpassword@localhost:10035 \
    --name=bitcoin \
    --workers=4 \
    --clear \
    --start-height=570000 \
    --end-height=580000
```


## Example queries
Following are the examples of using SPARQL to extract different information
about block data:
- number of known blocks:
  ```sparql
  PREFIX : <https://raw.githubusercontent.com/franzinc/agraph-examples/master/data/bitcoin/model.ttl#>
  SELECT (COUNT(*) AS ?count) WHERE { ?b a btcm:Block. }
  ```

- total number of transactions:
  ```sparql
  PREFIX : <https://raw.githubusercontent.com/franzinc/agraph-examples/master/data/bitcoin/model.ttl#>
  SELECT (COUNT(*) AS ?count) WHERE { ?tx a btcm:Transaction. }
  ```

- transaction in block 570001:
  ```sparql
  PREFIX : <https://raw.githubusercontent.com/franzinc/agraph-examples/master/data/bitcoin/model.ttl#>
  SELECT ?txid
  WHERE {
    ?b a :Block.
    ?b :height "570001"^^xsd:int.
    ?b :transaction ?tx.
    ?tx :txid ?txid.
  }
  ```

- transactions sending more than 1000 BTC:
  ```sparql
  PREFIX : <https://raw.githubusercontent.com/franzinc/agraph-examples/master/data/bitcoin/model.ttl#>
  SELECT ?tx
  WHERE {
    ?b a :Block.
    ?b :transaction ?tx.
    ?tx :output ?out.
    ?out :amount ?amt.
  }
  GROUP BY ?tx
  HAVING (SUM(?amt) > 100000000000)
  ```

- transactions sending BTC to Pirate Bay's address:
  ```sparql
  PREFIX : <https://raw.githubusercontent.com/franzinc/agraph-examples/master/data/bitcoin/model.ttl#>
  SELECT ?tx
  WHERE {
    ?tx :output ?out.
    ?out :lockScript ?s.
    FILTER REGEX (?s, "<tpb address>").
  }
  ```

[model]: https://raw.githubusercontent.com/franzinc/agraph-examples/master/data/bitcoin/model.ttl#
[RDF]: https://en.wikipedia.org/wiki/Resource_Description_Framework
[Crystal]: https://crystalblockchain.com/
[CoinJoin]: https://en.bitcoin.it/wiki/CoinJoin
[Whirlpool]: https://samouraiwallet.com/whirlpool
[SPARQL]: https://en.wikipedia.org/wiki/SPARQL
[agraph-bitcoin]: https://github.com/franzinc/agraph-examples/tree/master/data/bitcoin
[agraph-examples]: https://github.com/franzinc/agraph-examples
