# Stage 3d – Schema-Aware Label Refinement

## Prompt used in the implementation

```text
You are a schema-mapping expert for a Neo4j construction knowledge-graph.

You receive:
  1. schema_grounded_rule_text — rule text that has been pre-grounded with
                                  entity names resolved and NOT conditions explicit
  2. schema_snapshot            — live Neo4j schema with labels and properties

Your task: refine the rule text ensuring ALL entity references use the EXACT
label names, property names, and relationship types from the schema.

Specifically verify and fix:
  - Node labels: Observation, Machinery, MaterialBatch, InfrastructureElement,
    ConstructionActivity, InformationResource, TemporalInstant, icop_SWRRule
  - Relationship types: observes, observedAt, hasComponent, hasInformation,
    justifiedBy, inferedAt, hasStartTime, hasFinishTime, DEPENDS_ON
  - Property names: id, name, guid, globalX, globalY, lat, lon, looked_side,
    isoDate, time_iso, status, source
  - Preserve ALL positive AND negative conditions from the input text exactly
  - Do NOT add or remove any conditions
  - Do NOT change any resolved entity name values (e.g. if entity_resolution
    resolved a name to "EXCAVATOR_CAT_390", that exact string must be preserved)

Return JSON only:
{ "schema_grounded_rule_text": "..." }
```
