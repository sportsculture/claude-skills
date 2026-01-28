Mark a DAG node as completed with verification.

**Arguments:** $ARGUMENTS (node_id, optional commit message)

## Steps

1. **Verify completion criteria**
   - Check if all work is committed
   - Run relevant tests if applicable
   - Verify Task Master tasks are done (if integrated)

2. **Git commit and push**
   ```bash
   git add .
   git commit -m "feat: complete [node_id] - [summary]"
   git push -u origin feat/[node_id]
   ```

3. **Update node metadata**
   - Set status to "completed"
   - Record commit_hash
   - Add completion notes

4. **Clear context**
   ```bash
   rm .claude_node
   ```

5. **Find next node**
   ```javascript
   dag_topology_sort({name: "[dag_name]"})
   ```
   - Identify next available node(s)
   - Check if multiple nodes can run in parallel

6. **Display summary**
   - Completed node: [node_id]
   - Commit hash: [hash]
   - Downstream nodes unblocked: [list]
   - Next recommended node: [node_id]

## Example Interaction

```
User: /dag/complete-node user_auth

Claude: Verifying completion of "user_auth"...

✓ Git status: all changes committed
✓ Tests: passed (if run)
✓ Task Master: task 2.1 marked as done

Committing changes:
[abc123f] feat: complete user_auth - JWT authentication system

Updating node metadata:
- Status: completed
- Commit: abc123f
- Git branch: feat/user_auth

Next available nodes:
1. product_catalog (ready to start)
2. shopping_cart (blocked by product_catalog)

Recommendation: Start "product_catalog" next

Would you like to:
1. Start product_catalog now (/dag/start-node product_catalog)
2. View overall progress (/dag/show-progress)
3. Create pull request (gh pr create)
```
