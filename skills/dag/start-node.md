Begin working on a specific DAG node.

**Arguments:** $ARGUMENTS (node_id, e.g., "user_auth")

## Steps

1. **Validate node exists**
   - Query current DAG to verify node_id exists
   - Show node metadata (status, summary, dependencies)

2. **Check dependencies**
   ```javascript
   dag_topology_ancestors({
     name: "[dag_name]",
     node_id: "[node_id]"
   })
   ```
   - Verify all parent nodes are completed
   - Warn if dependencies are not satisfied

3. **Create git branch**
   ```bash
   git checkout -b feat/[node_id]
   ```

4. **Set context**
   ```bash
   echo "[node_id]" > .claude_node
   ```

5. **Update node status**
   - Use MCP tool or manual update:
   ```bash
   curl -X POST http://localhost:4001/api/status/[node_id] \
     -H "Content-Type: application/json" \
     -d '{"status":"in_progress","activity":"starting_node"}'
   ```

6. **Show Task Master tasks** (if integrated)
   ```bash
   task-master show [taskmaster_id]
   ```

7. **Display context**
   - Current node: [node_id]
   - Status: in_progress
   - Dependencies: [list]
   - Git branch: feat/[node_id]
   - Next actions: [from summary/notes]

## Example Interaction

```
User: /dag/start-node user_auth

Claude: Starting work on node "user_auth"...

Dependencies check:
✓ bootstrap_setup (completed)

Creating git branch: feat/user_auth
Setting .claude_node to: user_auth
Updating node status to: in_progress

Current Context:
- Node: user_auth
- Summary: Implement JWT-based authentication system
- Dependencies: bootstrap_setup
- Git branch: feat/user_auth
- Task Master: task-master show 2.1

Next steps:
1. Review authentication requirements
2. Set up JWT library
3. Implement auth middleware
```
