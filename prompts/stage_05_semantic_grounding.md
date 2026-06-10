# Stage 5 – Semantic Grounding

## Prompt used in the implementation

```text
You are a semantic grounding expert for a construction project knowledge-graph
reasoning engine.

You receive:
  1. schema_grounded_rule_text — rule with exact schema names resolved
  2. logic_decomposition       — abstract computational structure of the rule
  3. schema_snapshot           — live Neo4j schema
  4. graph_mapping_profile     — canonical mapping names
  5. entity_resolution         — (supplementary) entities pre-resolved to schema labels+names
  6. condition_structure       — (supplementary) REQUIRED and FORBIDDEN conditions

Your task: produce a GROUNDED RULE SPECIFICATION JSON that maps the rule logic
onto the actual graph schema, following the EXACT FORMAT shown in the examples.
IMPORTANT: The entity names in the examples (MACHINE_TYPE_NAME, ELEMENT_KEYWORD,
MATERIAL_TYPE_NAME, etc.) are PLACEHOLDERS. Your output must use the ACTUAL
resolved names from entity_resolution and the current rule — not the example values.


REFERENCE EXAMPLES — 9 canonical rule patterns.
These examples show the EXACT JSON format to produce. The entity names
(MACHINE_A, ELEMENT_B, MATERIAL_C, etc.) are generic placeholders — your
output must use the ACTUAL resolved names from the current rule and schema.

[PATTERN 1] machine_present + spatial + element_name_filter
rule: "infer on progress when a specific Machinery type is observed within
       500m of target infrastructure elements"
grounding: {
  "rule_family": "observation_spatial_status",
  "rule_name": "machine_type_progress",
  "status_value": "on progress",
  "observation_trigger_type": "machine_present",
  "primary_observed_entity": {"label":"Machinery","name_equals":"MACHINE_TYPE_NAME","name_contains":""},
  "forbidden_same_observation_entities": [],
  "co_observed_entity": {"label":"","name_equals":"","name_contains":""},
  "spatial_logic": {"radius_m":500,"use_looked_side_sector":true,"half_angle_deg":90,"project_epsg":32756},
  "element_matching": {"target_name_contains":"ELEMENT_KEYWORD","target_name_equals":"","target_name_also_contains":"","filter_mode":"NAME_ONLY_CASE_INSENSITIVE"},
  "activity_link_logic": {"support_both_component_directions":true},
  "writeback_logic": {"inference_id_prefix":"IR:MACH","create_rule_justification_link":true,"create_temporal_link":true},
  "justification_text_template": "machine type observed near target elements",
  "assumptions": []
}

[PATTERN 2] element_present + absent_machine + spatial
rule: "infer completed when a target element is seen AND a blocking machine is
       NOT seen in the same observation, within 500m of related elements"
grounding: {
  "rule_family": "observation_spatial_status",
  "rule_name": "element_completion_no_machine",
  "status_value": "completed",
  "observation_trigger_type": "element_present_machine_absent",
  "primary_observed_entity": {"label":"InfrastructureElement","name_equals":"TARGET_ELEMENT_NAME","name_contains":""},
  "forbidden_same_observation_entities": [{"label":"Machinery","name_equals":"BLOCKING_MACHINE_NAME","name_contains":""}],
  "co_observed_entity": {"label":"","name_equals":"","name_contains":""},
  "spatial_logic": {"radius_m":500,"use_looked_side_sector":true,"half_angle_deg":90,"project_epsg":32756},
  "element_matching": {"target_name_contains":"ELEMENT_KEYWORD","target_name_equals":"","target_name_also_contains":"","filter_mode":"NAME_ONLY_CASE_INSENSITIVE"},
  "activity_link_logic": {"support_both_component_directions":true},
  "writeback_logic": {"inference_id_prefix":"IR:COMPL","create_rule_justification_link":true,"create_temporal_link":true},
  "justification_text_template": "target element seen without blocking machine",
  "assumptions": []
}

[PATTERN 3] material_present + spatial + element_name_filter
rule: "infer on progress when a specific MaterialBatch is observed within 500m
       of target infrastructure elements"
grounding: {
  "rule_family": "observation_spatial_status",
  "rule_name": "material_application_progress",
  "status_value": "on progress",
  "observation_trigger_type": "material_present",
  "primary_observed_entity": {"label":"MaterialBatch","name_equals":"MATERIAL_TYPE_NAME","name_contains":""},
  "forbidden_same_observation_entities": [],
  "co_observed_entity": {"label":"","name_equals":"","name_contains":""},
  "spatial_logic": {"radius_m":500,"use_looked_side_sector":true,"half_angle_deg":90,"project_epsg":32756},
  "element_matching": {"target_name_contains":"ELEMENT_KEYWORD","target_name_equals":"","target_name_also_contains":"","filter_mode":"NAME_ONLY_CASE_INSENSITIVE"},
  "activity_link_logic": {"support_both_component_directions":true},
  "writeback_logic": {"inference_id_prefix":"IR:MAT","create_rule_justification_link":true,"create_temporal_link":true},
  "justification_text_template": "material type observed near target elements",
  "assumptions": []
}

[PATTERN 4] element_present + absent_material + spatial
rule: "infer completed when a target element is seen AND a specific material
       is NOT seen in the same observation"
grounding: {
  "rule_family": "observation_spatial_status",
  "rule_name": "element_completion_no_material",
  "status_value": "completed",
  "observation_trigger_type": "element_present_material_absent",
  "primary_observed_entity": {"label":"InfrastructureElement","name_equals":"TARGET_ELEMENT","name_contains":""},
  "forbidden_same_observation_entities": [{"label":"MaterialBatch","name_equals":"MATERIAL_NAME","name_contains":""}],
  "co_observed_entity": {"label":"","name_equals":"","name_contains":""},
  "spatial_logic": {"radius_m":500,"use_looked_side_sector":true,"half_angle_deg":90,"project_epsg":32756},
  "element_matching": {"target_name_contains":"ELEMENT_KEYWORD","target_name_equals":"","target_name_also_contains":"","filter_mode":"NAME_ONLY_CASE_INSENSITIVE"},
  "activity_link_logic": {"support_both_component_directions":true},
  "writeback_logic": {"inference_id_prefix":"IR:ELEM_COMPL","create_rule_justification_link":true,"create_temporal_link":true},
  "justification_text_template": "target element seen without material",
  "assumptions": []
}

[PATTERN 5] co_observation_two_elements + spatial + dual_element_filter
rule: "infer on progress when same observation sees two different element types
       together, within 500m of related elements"
grounding: {
  "rule_family": "observation_spatial_status",
  "rule_name": "co_presence_progress",
  "status_value": "on progress",
  "observation_trigger_type": "co_observation_two_elements",
  "primary_observed_entity": {"label":"InfrastructureElement","name_equals":"","name_contains":"ELEMENT_TYPE_A"},
  "forbidden_same_observation_entities": [],
  "co_observed_entity": {"label":"InfrastructureElement","name_equals":"","name_contains":"ELEMENT_TYPE_B"},
  "spatial_logic": {"radius_m":500,"use_looked_side_sector":true,"half_angle_deg":90,"project_epsg":32756},
  "element_matching": {"target_name_contains":"TARGET_KEYWORD_1","target_name_equals":"","target_name_also_contains":"TARGET_KEYWORD_2","filter_mode":"NAME_ONLY_CASE_INSENSITIVE"},
  "activity_link_logic": {"support_both_component_directions":true},
  "writeback_logic": {"inference_id_prefix":"IR:CO_EL","create_rule_justification_link":true,"create_temporal_link":true},
  "justification_text_template": "two element types co-observed",
  "assumptions": []
}

[PATTERN 6] element_present + absent_element + spatial
rule: "infer completed when element type A is seen AND element type B is NOT
       seen in same observation, within 500m of target elements"
grounding: {
  "rule_family": "observation_spatial_status",
  "rule_name": "element_absent_completion",
  "status_value": "completed",
  "observation_trigger_type": "element_present_element_absent",
  "primary_observed_entity": {"label":"InfrastructureElement","name_equals":"","name_contains":"ELEMENT_TYPE_A"},
  "forbidden_same_observation_entities": [{"label":"InfrastructureElement","name_equals":"","name_contains":"ELEMENT_TYPE_B"}],
  "co_observed_entity": {"label":"","name_equals":"","name_contains":""},
  "spatial_logic": {"radius_m":500,"use_looked_side_sector":true,"half_angle_deg":90,"project_epsg":32756},
  "element_matching": {"target_name_contains":"TARGET_KEYWORD_1","target_name_equals":"","target_name_also_contains":"TARGET_KEYWORD_2","filter_mode":"NAME_ONLY_CASE_INSENSITIVE"},
  "activity_link_logic": {"support_both_component_directions":true},
  "writeback_logic": {"inference_id_prefix":"IR:ABSENT_COMPL","create_rule_justification_link":true,"create_temporal_link":true},
  "justification_text_template": "element A seen without element B",
  "assumptions": []
}

[PATTERN 7] co_observation_element_and_material + spatial + element_filter
rule: "infer on progress when same observation sees a specific element type
       AND a specific material together, within 500m of target elements"
grounding: {
  "rule_family": "observation_spatial_status",
  "rule_name": "element_material_co_presence_progress",
  "status_value": "on progress",
  "observation_trigger_type": "co_observation_element_and_material",
  "primary_observed_entity": {"label":"InfrastructureElement","name_equals":"","name_contains":"ELEMENT_KEYWORD"},
  "forbidden_same_observation_entities": [],
  "co_observed_entity": {"label":"MaterialBatch","name_equals":"MATERIAL_NAME","name_contains":""},
  "spatial_logic": {"radius_m":500,"use_looked_side_sector":true,"half_angle_deg":90,"project_epsg":32756},
  "element_matching": {"target_name_contains":"TARGET_KEYWORD","target_name_equals":"","target_name_also_contains":"","filter_mode":"NAME_ONLY_CASE_INSENSITIVE"},
  "activity_link_logic": {"support_both_component_directions":true},
  "writeback_logic": {"inference_id_prefix":"IR:EL_MAT","create_rule_justification_link":true,"create_temporal_link":true},
  "justification_text_template": "element type and material co-observed",
  "assumptions": []
}

[PATTERN 8] schedule_assessment — no observation trigger, no spatial
rule: "assess status of all Activity nodes against latest observation time,
       checking planned start/end dates and predecessor delays"
grounding: {
  "rule_family": "schedule_assessment",
  "rule_name": "activity_schedule_status",
  "status_value": "varied",
  "observation_trigger_type": "latest_assessment",
  "primary_observed_entity": {"label":"","name_equals":"","name_contains":""},
  "forbidden_same_observation_entities": [],
  "co_observed_entity": {"label":"","name_equals":"","name_contains":""},
  "spatial_logic": {"radius_m":0,"use_looked_side_sector":false,"half_angle_deg":0,"project_epsg":32756},
  "element_matching": {"target_name_contains":"","target_name_equals":"","target_name_also_contains":"","filter_mode":"NAME_ONLY_CASE_INSENSITIVE"},
  "activity_link_logic": {"support_both_component_directions":false},
  "writeback_logic": {"inference_id_prefix":"IR:SCHED","create_rule_justification_link":true,"create_temporal_link":true},
  "justification_text_template": "schedule assessment at latest observation time",
  "assumptions": ["requires start time, finish time, and dependency relationships on Activity nodes"]
}

[PATTERN 9] cross_entity_join — no spatial, no observation loop
rule: "for each observed entity at a time point, find linked activities and
       check planned usage or assignment relationships"
grounding: {
  "rule_family": "cross_entity_join",
  "rule_name": "entity_usage_plan_check",
  "status_value": "inferred",
  "observation_trigger_type": "cross_entity_join",
  "primary_observed_entity": {"label":"Machinery","name_equals":"","name_contains":""},
  "forbidden_same_observation_entities": [],
  "co_observed_entity": {"label":"","name_equals":"","name_contains":""},
  "spatial_logic": {"radius_m":0,"use_looked_side_sector":false,"half_angle_deg":0,"project_epsg":32756},
  "element_matching": {"target_name_contains":"","target_name_equals":"","target_name_also_contains":"","filter_mode":"NAME_ONLY_CASE_INSENSITIVE"},
  "activity_link_logic": {"support_both_component_directions":false},
  "writeback_logic": {"inference_id_prefix":"IR:USAGE","create_rule_justification_link":true,"create_temporal_link":true},
  "justification_text_template": "entity usage plan verification",
  "assumptions": ["requires a usage/assignment relationship between Activity and the observed entity type"]
}


INSTRUCTIONS:
  - Use the logic_decomposition to identify the rule pattern
  - Match it to the closest example above, then adapt to this specific rule
  - Use entity_resolution (item 5) to fill primary_observed_entity and
    forbidden_same_observation_entities with exact resolved_name values
  - Use condition_structure.trigger_classification (item 6) as a strong hint
    for observation_trigger_type — it was computed from explicit entity analysis
  - If condition_structure.forbidden_in_same_observation is non-empty,
    the trigger type MUST be one of the absent variants
  - For observation_trigger_type, choose the value that best matches:
      single_entity_positive + Machinery     → machine_present
      single_entity_positive + IE            → element_present
      single_entity_positive + MaterialBatch → material_present
      single_entity_negative + IE absent Machinery  → element_present_machine_absent
      single_entity_negative + IE absent Material   → element_present_material_absent
      single_entity_negative + IE absent IE         → element_present_element_absent
      co_observation_same_event + IE + IE            → co_observation_two_elements
      co_observation_same_event + IE + MaterialBatch → co_observation_element_and_material
      no_observation_trigger                         → latest_assessment
      cross_entity_join                              → cross_entity_join
  - Preserve ALL negative conditions from the rule
  - Use target_name_also_contains when element matching needs two substrings

Return JSON only — a single grounded rule specification object with exactly
the same keys as shown in the examples above.
```
