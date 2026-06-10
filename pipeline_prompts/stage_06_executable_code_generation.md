# Stage 6 – Executable Code Generation

## Prompt used in the implementation

```text
You are an expert Neo4j developer generating executable Python/Cypher code for a
construction project knowledge-graph reasoning engine.

You will receive:
  1. grounded_rule_spec    — the structured JSON specification of the rule
  2. schema_grounded_rule  — the rule text with exact schema names
  3. logic_decomposition   — abstract computational structure
  4. schema_snapshot       — live Neo4j schema
  5. graph_mapping_profile — canonical name mappings
  6. condition_structure   — (supplementary) REQUIRED and FORBIDDEN conditions

Generate complete, runnable Python code that implements the rule.

MANDATORY RULES:
0. READ condition_structure.forbidden_in_same_observation FIRST.
   Each entity listed there requires an explicit AND NOT clause in CYP_GET_OBS.
   If trigger_classification contains "absent", CYP_GET_OBS MUST include:
   AND NOT (o)-[:observes]->(:Label {name:$blocked_name}) for each forbidden entity.
   Use the ACTUAL label and name from forbidden_in_same_observation — not example values.
1. NEO4J_URI, NEO4J_USER, NEO4J_PASSWORD are pre-existing globals — NEVER hardcode credentials
2. Use MERGE not CREATE for InformationResource nodes (idempotent)
3. Cypher relationships MUST use -[:REL]-> syntax. NEVER use )->[:
4. For the activity query, check BOTH directions of hasComponent:
   MATCH (ca:ConstructionActivity)-[:hasComponent]->(e:...) ... UNION ...
   MATCH (e:...)-[:hasComponent]->(ca:ConstructionActivity) ...
5. Always include a summary print block at the end
6. Code must run inside exec() — no if __name__ == "__main__" guards
7. Do NOT wrap output in exec(). Write plain Python.
8. Python syntax must be valid — verify all triple-quoted strings are closed
9. All imports from: neo4j, pyproj, numpy, pandas, datetime, logging

REFERENCE CODE EXAMPLES:

EXAMPLE 1 — machine_present trigger, spatial, single-needle element filter:
Rule: "infer on progress when a specific machine type is observed within 500m
       of target infrastructure elements"
NOTE: Replace MACHINE_TYPE_NAME, FILTER_KEYWORD, IR_PREFIX, RULE_ID_VALUE
      with the actual values from grounded_rule_spec.

```python
from neo4j import GraphDatabase
from pyproj import Transformer, CRS
import numpy as np
import pandas as pd

RULE_ID = "machine_type_progress"          # ← from grounded_rule_spec.rule_name
MACHINE_NAME = "MACHINE_TYPE_NAME"         # ← from primary_observed_entity.name_equals
RADIUS_M = 500.0                           # ← from spatial_logic.radius_m
HALF_ANGLE = 90.0                          # ← from spatial_logic.half_angle_deg
SRC_TEMPLATE = "inference: machine observed within {R}m and sector '{SIDE}'"
FILTER_LIT = "ELEMENT_KEYWORD"             # ← from element_matching.target_name_contains
PROJECT_EPSG = 32756                       # ← from spatial_logic.project_epsg
FILTER_MODE = "NAME_ONLY_CASE_INSENSITIVE"

def side_to_deg(side):
    side_dir_map = {
        "N":0.0,"NORTH":0.0,"NE":45.0,"NORTHEAST":45.0,"NORTH-EAST":45.0,"NORTH EAST":45.0,
        "E":90.0,"EAST":90.0,"SE":135.0,"SOUTHEAST":135.0,"SOUTH-EAST":135.0,"SOUTH EAST":135.0,
        "S":180.0,"SOUTH":180.0,"SW":225.0,"SOUTHWEST":225.0,"SOUTH-WEST":225.0,"SOUTH WEST":225.0,
        "W":270.0,"WEST":270.0,"NW":315.0,"NORTHWEST":315.0,"NORTH-WEST":315.0,"NORTH WEST":315.0
    }
    if not side: return None
    return side_dir_map.get(str(side).strip().upper(), None)

def ll_to_xy(lon, lat, epsg=32756):
    if lon is None or lat is None: return None, None
    t = Transformer.from_crs(CRS.from_epsg(4326), CRS.from_epsg(epsg), always_xy=True)
    x, y = t.transform(float(lon), float(lat))
    return float(x), float(y)

driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))

CYP_GET_OBS = """
MATCH (o:Observation)-[:observes]->(m:Machinery)
WHERE m.name = $machine_name
OPTIONAL MATCH (o)-[:observedAt]->(t:TemporalInstant)
RETURN o.id AS oid, o.looked_side AS looked_side, o.lat AS lat, o.lon AS lon,
       coalesce(t.isoDate, o.time_iso) AS isoDate
"""

CYP_GET_ELEMENTS = """
MATCH (e:InfrastructureElement)
WHERE e.name IS NOT NULL AND toUpper(e.name) CONTAINS toUpper($needle)
  AND e.globalX IS NOT NULL AND e.globalY IS NOT NULL
RETURN e.guid AS guid, e.name AS name, e.globalX AS x, e.globalY AS y
"""

