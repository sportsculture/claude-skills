Find the next available DAG node(s) to work on based on dependencies.

**Arguments:** None (uses auto-detected DAG)

## Steps

1. **Get current DAG state**
   ```javascript
   dag_get({name: "[dag_name]"})
   ```

2. **Find ready nodes**
   - Filter nodes with status "pending"
   - Check dependencies are satisfied (all parents completed)
   - Sort by priority or level in DAG

3. **Analyze parallelization**
   ```javascript
   dag_analyze_levels({name: "[dag_name]"})
   ```
   - Identify if multiple nodes can be worked simultaneously
   - Group by level and independence

4. **Rank recommendations**
   - Critical path nodes (highest priority)
   - Nodes with many dependents (unlocks more work)
   - Nodes with no dependents (can be deferred)

5. **Display options**
   - Primary recommendation with reasoning
   - Alternative options if parallelizing
   - Show what each choice unblocks

## Example Interaction

```
User: /dag/next-available

Claude: Next Available Nodes for "ecommerce_plan"
================================================

✨ PRIMARY RECOMMENDATION: payment_integration
Reason: On critical path, blocks 2 downstream nodes
Dependencies: ✓ All satisfied (shopping_cart completed)
Complexity: Medium
Estimated effort: 2-3 days

Alternative Options (can work in parallel):
------------------------------------------

Option A: payment_integration + order_management_prep
- Parallel work opportunity
- payment_integration: Critical path
- order_management_prep: Preparatory work (documentation, schema design)
- Requires: 2 developers or git worktrees

Option B: payment_integration only
- Single focus approach
- Unlocks order_management after completion
- Recommended for solo development

🚫 BLOCKED NODES:
- order_management: waiting for payment_integration
- deployment_prep: waiting for order_management

📊 IMPACT ANALYSIS:
Starting payment_integration will:
→ Unblock order_management (next in sequence)
→ Progress critical path by 25%
→ Enable deployment_prep planning

🎯 RECOMMENDATION:
Start with: /dag/start-node payment_integration

Integration Points:
- Task Master: task-master show 4.1
- Git branch: git checkout -b feat/payment_integration
- Dependencies: shopping_cart API endpoints
```
