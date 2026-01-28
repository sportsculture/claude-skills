---
description: Generate a comprehensive sprint review checklist from git history
---

# Generate Sprint Review Checklist

Generate a comprehensive review checklist for the current sprint by analyzing:
1. Git commits from the last sprint
2. Changed files and their impact
3. Test coverage changes
4. Any architecture modifications

## Usage

```
/sprint-review
```

## Steps:

1. **Gather Sprint Data**
   ```bash
   # Get commits from last 7 days (adjust as needed)
   git log --oneline --since="7 days ago" --pretty=format:"%h %s" > sprint-commits.txt

   # Get changed files
   git diff --name-only HEAD~20..HEAD > changed-files.txt

   # Get test results if available
   npm test -- --coverage --json > test-results.json 2>/dev/null || echo "No test results"
   ```

2. **Analyze Changes**
   - Review each commit message for:
     - Root cause fixes vs symptom patches
     - Interface changes
     - New dependencies added
     - Architecture modifications

3. **Generate Checklist**
   Create a markdown checklist with these sections:

   ### 🎯 Root Cause Analysis
   - [ ] Were all fixes addressing root causes?
   - [ ] List any band-aid fixes that need revisiting
   - [ ] Document any deferred root cause fixes

   ### 🔒 Interface Stability
   - [ ] Did any public interfaces change?
   - [ ] Were all changes backward compatible?
   - [ ] List any breaking changes

   ### 🧪 Test Coverage
   - [ ] Did test coverage increase/decrease?
   - [ ] Were tests added for all bug fixes?
   - [ ] Are there any untested edge cases?

   ### 🏗️ Architecture Health
   - [ ] Were architectural boundaries respected?
   - [ ] Any new technical debt introduced?
   - [ ] Circular dependencies added?

   ### 📚 Learnings
   - [ ] What patterns worked well?
   - [ ] What caused delays or rework?
   - [ ] Improvements for next sprint?

4. **Save Results**
   - Save checklist to `.claude/reviews/sprint-[DATE].md`
   - Update `.claude/learnings.md` with key insights

5. **Action Items**
   - Create Taskmaster tasks for any identified issues
   - Update prompt templates based on learnings
