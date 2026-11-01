# Agent instructions

## Skill descriptions

Start every skill's `description` with `Trigger this when the user asks`, then name each distinct request that should load the skill, one clause per request. When running a command should load it too, end with `, or before you run <command>.` (`commit-message` and `pr-description` show both forms.)

The agent reads the description to decide when to load the skill, so every skill stays model-invoked: leave out `disable-model-invocation`.

## Sub-agents

Every skill that dispatches a sub-agent says to start it with a fresh context window. Some harnesses fork the parent conversation, and a fork inherits the context the sub-agent exists to escape.

## Skills calling skills

A step that runs another skill says `call the Skill tool with <name>`, and names only model-invoked skills. Reach another skill's material through the Skill tool, never through a path into its folder: install paths differ between harnesses.

## Harnesses

Skills run in Claude Code and Codex. Write steps both can follow: name a command for the agent to run in place of `!` context injection, and name a capability ("the harness's cross-provider delegation") in place of one harness's tool.

## Skill folders

Every skill has `agents/openai.yaml` with `interface.display_name` and `interface.short_description` for the Codex skill picker. The README lists every skill in its group, and the "How they fit" flow shows every skill `ship` runs.
