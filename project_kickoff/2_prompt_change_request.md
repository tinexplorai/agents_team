CHANGE REQUEST — re-run a subset of the agent team after QA / deploy.

I want to change something the team has already finished. Handle this per agent_team/workflow.md §2 step 9 (Change Request loop).

Process:
1. Ground yourself. Read in this order:
   - agent_team/workflow.md (§2 step 9 — the rules for this loop)
   - project_setup/step_1_project.md (project scope + constraints)
   - agent_team/task_board.md (current state — what phase the team last completed)
   - project_code/documentation/user_stories.md, project_code/documentation/design_spec.md, project_code/documentation/api_contract.md (existing agreements you must not silently break)
   - project_code/documentation/qa_report.md (last QA pass — this is the regression baseline)
   - project_code/documentation/deployment.md if it exists (signals the project has shipped → redeploy gate applies)

2. Classify the change as one of: **Small / Medium / Large-backend / Large-UX** per agent_team/workflow.md §2 step 9. Report back to me in chat with:
   - The classification + reasoning (why not the next tier up or down).
   - Which agents you plan to spawn, in which order.
   - Which existing docs will get appended sections and which files will likely change.
   - Any ambiguity in my request that would change the plan if clarified.

   Then STOP and wait for my approval. Do NOT spawn any agent yet.

3. After I approve, open a new section in task_board.md:

   ```
   ## Phase 9 — Change Request: <short title>
   ```

   Spawn the minimum set of agents per the classification. Pass these constraints to every agent you spawn:
   - "Read the existing doc (user_stories.md / design_spec.md / api_contract.md) and APPEND a new section for this change — do not rewrite earlier sections. Mark the section with the change request title so history is preserved."
   - "Update the new Phase 9 section in task_board.md when you finish; pass the baton to the next agent through the task board's message table, same as a normal phase."

4. QA Agent must run regression on **all flows that share code with the changed component**, not only the new behavior. If regression fails, fix the code and re-run before reporting PASS.

5. After QA passes, compile project_code/documentation/change_report_<short-title>.md covering:
   - What changed (one paragraph).
   - Which agents ran.
   - QA results — new tests + regression deltas vs. project_code/documentation/qa_report.md.
   - Files touched (paths only).

6. **Redeploy gate.** If project_code/documentation/deployment.md exists, STOP and ask: "Redeploy via DevOps Agent? Or fix something first?" Do NOT spawn DevOps without my explicit yes. If the project has not been deployed yet, just hand back to me with the change report.

My change request:
=== START ===
What:        [describe the change in plain language — one or two sentences]
Where:       [which screen / endpoint / file area — leave blank if you don't know, the agents will figure it out]
Why:         [user feedback / new requirement / bug discovered — context helps agents make judgment calls on edge cases]
Severity:    [blocker / nice-to-have]
Constraints: [must keep API backward-compatible / must ship before <date> / must not touch <area> / etc. — leave blank if none]
=== END ===