CYP_GET_ACTIVITIES = """
MATCH (ca:ConstructionActivity)-[:hasComponent]->(e:InfrastructureElement {guid:$guid})
RETURN DISTINCT ca.id AS actID, ca.name AS actName
UNION
MATCH (e:InfrastructureElement {guid:$guid})-[:hasComponent]->(ca:ConstructionActivity)
RETURN DISTINCT ca.id AS actID, ca.name AS actName
"""

CYP_WRITE_IR = """
MATCH (ca:ConstructionActivity {id:$actID})
MERGE (ir:InformationResource {id:$ir_id})
  ON CREATE SET ir.status = 'on progress', ir.source = $source
  ON MATCH  SET ir.status = 'on progress', ir.source = $source
MERGE (ca)-[:hasInformation]->(ir)
MERGE (rr:icop_SWRRule {id:$rid})
MERGE (ir)-[:justifiedBy]->(rr)
FOREACH (_ IN CASE WHEN $iso IS NOT NULL THEN [1] ELSE [] END |
  MERGE (t:TemporalInstant {isoDate:$iso})
  MERGE (ir)-[:inferedAt]->(t)
)
RETURN ir.id AS irid
"""

with driver.session() as dsess:
    obs_rows = dsess.run(CYP_GET_OBS, machine_name=MACHINE_NAME).data()
    el_rows  = dsess.run(CYP_GET_ELEMENTS, needle=FILTER_LIT).data()
    elements = pd.DataFrame(el_rows) if el_rows else pd.DataFrame(columns=["guid","name","x","y"])
    if not elements.empty:
        elements["x"] = pd.to_numeric(elements["x"], errors="coerce")
        elements["y"] = pd.to_numeric(elements["y"], errors="coerce")
        elements = elements.dropna(subset=["x","y"])
    total_ir=0; matched_obs=0; matched_cand=0; matched_acts=0
    if not obs_rows: print("No matching observations for machine:", MACHINE_NAME)
    if elements.empty: print("No elements found containing:", FILTER_LIT)
    for o in obs_rows:
        oid=o["oid"]; side=o["looked_side"]; iso=o["isoDate"]; lat=o["lat"]; lon=o["lon"]
        if lat is None or lon is None or elements.empty: continue
        ox, oy = ll_to_xy(lon, lat, PROJECT_EPSG)
        if ox is None: continue
        matched_obs += 1; center = side_to_deg(side)
        dx=elements["x"].values-ox; dy=elements["y"].values-oy
        dist=np.hypot(dx,dy); bearing=(np.degrees(np.arctan2(dx,dy))+360.0)%360.0
        in_radius=dist<=RADIUS_M
        if center is None: in_fov=np.ones_like(dist,dtype=bool)
        else:
            diff=np.abs((bearing-center+180.0)%360.0-180.0); in_fov=diff<=HALF_ANGLE
        candidates=elements[in_radius&in_fov]
        if candidates.empty: continue
        matched_cand += len(candidates)
        for _,e in candidates.iterrows():
            acts=dsess.run(CYP_GET_ACTIVITIES, guid=e["guid"]).data()
            for a in acts:
                actID=a["actID"]
                if not actID: continue
                matched_acts += 1
                ir_id=f"IR:GEN:{oid}:{actID}"
                source=SRC_TEMPLATE.replace("{R}",str(int(RADIUS_M))).replace("{SIDE}",str(side) if side else "ALL")
                res=dsess.run(CYP_WRITE_IR,rid=RULE_ID,ir_id=ir_id,actID=actID,source=source,iso=iso).single()
                if res and res.get("irid"): total_ir+=1
print(f"Rule: {RULE_ID} | obs: {len(obs_rows)} | loc: {matched_obs} | cand: {matched_cand} | acts: {matched_acts} | IR: {total_ir}")
driver.close()
```


EXAMPLE 2 — element_present_machine_absent trigger (negative condition in CYP_GET_OBS):
Rule: "infer completed when a target element IS seen AND a blocking machine is NOT seen"
NOTE: Replace TARGET_ELEMENT_NAME, BLOCKED_MACHINE_NAME, FILTER_KEYWORD, RULE_ID_VALUE
      with actual values from grounded_rule_spec.

Key difference from Example 1: the observation query uses AND NOT pattern.

```python
# ... same imports and helpers as Example 1 ...

RULE_ID = "element_completion_no_machine"
TARGET_IE_NAME = "TARGET_ELEMENT_NAME"
BLOCKED_MACH_NAME = "BLOCKING_MACHINE"

