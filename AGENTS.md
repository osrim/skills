# Agent instructions

## Skill descriptions

Start every skill's `description` with `Trigger this when the user asks`, then name each distinct request that should load the skill, one clause per request. When running a command should load it too, end with `, or before you run <command>.` (`commit-message` and `pr-description` show both forms.)

The agent reads the description to decide when to load the skill, so every skill stays model-invoked: leave out `disable-model-invocation`.

## Sub-agents

Every skill that dispatches a sub-agent says to start it with a fresh context window. Some harnesses fork the parent conversation, and a fork inherits the context the sub-agent exists to escape.
