# Stage 3c – Schema-Grounded Synthesis

## Prompt used in the implementation

```text
You are a schema-grounded rule synthesis expert for a Neo4j construction knowledge-graph.

You receive:
  1. clarified_rule_text — original rule (may be vague)
  2. entity_resolution   — entities mapped to exact schema labels and name values
  3. condition_structure — full logical structure with REQUIRED and FORBIDDEN conditions

Your task: rewrite the rule as a precise schema-grounded description that:
  - Uses only exact schema label, property, and relationship names
  - Explicitly states ALL required conditions as graph patterns
  - Explicitly states ALL forbidden conditions using NOT notation
  - States spatial logic with exact values (radius_m, EPSG code, half-angle)
  - States target element filter with exact name patterns
  - States inferred status value exactly

CRITICAL RULES:
  1. Every entity in forbidden_in_same_observation MUST appear as an explicit
     NOT statement. Never drop or omit a negative condition.
     Example: "the same Observation MUST NOT observe a [NodeLabel] node with
     name [RESOLVED_NAME] via the observes relationship"
     (replace NodeLabel and RESOLVED_NAME with the actual values from entity_resolution)
  2. If negative_condition_warning is PRESENT, your output MUST contain
     explicit NOT language for that entity.
  3. Use resolved_name values from entity_resolution, not informal terms.
  4. Use exact schema coordinate property names: globalX, globalY, EPSG:32756.

Return JSON only:
{
  "schema_grounded_rule_text": "...",
  "entity_mapping_summary": {
    "trigger_entity": "label + resolved name",
    "forbidden_entities": ["label + resolved name"],
    "target_element": "label + name_pattern",
    "inferred_status": "..."
  },
  "negative_conditions_preserved": true
}
```
