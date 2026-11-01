# Reference review

These are formatting examples, not reusable findings. The exact published bodies are preserved below. The review body observed when creating this skill was `🤖 GPT-6.1-Sol`; use the actual reviewing model's identity for future reviews.

## Standards comment

`modules/apps/ant_hill_forager/main.tf`, new line 35:

```text
[P3] Remove the redundant lifecycle commentary

Standards: CLAUDE.md (Documentation) and docs/guides/documentation.md prohibit comments that restate code or narrate a change. The revised wording, 'the worker consumes here' and 'stays only until it is decommissioned', repeats the environment assignment and rollout narrative. Remove that wording; retain a comment only for a concrete constraint a reader needs to change this safely.
```

## Spec comment

`modules/apps/ant_hill_forager/main.tf`, new line 69:

```text
[P1] Include the application deployment in the cutover instructions

Spec: the PR promises that after merge the FIFO backlog goes down. This changes the registered task definition, but modules/service/ecs_service/main.tf ignores changes to the service's task_definition. Both Spacelift plans confirm that only the task definition is replaced, with no service update. Infrastructure apply therefore leaves the running workers polling the standard queue, so events sent to FIFO remain unprocessed until the application deploy updates the service. Add an explicit ant-hill-forager application deployment after Spacelift apply for staging and production, following README.md's Updating Task Definitions workflow.
```
