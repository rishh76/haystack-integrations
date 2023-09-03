---
layout: integration
name: Milvus Document Store
description: Use the Milvus vector database with Haystack
authors:
    - name: Zilliz 
      socials:
        github: zilliztech
        twitter: zilliz_universe
pypi: https://pypi.org/project/milvus-haystack/
repo: https://github.com/milvus-io/milvus-haystack
type: Document Store
report_issue: https://github.com/milvus-io/milvus-haystack/issues
logo: /logos/milvus.png
---

An integration of [Milvus](https://milvus.io/) vector database with [Haystack](https://haystack.deepset.ai/).

Milvus is a flexible, reliable, and fast cloud-native, open-source vector database. It powers embedding similarity search and AI applications and strives to make vector databases accessible to every organization. Milvus can store, index, and manage a billion+ embedding vectors generated by deep neural networks and other machine learning (ML) models. This level of scale is vital to handling the volumes of unstructured data generated to help organizations to analyze and act on it to provide better service, reduce fraud, avoid downtime, and make decisions faster.
Milvus is a graduated-stage project of the LF AI & Data Foundation.

Use Milvus as storage for Haystack pipelines as `MilvusDocumentStore`.

🚀 See an example application that uses the `MilvusDocumentStore` to do Milvus documentation QA [here](https://github.com/TuanaCelik/milvus-documentation-qa).

## Installation

```bash
pip install milvus-haystack
```

## Usage

Once installed and running, you can start using Milvus with Haystack by initializing it: 

```python
from milvus_haystack import MilvusDocumentStore

document_store = MilvusDocumentStore()
```

### Writing Documents to MilvusDocumentStore

To write documents to your `MilvusDocumentStore`, create an indexing pipeline, or use the `write_documents()` function.
For this step, you may make use of the available [FileConverters](https://docs.haystack.deepset.ai/docs/file_converters) and [PreProcessors](https://docs.haystack.deepset.ai/docs/preprocessor), as well as other [Integrations](/integrations) that might help you fetch data from other resources. Below is the example indexing pipeline used in the Milvus Documentation QA demo, which makes use of the `Crawler` component.

#### Indexing Pipeline

```python
from haystack import Pipeline
from haystack.nodes import Crawler, PreProcessor, EmbeddingRetriever
from milvus_haystack import MilvusDocumentStore

document_store = MilvusDocumentStore(recreate_index=True, return_embedding=True, similarity="cosine")
crawler = Crawler(urls=["https://milvus.io/docs/"], crawler_depth=1, overwrite_existing_files=True, output_dir="crawled_files")
preprocessor = PreProcessor(
    clean_empty_lines=True,
    clean_whitespace=False,
    clean_header_footer=True,
    split_by="word",
    split_length=500,
    split_respect_sentence_boundary=True,
)
retriever = EmbeddingRetriever(document_store=document_store, embedding_model="sentence-transformers/multi-qa-mpnet-base-dot-v1")

indexing_pipeline = Pipeline()
indexing_pipeline.add_node(component=crawler, name="crawler", inputs=['File'])
indexing_pipeline.add_node(component=preprocessor, name="preprocessor", inputs=['crawler'])
indexing_pipeline.add_node(component=retriever, name="retriever", inputs=['preprocessor'])
indexing_pipeline.add_node(component=document_store, name="document_store", inputs=['retriever'])

indexing_pipeline.run()
```

### Using Milvus in a Retrieval Augmented Generative Pipeline

Once you have documents in your `MilvusDocumentStore`, it's ready to be used in any Haystack pipeline. For example, below is a pipeline that makes use of the ["deepset/question-answering"](https://prompthub.deepset.ai/?prompt=deepset%2Fquestion-answering) prompt that is designed to generate answers for the retrieved documents. Below is the example pipeline used in the Milvus Documentation QA deme that generates replies to queries using GPT-4:

```python
from haystack import Pipeline
from haystack.nodes import EmbeddingRetriever, PromptNode, PromptTemplate, AnswerParser
from milvus_haystack import MilvusDocumentStore

document_store = MilvusDocumentStore()

retriever = EmbeddingRetriever(document_store=document_store, embedding_model="sentence-transformers/multi-qa-mpnet-base-dot-v1")
template = PromptTemplate(prompt="deepset/question-answering", output_parser=AnswerParser())
prompt_node = PromptNode(model_name_or_path="gpt-4", default_prompt_template=template, api_key=YOUR_OPENAI_API_KEY, max_length=200)

query_pipeline = Pipeline()
query_pipeline.add_node(component=retriever, name="Retriever", inputs=["Query"])
query_pipeline.add_node(component=prompt_node, name="PromptNode", inputs=["Retriever"])
```