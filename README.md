<h1 align="center">neo4j-haystack</h1>

<p align="center">A <a href="https://docs.haystack.deepset.ai/docs/document_store"><i>Haystack</i></a> Document Store for <a href="https://neo4j.com/"><i>Neo4j</i></a>.</p>

<p align="center">
  <a href="https://github.com/prosto/neo4j-haystack/actions?query=workflow%3Aci">
    <img alt="ci" src="https://github.com/prosto/neo4j-haystack/workflows/ci/badge.svg" />
  </a>
  <a href="https://prosto.github.io/neo4j-haystack/">
    <img alt="documentation" src="https://img.shields.io/badge/docs-mkdocs%20material-blue.svg?style=flat" />
  </a>
  <a href="https://pypi.org/project/neo4j-haystack/">
    <img alt="pypi version" src="https://img.shields.io/pypi/v/neo4j-haystack.svg" />
  </a>
  <a href="https://img.shields.io/pypi/pyversions/neo4j-haystack.svg">
    <img alt="python version" src="https://img.shields.io/pypi/pyversions/neo4j-haystack.svg" />
  </a>
  <a href="https://pypi.org/project/haystack-ai/">
    <img alt="haystack version" src="https://img.shields.io/pypi/v/haystack-ai.svg?label=haystack" />
  </a>
</p>

----

**Table of Contents**

