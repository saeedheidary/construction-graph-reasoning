# Interactive LLM-Assisted Reasoning-Rule Generation Prompts

This folder contains the prompt library used in the interactive reasoning-rule generation pipeline.

The implementation includes nine LLM-assisted stages and one deterministic schema-retrieval stage. The deterministic schema-retrieval stage has no LLM prompt; it retrieves the live Neo4j schema and representative node samples and passes the resulting JSON snapshot to later stages.
