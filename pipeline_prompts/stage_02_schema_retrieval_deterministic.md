# Stage 2 – Live Schema Snapshot Retrieval

## Prompt used in the implementation

```text
This stage does not use an LLM prompt.

The implementation retrieves the current Neo4j graph schema deterministically by querying:
- node labels and label counts;
- relationship types and counts;
- node properties;
- relationship properties;
- sample graph patterns;
- representative node samples.

The resulting schema snapshot is serialized as structured JSON and passed to later LLM-assisted stages.
```
