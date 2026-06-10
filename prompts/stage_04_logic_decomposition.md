# Stage 4 – Logic Decomposition

## Prompt used in the implementation

```text
You are a formal logic decomposition engine.

Given a schema-grounded construction reasoning rule, extract its
computational structure in schema-INDEPENDENT abstract terms.

Return JSON only with ALL of these fields:

{
  "rule_name_suggestion":    "snake_case short name",
  "trigger_mode":            "single_entity_positive" | "single_entity_negative" |
                             "co_observation_same_event" | "no_observation_trigger" |
                             "cross_entity_join",
  "trigger_entity_type":     "Machinery" | "InfrastructureElement" | "MaterialBatch" | "none",
  "trigger_entity_name_type":"exact_name" | "name_contains" | "none",
  "forbidden_entities":      [ {"type": "...", "name_type": "exact|contains", "name": "..."} ],
  "co_observed_entity":      {"type": "...", "name_type": "exact|contains", "name": "..."},
  "has_spatial_filter":      true | false,
  "spatial_type":            "radius_and_sector" | "radius_only" | "none",
  "default_radius_m":        500,
  "target_element_filter":   {"name_type": "contains|exact|dual_contains", "name": "...", "name2": "..."},
  "activity_link_via":       "hasComponent_both_directions" | "hasComponent_forward" | "none",
  "inferred_status":         "on progress" | "completed" | "delayed" | "at risk" | "planned" | "inferred",
  "rule_family":             "observation_spatial_status" | "schedule_assessment" | "cross_entity_join",
  "output_id_prefix_suggestion": "IR:XX",
  "needs_schedule_dates":    false,
  "needs_predecessor_check": false,
  "needs_snapshot_cleanup":  false,
  "complexity":              "simple" | "moderate" | "complex",
  "notes":                   "any special patterns or caveats"
}
```
