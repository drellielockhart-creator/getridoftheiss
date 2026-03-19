# ISS Mission Control Simulation — Implementation Spec

## Overview

Build a multi-agent simulation where six AI agents role-play as a Mission Control team solving an impossible engineering crisis: removing the contaminated International Space Station from Earth orbit. The agents argue, calculate, and collaborate to produce a Mission Architecture Document.

This runs on a single Ubuntu VM. All LLM calls go through OpenRouter. The user will provide an OpenRouter API key when ready to run.

-----

## Architecture

```
┌─────────────────────────────────────────────┐
│                AG2 GroupChat                 │
│                                             │
│  ┌───────┐  ┌────────┐  ┌─────┐  ┌──────┐  │
│  │ GENE  │  │ STACKS │  │ KAI │  │ VOSS │  │
│  │Flight │  │Propul- │  │Struc│  │Logis-│  │
│  │  Dir  │  │ sion   │  │tural│  │tics  │  │
│  └───┬───┘  └────────┘  └─────┘  └──────┘  │
│      │                                      │
│  ┌───┴───┐  ┌────────┐                      │
│  │ CROSS │  │ ORACLE │ ← stronger model     │
│  │ Risk  │  │Referee │                      │
│  └───────┘  └────────┘                      │
│                                             │
│         GroupChatManager                    │
│     (custom speaker selection)              │
└──────────────┬──────────────────────────────┘
               │
        ┌──────┴──────┐
        │   LiteLLM   │
        │  (routing)   │
        └──────┬──────┘
               │
        ┌──────┴──────┐
        │ OpenRouter  │
        │   API       │
        └─────────────┘
```

-----

## Tech Stack

|Component      |Choice                                                    |Why                                                                                                               |
|---------------|----------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
|Agent framework|**AG2** (the community fork of AutoGen, `pip install ag2`)|GroupChat primitive maps directly to multi-agent debate; custom speaker selection gives us Flight Director routing|
|LLM routing    |**LiteLLM** (already a dependency of AG2)                 |Unified OpenAI-compatible interface to OpenRouter models                                                          |
|LLM provider   |**OpenRouter**                                            |Single API key, access to many cheap models + a stronger model for Oracle                                         |
|Logging        |File-based transcript + optional SQLite                   |Full conversation log for review                                                                                  |
|Output         |Markdown file (Mission Architecture Document)             |Final deliverable the agents collaboratively build                                                                |

### Model Assignments (via OpenRouter)

These are **defaults** — the user may swap models before running. Configure these as variables at the top of the script so they’re easy to change.

|Agent                         |Model                                                               |Rationale                             |
|------------------------------|--------------------------------------------------------------------|--------------------------------------|
|Gene, Stacks, Kai, Voss, Cross|`google/gemini-2.0-flash-001` or `meta-llama/llama-3.3-70b-instruct`|Cheap, fast, decent reasoning         |
|Oracle                        |`anthropic/claude-sonnet-4` or `openai/gpt-4o`                      |Needs reliable math; worth paying more|

The user will decide final model assignments. Make these configurable via a `config.py` or top-of-file constants.

-----

## Project Structure

```
mission_control/
├── config.py              # API key, model assignments, simulation parameters
├── agents.py              # Agent definitions (system prompts, model config)
├── speaker_selection.py   # Custom speaker selection logic
├── simulation.py          # Main simulation loop
├── shared_state.py        # Shared documents (risk register, validated params, etc.)
├── transcript_logger.py   # Logs all messages to file
├── run.py                 # Entry point
├── prompts/
│   ├── scenario_briefing.py   # Shared context injected into all agents
│   ├── gene.py
│   ├── stacks.py
│   ├── kai.py
│   ├── voss.py
│   ├── cross.py
│   └── oracle.py
├── output/
│   ├── transcript.md      # Full conversation log
│   └── mission_architecture.md  # Final deliverable (built during sim)
└── requirements.txt
```

-----

## Implementation Details

### 1. config.py

