# `retrieval/retriever.py` — Detailed Walkthrough

**File location:** `inkly/retrieval/retriever.py`  
**Role:** Figures out which plugins are actually relevant to the user's question, so the runtime doesn't run all of them every time.

---

## The Problem This Solves

If Inkly ran every plugin for every question, prompts would be bloated with irrelevant context. A question about job failures doesn't need partition info. A question about the queue doesn't need Gaussian documentation. The retriever's job is to narrow down the list to only the plugins that matter.

---

## How It Works — The Big Picture

The retriever uses a technique called **vector similarity search**:

1. Every plugin is converted to a block of text (name + description + example queries).
2. That text is converted into a **vector** — a list of numbers that captures the meaning of the text.
3. When a user asks a question, the question is also converted into a vector.
4. The question vector is compared to all plugin vectors mathematically.
5. The plugins with the most similar vectors are selected.

This is the same general idea behind how search engines and recommendation systems work.

---

## Key Vocabulary

| Term | What It Means |
|---|---|
| **Vector** | A list of numbers representing the meaning of a piece of text |
| **Embedding** | The process of converting text into a vector |
| **TF-IDF** | The specific embedding method used here — scores words by how important they are in a document relative to all documents |
| **Vector store** | A database of stored vectors that can be searched efficiently |
| **Cosine similarity** | The math used to measure how "close" two vectors are (how similar two texts are) |
| **top_k** | How many top results to return (configured as `3` in `config.toml`) |

---

## `RetrievalResult` Dataclass

```python
@dataclass(frozen=True)
class RetrievalResult:
    item_id: str
    name: str
    category: str
    score: float
    item_type: str = "plugin"
```

A lightweight result object. The `score` field is the similarity score between the user's query and this plugin — higher is more relevant.

---

## `PluginRetriever.__init__()`

```python
self.embedder = TfidfEmbedder()
self.store = JsonVectorStore(self.index_path)
```

Two key components:
- **`TfidfEmbedder`** — converts text into vectors using TF-IDF.
- **`JsonVectorStore`** — stores and searches vectors, persisted to a JSON file on disk at `~/.inkly/retrieval_index.json`.

---

## `build_plugin_text()` — Describing a Plugin in Text

```python
parts = [
    f"name: {plugin.name}",
    f"category: {plugin.category}",
    f"description: {plugin.description}",
]
if plugin.example_queries:
    parts.append("example_queries:")
    parts.extend(f"- {query}" for query in plugin.example_queries)
```

Before a plugin can be indexed, it needs to be described as a single block of text. This method assembles that text from the plugin's metadata. The `example_queries` field is especially valuable here — they look like real user questions, which means they embed similarly to real user questions.

---

## `rebuild_index()` — Building the Search Index

This method runs when there's no existing index or the plugin set has changed:

1. Build the text description for every plugin.
2. **Fit** the embedder — it learns which words are important across all plugin descriptions.
3. Clear any old index data.
4. **Encode** each plugin's text into a vector.
5. Store all vectors in the `JsonVectorStore`.
6. Save to disk.

This is a one-time (or infrequent) setup cost. After this, searching is fast.

---

## `_ensure_index()` — Reuse or Rebuild

```python
if self.store.load() and set(self.store.items) == set(plugins):
    # reuse existing index
else:
    self.rebuild_index(plugins)
```

Checks whether a saved index exists and whether it still matches the current set of plugins. If yes, reuse it (much faster). If no (e.g. a new plugin was added), rebuild it.

---

## `classify_categories()` — Pre-Filtering by Category

Before doing a full vector search across all plugins, the retriever first asks: **what category of plugin is this question likely about?**

```python
classifier = CategoryClassifier(self.embedder, categories)
predictions = classifier.predict(query, top_n=2)
return [row.category for row in predictions if row.score > 0.0]
```

For example, a question about "how busy is the queue" would likely score highly for the `queue-status` category, meaning only queue-related plugins need to be searched. This **narrows the search space** before the more expensive vector comparison happens.

If no category scores above zero, all categories are allowed (fallback behavior).

---

## `search_plugins()` — Full Search Flow

```python
predicted_categories = set(self.classify_categories(query, plugins))
allowed_ids = [p.name for p in plugins.values() if p.category in predicted_categories]
query_vector = self.embedder.encode(query)
hits = self.store.search(query_vector, top_k=top_k, allowed_item_ids=allowed_ids, ...)
```

Step by step:
1. Classify likely categories for the query.
2. Build a list of plugin IDs restricted to those categories.
3. Encode the query into a vector using the **same fitted embedder** (important — must use the same vocabulary).
4. Search the vector store for the closest matching plugin vectors.
5. If the restricted search finds nothing and `fallback_to_all_plugins` is true, repeat the search across all plugins with no score threshold.

---

## `select_plugins()` — Final Step for the Runtime

```python
results = self.search_plugins(query, plugins, top_k=top_k)
return [plugins[result.name] for result in results if result.name in plugins]
```

This is the method the runtime actually calls. It runs `search_plugins()` and maps the `RetrievalResult` objects back to the actual `Plugin` objects so the runtime can call `plugin.run()` on them.

---

## Full Retrieval Flow Summary

```
User query: "how busy is the cluster?"
        │
        ├── classify_categories()
        │       └── predicts: ["queue-status"]
        │
        ├── filter plugins to queue-status category
        │
        ├── encode query → query vector
        │
        ├── search vector store (queue-status plugins only)
        │       └── returns: [queue_status (score=0.87), node_info (score=0.61)]
        │
        └── select_plugins() returns: [Plugin(queue_status), Plugin(node_info)]
                │
                └── runtime runs these two, skips the rest
```

---

## Next Files to Explore

- `inkly/retrieval/embedding.py` — the TF-IDF embedder that converts text to vectors
- `inkly/retrieval/vector_store.py` — how vectors are stored and searched on disk
- `inkly/core/conversation.py` — how conversation history is stored and used

---

## Related Notes

- [[runtime.py]] — calls `select_plugins()` during `handle_query`
- [[Plugin System]] — the plugins that retriever selects from
- [[Inkly Codebase Notes Start]] — overview of what retrieval does in plain terms
