Update metadata for a DAG node (status, notes, context links).

**Arguments:** $ARGUMENTS (node_id and optional fields to update)

## Steps

1. **Get current node state**
   ```javascript
   dag_node_get({
     name: "[dag_name]",
     node_id: "[node_id]"
   })
   ```

2. **Ask for updates** (if not provided in arguments)
   - Status change? (pending → in_progress → completed)
   - Add notes?
   - Update summary?
   - Link context? (files, commits, PRs, Task Master IDs)

3. **Update via MCP** (if using HTTP transport)
   ```bash
   curl -X POST http://localhost:4001/api/status/[node_id] \
     -H "Content-Type: application/json" \
     -d '{
       "status": "in_progress",
       "activity": "updating_metadata",
       "details": "Added context links",
       "notes": "...",
       "context_links": ["file://src/auth.js"]
     }'
   ```

4. **Alternative: Manual metadata update**
   - For direct DAG modifications
   - Use if HTTP API unavailable

5. **Record in session log**
   - Append update to .claude/session_[timestamp].log
   - Include what changed and why

6. **Display confirmation**
   - Show before/after metadata
   - Confirm updates applied

## Example Interaction

```
User: /dag/update-node shopping_cart

Claude: Current metadata for "shopping_cart":
- Status: in_progress
- Owner: developer
- Summary: Implement shopping cart functionality
- Git branch: feat/shopping_cart
- Commit hash:
- Task Master ID: 3.2
- Context links: []
- Notes: Started implementation

What would you like to update?
1. Status
2. Add notes
3. Add context links
4. Update summary
5. All of the above

User: 2

Claude: Enter notes to add:

User: Completed cart item CRUD operations. Next: persist to localStorage.

Claude: Updating node "shopping_cart"...

✓ Added notes: "Completed cart item CRUD operations. Next: persist to localStorage."
✓ Recorded in session log: .claude/session_2025-10-30_14-32.log

Updated metadata:
- Status: in_progress (unchanged)
- Notes: "Completed cart item CRUD operations. Next: persist to localStorage."
- Context links: ["file://src/components/ShoppingCart.jsx"]

Node updated successfully!
```

## Quick Syntax

```bash
# Status update only
/dag/update-node shopping_cart --status=completed

# Add notes
/dag/update-node shopping_cart --notes="Implemented localStorage persistence"

# Add context link
/dag/update-node shopping_cart --link="file://src/cart.js"

# Multiple updates
/dag/update-node shopping_cart --status=completed --notes="All tests passing" --commit=abc123f
```