- [Overview](#overview)
- [Usage](#usage)
- [Installation](#installation)
- [License](#license)

## Overview

An integration of [Neo4j](https://neo4j.com/) graph database with [Haystack v2.0](https://docs.haystack.deepset.ai/v2.0/docs/intro)
by [deepset](https://www.deepset.ai). In Neo4j [Vector search index](https://neo4j.com/docs/cypher-manual/current/indexes-for-vector-search/)
is being used for storing document embeddings and dense retrievals.

The library allows using Neo4j as a [DocumentStore](https://docs.haystack.deepset.ai/v2.0/docs/document-store), and implements the required [Protocol](https://docs.haystack.deepset.ai/v2.0/docs/document-store#documentstore-protocol) methods. You can start working with the implementation by importing it from `neo4_haystack` package:

```python
from neo4_haystack import Neo4jDocumentStore
```

In addition to the `Neo4jDocumentStore` the library includes the following haystack components which can be used in a pipeline:

- [Neo4jEmbeddingRetriever](https://prosto.github.io/neo4j-haystack/reference/neo4j_retriever/#neo4j_haystack.components.neo4j_retriever.Neo4jEmbeddingRetriever) - is a typical [retriever component](https://docs.haystack.deepset.ai/v2.0/docs/retrievers) which can be used to query vector store index and find related Documents. The component uses `Neo4jDocumentStore` to query embeddings.
- [Neo4jDynamicDocumentRetriever](https://prosto.github.io/neo4j-haystack/reference/neo4j_retriever/#neo4j_haystack.components.neo4j_retriever.Neo4jDynamicDocumentRetriever) is also a retriever component in a sense that it can be used to query Documents in Neo4j. However it is decoupled from `Neo4jDocumentStore` and allows to run arbitrary [Cypher query](https://neo4j.com/docs/cypher-manual/current/queries/) to extract documents. Practically it is possible to query Neo4j same way `Neo4jDocumentStore` does, including vector search.

The `neo4j-haystack` library uses [Python Driver](https://neo4j.com/docs/api/python-driver/current/api.html#api-documentation) and
[Cypher Queries](https://neo4j.com/docs/cypher-manual/current/introduction/) to interact with Neo4j database and hide all complexities under the hood.

`Neo4jDocumentStore` will store Documents as Graph nodes in Neo4j. Embeddings are stored as part of the node, but indexing and querying of vector embeddings using ANN is managed by a dedicated [Vector Index](https://neo4j.com/docs/cypher-manual/current/indexes-for-vector-search/).

```text
                                   +-----------------------------+
                                   |       Neo4j Database        |
                                   +-----------------------------+
                                   |                             |
                                   |      +----------------+     |
                                   |      |    Document    |     |
                write_documents    |      +----------------+     |
          +------------------------+----->|   properties   |     |
          |                        |      |                |     |
+---------+----------+             |      |   embedding    |     |
|                    |             |      +--------+-------+     |
| Neo4jDocumentStore |             |               |             |
|                    |             |               |index/query  |
+---------+----------+             |               |             |
          |                        |      +--------+--------+    |
          |                        |      |  Vector Index   |    |
          +----------------------->|      |                 |    |
               query_embeddings    |      | (for embedding) |    |
                                   |      +-----------------+    |
                                   |                             |
                                   +-----------------------------+
```

In the above diagram:

- `Document` is a Neo4j node (with "Document" label)
- `properties` are Document [attributes](https://docs.haystack.deepset.ai/v2.0/docs/data-classes#document) stored as part of the node.
- `embedding` is also a property of the Document node (just shown separately in the diagram for clarity) which is a vector of type `LIST[FLOAT]`.
- `Vector Index` is where embeddings are getting indexed by Neo4j as soon as those are updated in Document nodes.

## Installation

`neo4j-haystack` can be installed as any other Python library, using pip:

```bash
pip install --upgrade pip # optional
pip install neo4j-haystack
```

## Usage

Once installed, you can start using `Neo4jDocumentStore` as any other document stores that support embeddings.

```python
from neo4j_haystack import Neo4jDocumentStore

document_store = Neo4jDocumentStore(
    url="bolt://localhost:7687",
    username="neo4j",
    password="passw0rd",
    database="neo4j",
    embedding_dim=384,
    embedding_field="embedding",
    index="document-embeddings", # The name of the Vector Index in Neo4j
    node_label="Document", # Providing a label to Neo4j nodes which store Documents
)
```

Assuming there is a list of documents available you can write/index those in Neo4j, e.g.:

```python
documents: List[Document] = ...
document_store.write_documents(documents)
```

The full list of parameters accepted by `Neo4jDocumentStore` can be found in
[API documentation](https://prosto.github.io/neo4j-haystack/reference/neo4j_store/#neo4j_haystack.document_stores.neo4j_store.Neo4jDocumentStore.__init__).

Please notice you will need to have a running instance of Neo4j database (in-memory version of Neo4j is not supported). There are several options available:

- [Docker](https://neo4j.com/docs/operations-manual/5/docker/), other options available in the same Operations Manual
- [AuraDB](https://neo4j.com/cloud/platform/aura-graph-database/) - a fully managed Cloud Instance of Neo4j
- [Neo4j Desktop](https://neo4j.com/docs/desktop-manual/current/) client application

The simplest way to start database locally will be with Docker container:

```bash
docker run \
    --restart always \
    --publish=7474:7474 --publish=7687:7687 \
    --env NEO4J_AUTH=neo4j/passw0rd \
    neo4j:5.15.0
```

### Retrieving documents

`Neo4jEmbeddingRetriever` component can be used to retrieve documents from Neo4j by querying vector index using an embedded query. Below is a pipeline which finds documents using query embedding s well as [metadata filtering](https://docs.haystack.deepset.ai/v2.0/docs/metadata-filtering):

```python
from haystack import Document, Pipeline
from haystack.components.embedders import SentenceTransformersTextEmbedder
from neo4j_haystack import Neo4jEmbeddingRetriever, Neo4jDocumentStore

document_store = Neo4jDocumentStore(
    url="bolt://localhost:7687",
    username="neo4j",
    password="passw0rd",
    database="neo4j",
    embedding_dim=384,
    index="document-embeddings",
)

pipeline = Pipeline()
pipeline.add_component("text_embedder", SentenceTransformersTextEmbedder(model="sentence-transformers/all-MiniLM-L6-v2"))
pipeline.add_component("retriever", Neo4jEmbeddingRetriever(document_store=document_store))
pipeline.connect("text_embedder.embedding", "retriever.query_embedding")

result = pipeline.run(
    data={
        "text_embedder": {"text": "Query to be embedded"},
        "retriever": {
            "top_k": 5,
            "filters": {"field": "release_date", "operator": "==", "value": "2018-12-09"},
        },
    }
)

documents: List[Document] = result["retriever"]["documents"]
```

### Retrieving documents using Cypher

`Neo4jDynamicDocumentRetriever` is a flexible retriever component which can run a Cypher query to obtain documents. The above example of `Neo4jEmbeddingRetriever` could be rewritten without usage of `Neo4jDocumentStore`:

```python
from haystack import Document, Pipeline
from haystack.components.embedders import SentenceTransformersTextEmbedder

from neo4j_haystack import Neo4jClientConfig, Neo4jDynamicDocumentRetriever

client_config = Neo4jClientConfig(
    url="bolt://localhost:7687",
    username="neo4j",
    password="passw0rd",
    database="neo4j",
)

cypher_query = """
            CALL db.index.vector.queryNodes($index, $top_k, $query_embedding)
            YIELD node as doc, score
            MATCH (doc) WHERE doc.release_date = $release_date
            RETURN doc{.*, score}, score
            ORDER BY score DESC LIMIT $top_k
        """

embedder = SentenceTransformersTextEmbedder(model="sentence-transformers/all-MiniLM-L6-v2")
retriever = Neo4jDynamicDocumentRetriever(
    client_config=client_config, runtime_parameters=["query_embedding"], doc_node_name="doc"
)

pipeline = Pipeline()
pipeline.add_component("text_embedder", embedder)
pipeline.add_component("retriever", retriever)
pipeline.connect("text_embedder.embedding", "retriever.query_embedding")

result = pipeline.run(
    data={
        "text_embedder": {"text": "Query to be embedded"},
        "retriever": {
            "query": cypher_query,
            "parameters": {"index": "document-embeddings", "top_k": 5, "release_date": "2018-12-09"},
        },
    }
)

documents: List[Document] = result["retriever"]["documents"]
```

Please notice how query parameters are being used in the `cypher_query`:

- `runtime_parameters` is a list of parameter names which are going to be input slots when connecting components
    in a pipeline. In our case `query_embedding` input is connected to the `text_embedder.embedding` output.
- `pipeline.run` specifies additional parameters to the `retriever` component which can be referenced in the
    `cypher_query`, e.g. `top_k`.

## License

`neo4j-haystack` is distributed under the terms of the [MIT](https://spdx.org/licenses/MIT.html) license.