CYP_GET_OBS = """
MATCH (o:Observation)-[:observes]->(ie:InfrastructureElement)
WHERE ie.name IS NOT NULL
  AND toUpper(ie.name) = toUpper($target_ie_name)
  AND NOT (o)-[:observes]->(:Machinery {name:$blocked_mach_name})
OPTIONAL MATCH (o)-[:observedAt]->(t:TemporalInstant)
RETURN o.id AS oid, o.looked_side AS looked_side, o.lat AS lat, o.lon AS lon,
       coalesce(t.isoDate, o.time_iso) AS isoDate
"""
```

For element_present_element_absent: use NOT EXISTS { MATCH (o)-[:observes]->(fe:InfrastructureElement) WHERE toUpper(fe.name) CONTAINS ... }
For element_present_material_absent: use AND NOT (o)-[:observes]->(:MaterialBatch {name:$blocked_name})
For co_observation_two_elements: use WITH DISTINCT o between the two MATCH clauses
For co_observation_element_and_material: use WHERE EXISTS{} AND EXISTS{} subquery pattern


EXAMPLE 3 — schedule_assessment (no observation trigger, no spatial, multi-status output):
Rule: "assess all ConstructionActivity nodes against latest observation time"

```python
from neo4j import GraphDatabase
import pandas as pd
from datetime import datetime, timezone

RULE_ID = "activity_status_assessment"
SRC_TEMPLATE = "latest activity status assessment snapshot at {T}"
RUN_TS = datetime.now(timezone.utc).isoformat()

driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))

CYP_GET_LAST_OBS_TIME = """
MATCH (o:Observation)-[:observedAt]->(t:TemporalInstant)
RETURN max(t.isoDate) AS lastIso
"""
CYP_GET_ACTIVITIES_WITH_TIMES = """
MATCH (ca:ConstructionActivity)
OPTIONAL MATCH (ca)-[:hasStartTime]->(ts:TemporalInstant)
OPTIONAL MATCH (ca)-[:hasFinishTime]->(te:TemporalInstant)
RETURN ca.id AS id, ca.name AS name, ts.isoDate AS startIso, te.isoDate AS endIso
"""
CYP_GET_COMPLETED_FLAGS = """
MATCH (ca:ConstructionActivity)-[:hasInformation]->(ir:InformationResource {status:'completed'})
WHERE ir.id STARTS WITH 'IR:' AND NOT ir.id STARTS WITH 'IR:ASSESS:'
OPTIONAL MATCH (ir)-[:inferedAt]->(t:TemporalInstant)
WHERE t.isoDate IS NULL OR t.isoDate <= $tlast
RETURN DISTINCT ca.id AS id
"""
CYP_WRITE_ASSESSMENT = """
MERGE (rr:icop_SWRRule {id:$rid}) WITH rr
MATCH (ca:ConstructionActivity {id:$actID})
MERGE (ir:InformationResource {id:$ir_id})
  ON CREATE SET ir.status=$status, ir.source=$source
  ON MATCH SET ir.status=$status, ir.source=$source
MERGE (ca)-[:hasInformation]->(ir) MERGE (ir)-[:justifiedBy]->(rr)
RETURN ir.id AS irid
"""
```


EXAMPLE 4 — cross_entity_join (single Cypher, no spatial, no loop):
Rule: "for each observed Machinery at a TemporalInstant, check if activity planned to use it"

```python
from neo4j import GraphDatabase
from datetime import datetime, timezone

RULE_ID = "machinery_usage_required_by_plan_check"
driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))
run_ts = datetime.now(timezone.utc).isoformat()

cypher = """
MATCH (obs:Observation)-[:observes]->(mach:Machinery)
MATCH (obs)-[:observedAt]->(ti:TemporalInstant)
MATCH (ca:ConstructionActivity)-[:hasInformation]->(:InformationResource)-[:inferedAt]->(ti)
OPTIONAL MATCH (ca)-[:usesMachinery]->(plan_mach:Machinery) WHERE plan_mach.name = mach.name
WITH DISTINCT obs, mach, ti, ca,
     CASE WHEN count(plan_mach) > 0 THEN true ELSE false END AS usage_required
MERGE (rr:icop_SWRRule {id:$rule_id})
MERGE (ir_new:InformationResource {id: 'IR:MACH_USAGE:' + obs.id + ':' + ca.id})
SET ir_new.status='inferred', ir_new.machinery_usage_required_by_plan=usage_required
MERGE (ca)-[:hasInformation]->(ir_new)
MERGE (ir_new)-[:inferedAt]->(ti)
MERGE (ir_new)-[:justifiedBy]->(rr)
RETURN count(DISTINCT ir_new) AS num_inferences
"""
```


Choose the most appropriate example as your base (based on observation_trigger_type),
then adapt it by substituting ALL placeholder values with actual values from grounded_rule_spec.
CRITICAL: Never use MACHINE_TYPE_NAME, TARGET_ELEMENT_NAME, ELEMENT_KEYWORD or any
other placeholder literally — replace every placeholder with the actual resolved value.

IMPORTANT: All variable values (entity names, IR prefix, rule id, status, etc.)
must come from grounded_rule_spec — never use hardcoded example values from
the code examples above. The examples show STRUCTURE, not actual values.

Return JSON only:
{ "generated_code": "<complete Python code as a plain string, no markdown fences>" }
```
