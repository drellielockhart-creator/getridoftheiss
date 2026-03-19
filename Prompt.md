# First Prompt for Coding Agent

Read both of the files I’m providing:

1. `iss_mission_control_agents.md` — Agent role descriptions, scenario briefing, and deliverable structure
1. `mission_control_implementation_spec.md` — Full implementation spec

Your job is to build this project incrementally. Do NOT try to build everything at once.

**Step 1 — Scaffolding and smoke test:**

1. Create the project directory structure as described in the spec under `~/mission_control/`
1. Install dependencies: `pip install "ag2[openai]" litellm`
1. Create `config.py` with the model assignments and simulation parameters. Leave `OPENROUTER_API_KEY = None` — I will provide the key when we’re ready to run. Use `os.environ.get("OPENROUTER_API_KEY")` as a fallback so I can set it either way.
1. Create the `prompts/` directory and port all six agent system prompts from the agents file into individual Python files, each exporting a string. Include the shared scenario briefing as `scenario_briefing.py` and compose each agent’s full system message as: scenario briefing + role description + communication rules (as described in the spec).
1. Create a minimal `smoke_test.py` that:
- Imports the config and one agent prompt (Gene)
- Creates a single AG2 AssistantAgent with Gene’s system prompt pointed at OpenRouter
- Sends one message (“Gene, confirm you are online and in character. State your name, role, and current situation assessment in under 100 words.”)
- Prints the response
- This is just to verify the AG2 → LiteLLM → OpenRouter pipeline works

Do NOT build the GroupChat, speaker selection, shared state, or simulation loop yet. Just the scaffolding and smoke test. We’ll verify the pipeline works before building the rest.

When you’re done with this step, show me the file tree and the contents of `smoke_test.py` so I can review before we proceed.
