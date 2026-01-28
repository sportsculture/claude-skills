Create a new work DAG for the current project using the evolution_plan pattern.

**Arguments:** $ARGUMENTS (DAG name, e.g., "my_project_plan")

## Steps

1. **Ask for DAG name** if not provided in arguments
   - Validate format: lowercase, underscores, ends with "_plan"
   - Example: `ecommerce_plan`, `api_refactor_plan`

2. **Ask for milestone list**
   - Prompt user to provide milestone phases:
     - Bootstrap (setup, schema, tooling)
     - Core features (main functionality)
     - Integrations (external systems)
     - Release prep (testing, docs, deployment)
   - Get milestone node IDs from user

3. **Create DAG**
   ```javascript
   dag_create({
     name: "[dag_name]",
     type: "workflow"
   })
   ```

4. **Add each milestone as event node**
   ```javascript
   dag_node_add({
     name: "[dag_name]",
     node: {
       type: "event",
       id: "[milestone_id]",
       metadata: {
         status: "pending",
         owner: "[current_user]",
         summary: "[milestone description]",
         git_branch: "",
         commit_hash: "",
         taskmaster_id: "",
         context_links: [],
         notes: ""
       }
     }
   })
   ```

5. **Ask about dependencies**
   - Ask user if milestones are sequential or have custom dependencies
   - If sequential, add causal edges between consecutive milestones
   - If custom, ask user to specify from→to relationships

6. **Add edges**
   ```javascript
   dag_edge_add({
     name: "[dag_name]",
     from: "[source_milestone]",
     to: "[target_milestone]",
     edge_type: "causal",
     weight: 1
   })
   ```

7. **Analyze parallelization**
   ```javascript
   dag_analyze_levels({name: "[dag_name]"})
   ```
   - Show level groups
   - Highlight parallel opportunities

8. **Set initial context**
   - Create `.claude_node` file with first milestone node
   - Update user on next steps

9. **Display summary**
   - DAG name
   - Number of nodes
   - Number of edges
   - Topological order
   - Parallel levels
   - Next recommended node to start

## Example Interaction

```
User: /dag/create-work-dag ecommerce_plan

Claude: I'll create a work DAG named "ecommerce_plan". Please provide the milestone phases as a comma-separated list:

User: bootstrap_setup, user_auth, product_catalog, shopping_cart, payment_integration, order_management, deployment_prep

Claude: Creating DAG with 7 milestones...

[Creates DAG and adds nodes]

Summary:
- DAG: ecommerce_plan (7 nodes, 6 sequential edges)
- Parallel opportunities: 3 milestones can run in parallel at level 2
- Next step: Start with bootstrap_setup
- TUI monitoring: Run `cargo run --release` in dag-playground/clients/coding-gps-tui

Your .claude_node file has been set to: bootstrap_setup
```