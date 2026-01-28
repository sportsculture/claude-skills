---
description: Query the GridPlay DAG system to understand code dependencies and cross-workflow impacts
---

# DAG Consultation Command

Query the GridPlay DAG system to understand code dependencies and cross-workflow impacts.

## Usage

```
/dag-consult
```

When planning changes or analyzing impacts, use this command to:

1. **Identify affected workflows** based on code area
2. **Trace cross-DAG dependencies** to understand cascade effects
3. **Get specific node information** from the DAG playground

## Steps

### 1. Understand what you're changing

Ask yourself:
- What domain area am I modifying? (Tournament, Roster, Player, Approvals)
- Which API endpoints or database tables are involved?
- What state transitions are affected?

### 2. Query relevant DAGs

Use MCP tools to get DAG structure:

**List all DAGs:**
```
Use mcp__dag-playground-http__dag_list
```

**Get specific DAG:**
```
Use mcp__dag-playground-http__dag_get with name="gridplay-tournament"
Use mcp__dag-playground-http__dag_get with name="gridplay-roster"
Use mcp__dag-playground-http__dag_get with name="gridplay-player-onboarding"
Use mcp__dag-playground-http__dag_get with name="gridplay-approvals"
```

**Trace node dependencies:**
```
Use mcp__dag-playground-http__dag_topology_ancestors with name="gridplay-tournament" node_id="rosters_locked"
Use mcp__dag-playground-http__dag_topology_descendants with name="gridplay-tournament" node_id="recruiting_open"
```

### 3. Check cross-DAG impacts

**Key Cross-DAG Relationships:**

| Change Area | Primary DAG | Affects DAGs | Critical Nodes |
|-------------|-------------|--------------|----------------|
| Tournament State | `gridplay-tournament` | roster, player-onboarding | `rosters_locked`, `recruiting_closed` |
| Recruiting State | `gridplay-tournament` | roster, player-onboarding | ALL recruiting state nodes |
| Roster Changes | `gridplay-roster` | player-onboarding, approvals | `roster_changes_allowed`, `player_added` |
| Player Joins | `gridplay-player-onboarding` | roster | `player_joined_team`, `invitation_accepted` |
| Approvals | `gridplay-approvals` | roster | `roster_change_approved`, `dob_change_approved` |

### 4. Validate your plan

Before implementing:
- ✅ Identified all affected DAG nodes?
- ✅ Checked ancestors (what must happen first)?
- ✅ Checked descendants (what happens after)?
- ✅ Considered cross-DAG cascade effects?
- ✅ Planned tests for each affected workflow?

## Common Patterns

### Pattern 1: State Transition Changes
1. Identify state node in primary DAG
2. Check all descendant nodes
3. Query related DAGs for decision nodes that check this state
4. Test each dependent workflow in old and new states

### Pattern 2: New Feature Addition
1. Identify which DAG(s) the feature belongs to
2. Add new nodes to appropriate DAG
3. Connect edges to show dependencies
4. Check for conflicts with existing nodes
5. Document cross-DAG impacts in node metadata

### Pattern 3: Bug Fix
1. Identify which DAG node represents the buggy behavior
2. Trace ancestors to find root cause
3. Trace descendants to find impact
4. Fix the issue without breaking dependent nodes
5. Verify fix doesn't cascade to other DAGs

## Requirements

- DAG playground MCP server running
- GridPlay DAGs defined in the system