```python
# User provides this at runtime or sets as env var
OPENROUTER_API_KEY = None  # Set before running, or use env OPENROUTER_API_KEY

# Model assignments — change these to whatever's available/cheap on OpenRouter
TEAM_MODEL = "google/gemini-2.0-flash-001"  # Gene, Stacks, Kai, Voss, Cross
ORACLE_MODEL = "anthropic/claude-sonnet-4"  # Oracle (physics validator)

# Simulation parameters
MAX_ROUNDS = 80          # Total conversation turns before forced wrap-up
WRAP_UP_ROUND = 60       # Round at which Gene is told to start producing the final document
LOG_FILE = "output/transcript.md"
OUTPUT_FILE = "output/mission_architecture.md"
```

### 2. LiteLLM Configuration for OpenRouter

AG2 uses LiteLLM under the hood. Configure it to route through OpenRouter:

```python
import os

# OpenRouter uses the OpenAI-compatible endpoint
os.environ["OPENROUTER_API_KEY"] = config.OPENROUTER_API_KEY

# AG2 LLM config for team agents
team_llm_config = {
    "config_list": [
        {
            "model": config.TEAM_MODEL,
            "api_key": config.OPENROUTER_API_KEY,
            "base_url": "https://openrouter.ai/api/v1",
        }
    ],
    "temperature": 0.7,
    "cache_seed": None,  # Disable caching so agents don't repeat
}

# AG2 LLM config for Oracle (stronger model)
oracle_llm_config = {
    "config_list": [
        {
            "model": config.ORACLE_MODEL,
            "api_key": config.OPENROUTER_API_KEY,
            "base_url": "https://openrouter.ai/api/v1",
        }
    ],
    "temperature": 0.2,  # Oracle should be precise, less creative
    "cache_seed": None,
}
```

### 3. Agent Definitions

Each agent is an `ag2.AssistantAgent` with:

- A **name** (short, no spaces): `Gene`, `Stacks`, `Kai`, `Voss`, `Cross`, `Oracle`
- A **system_message** composed of: the shared scenario briefing + the agent-specific role description
- The appropriate **llm_config** (team or oracle)

The system prompts are in the companion file `iss_mission_control_agents.md`. Each agent’s system_message should be constructed as:

```
{scenario_briefing}

---

{agent_specific_role_description}

---

COMMUNICATION RULES:
- Address other agents by name when responding to or requesting from them.
- When you state a number (delta-v, mass, thrust, timeline), flag it for Oracle validation by writing: "ORACLE CHECK: [the specific claim]"
- When Gene assigns you an action item, acknowledge it and deliver within 1-2 turns.
- Stay in character. You are {name}, not a general-purpose AI assistant.
- Keep responses focused and under 400 words. Do not monologue.
- {agent_specific_communication_addendum}
```

Agent-specific communication addenda:

- **Gene**: “You decide who speaks next by ending your messages with ‘NEXT: [agent name]’ or ‘NEXT: OPEN’ for open floor. You track action items. When you reach round {WRAP_UP_ROUND}, begin drafting the Mission Architecture Document by assigning sections to agents.”
- **Stacks**: “Always show your math. Use the Tsiolkovsky equation explicitly.”
- **Kai**: “Always state confidence levels (high/medium/low) for structural assessments.”
- **Voss**: “Always reference specific launch vehicles and realistic timelines.”
- **Cross**: “Maintain a running Risk Register. Number each risk. Assign likelihood (1-5) and consequence (1-5) scores.”
- **Oracle**: “You ONLY respond when asked to validate a claim. Respond with CONFIRMED, CORRECTED (with correct calculation shown), or PLAUSIBLE BUT UNVERIFIABLE. Do not participate in strategy discussions. Do not volunteer information. If asked a strategic question, decline and restate what you can validate.”

### 4. Custom Speaker Selection

This is the most important piece. The `speaker_selection_method` in AG2 GroupChat can be a callable function. Implement one that approximates Gene’s coordination role:

```python
def select_next_speaker(last_speaker, groupchat):
    """
    Speaker selection logic:
    1. If the last message contains "ORACLE CHECK:", Oracle speaks next.
    2. If the last message contains "NEXT: [name]", that agent speaks next.
    3. If the last message contains "NEXT: OPEN", rotate to the agent who
       has spoken least recently (excluding Oracle).
    4. If Oracle just spoke, return to whoever invoked Oracle.
    5. Every 8-10 turns, force Gene to speak (to keep coordination on track).
    6. Fallback: Gene speaks.
    """
```

Key behaviors:

- Oracle **never** speaks unless explicitly invoked via “ORACLE CHECK:” or “NEXT: Oracle”
- Gene speaks at least every 8-10 turns to maintain coordination
- After Oracle validates (or corrects), control returns to the agent who requested validation
- Cross should be guaranteed at least one turn every 12-15 messages to prevent the team from ignoring risk
- In the wrap-up phase (after WRAP_UP_ROUND), Gene speaks every 3-4 turns to drive document assembly

### 5. Shared State

Maintain a simple dict (or small class) that tracks shared documents the agents reference and build:

```python
shared_state = {
    "risk_register": [],          # Cross maintains this
    "validated_params": [],       # Oracle confirmations accumulate here
    "action_items": [],           # Gene tracks these
    "mission_phase": "planning",  # planning → analysis → design → wrap_up
    "current_round": 0,
}
```

Inject a summary of this shared state into each agent’s context periodically (every 10 rounds or so) as a system message update, so agents don’t lose track of what’s been decided.

**Implementation note:** AG2 GroupChat maintains conversation history automatically. The shared state is *supplementary* — it gives agents a structured summary so they don’t have to re-read 50 messages to remember what’s been validated.

### 6. Simulation Flow

```
1. Initialize all agents with system prompts
2. Create GroupChat with all 6 agents + custom speaker selection
3. Create GroupChatManager
4. Inject the opening message (see below)
5. Run the GroupChat for MAX_ROUNDS
6. Log everything to transcript.md
7. Extract and save the Mission Architecture Document to output/
```

**Opening message** (sent as a user proxy message to kick off the simulation):

```
MISSION CONTROL — EMERGENCY SESSION INITIATED

Date: March 18, 2026
Classification: CRITICAL

The International Space Station has been contaminated. Full details are in your
briefing materials. All ISS crew have been evacuated via Crew Dragon.

Orbital decay gives us approximately 18-24 months before uncontrolled reentry
becomes inevitable without intervention.

The President has authorized unlimited budget and full international cooperation.
Every launch provider on Earth is at our disposal.

Gene, you have the floor. Assemble your team and begin.
```

### 7. Transcript Logger

Log every message to `output/transcript.md` in this format:

```markdown
## Round {n} — {agent_name}

{message_content}

---
```

Also log to console in real-time so the user can watch the simulation unfold.

### 8. Wrap-Up and Document Extraction

At round WRAP_UP_ROUND, inject a system message to Gene:

```
SIMULATION NOTE: You are approaching the end of this session. Over the next
20 turns, produce the Mission Architecture Document. Assign each section to
the responsible agent. Compile their inputs into the final document.
```

After MAX_ROUNDS, the final document can be extracted from the conversation or (better) built incrementally as agents contribute sections. Save to `output/mission_architecture.md`.

-----

## Error Handling

- **API failures**: LiteLLM handles retries. Set `max_retries: 3` in the LLM config.
- **Agent going off-role**: If an agent’s response doesn’t match expected behavior (e.g., Oracle giving strategic advice), the speaker selection function can inject a correction prompt and re-query.
- **Token limits**: Keep agent responses under 400 words via system prompt instruction. If a response exceeds ~800 tokens, log a warning.
- **Model unavailability on OpenRouter**: Set a fallback model in the config list.

-----

## Testing

Before running the full simulation:

1. **Smoke test**: Create one agent (Gene), send one message, confirm OpenRouter responds.
1. **Two-agent test**: Gene + Stacks, 5 rounds, verify speaker selection works.
1. **Oracle test**: Send a message with “ORACLE CHECK: Δv for escape from 408km is 3.2 km/s” and confirm Oracle responds with validation.
1. **Full dry run**: All 6 agents, 10 rounds, verify no crashes and speaker rotation works.

-----

## Dependencies

```
ag2[openai]>=0.6
litellm>=1.40
```

That should be it. AG2 pulls in LiteLLM. No other external dependencies needed.

-----

## Files Referenced

The agent role descriptions (system prompts) are in the companion file: `iss_mission_control_agents.md`. That file contains the complete scenario briefing, all six agent personality/role descriptions, the deliverable structure, and orchestration notes. The implementation should read from that file or have the prompts ported into the `prompts/` directory.
