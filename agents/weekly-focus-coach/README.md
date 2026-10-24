# Weekly Focus Coach

> A text based Google ADK agent that helps people turn important goals into realistic next steps and see how their time and outcomes are progressing.

## Overview

### Problem Statement & Intended Users

Procrastination can make it hard to start work that matters. Weekly Focus Coach gives users a supportive check-in, tracks one or two weekly goals, and suggests a manageable action based on the user's actual progress and available time. It is intended for people working toward goals such as interview preparation, focused study, reading, or exercise.

### Key Features

- Record up to two weekly goals with priorities, clear success criteria, and optional longer term goal context.
- Track time spent separately from completed outcome milestones.
- Log and correct activities with a date, duration, accomplishment, and morning/afternoon/evening label.
- Show an independent text progress bar for each goal.
- Suggest a realistic next focus and save a plan only after the user approves it.

The first version accepts text. Voice check-ins, scheduled reminders, and a website dashboard are future extensions.

## Technical Stack

- **Language & Runtime:** Python 3.12 (Python 3.10+ supported)
- **Agent Framework / Core Library:** Google Agent Development Kit (ADK) 2.13.0
- **Models:** Gemini Flash via Google AI Studio
- **Data Storage:** Local, atomically replaced JSON file; Pydantic validates data
- **External APIs / Tools:** Gemini API; ADK's development chat interface

## Architecture & Data Flow

```text
Text check-in
     |
     v
Gemini through ADK <----> six Python function tools
                              |             |
                    validation/metrics   local JSON
                              |
     <---- response with per-goal progress bars and a next step
```

The deterministic models, calculations, and JSON store are separate from the ADK agent and can be wrapped by a future website backend. The ADK browser interface is for local development and demonstration, not a production website.

## Data Model

`weekly_focus_coach/models.py` defines JSON serializable, validated records:

- **WeeklyGoal:** title, week start, priority, success criteria, time or milestone metric, optional long term goal and target date.
- **Milestone:** a measurable outcome with an ID, reported completion state, and completion note.
- **Activity:** date, description, duration, period of day, optional goal link, and accomplishment.
- **FocusPlan:** date, available minutes, proposed actions, deferred items, and approval time. Only explicitly approved plans are saved.
- **FocusState:** schema version, revision, goals, activities, and plans.

Minute goal bars use logged minutes against the goal's target. Milestone goal bars use completed milestones against all planned milestones. An activity log never completes a milestone. Each goal keeps its own bar so progress on a secondary goal cannot obscure the primary goal.

The JSON file defaults to `data/focus.json` in the project folder. Set `FOCUS_DATA_PATH` to choose another location. Writes use a temporary file and atomic replace; the file is created with owner-only permissions. Malformed data is reported without overwriting the original. JSON is ignored by Git.

## Prerequisites

- Python 3.12 recommended; Python 3.10 or later supported.
- `uv` for an isolated environment and dependency installation.
- A Google AI Studio API key to run live Gemini conversations. Offline tests do not need a key.
- Terminal and browser for ADK's local development interface.

## Installation & Setup

From the repository root:

```bash
cd agents/weekly-focus-coach
uv venv --python 3.12 .venv
```

Install from public PyPI. `--no-config` avoids a globally configured package index:

```bash
uv --no-config pip install \
  --python .venv/bin/python \
  --default-index https://pypi.org/simple \
  -r requirements.txt
```

Set up the local key file and add the key:

```bash
cp .env.example weekly_focus_coach/.env
```

Edit `weekly_focus_coach/.env` and replace the placeholder `GOOGLE_API_KEY`. This secret file and the focus data are excluded from Git.

## Configuration & Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `GOOGLE_GENAI_USE_VERTEXAI` | Select Google AI Studio instead of Vertex AI. | `FALSE` |
| `GOOGLE_API_KEY` | Google AI Studio key used for live Gemini conversations. | Required for live use |
| `MODEL_NAME` | Compatible Gemini model name. | `gemini-3.5-flash-lite` |
| `FOCUS_TIMEZONE` | Timezone used to determine the current date and week. | `UTC` |
| `FOCUS_DATA_PATH` | Optional location for the JSON data file; relative paths use the project root. | `data/focus.json` |

Keep the real API key only in `weekly_focus_coach/.env`. Do not place it in source files, terminal transcripts, or committed documentation. Offline tests do not require the key.

## How to Run

Activate the environment and run the ADK web development interface from the project root:

```bash
source .venv/bin/activate
adk web --port 8000
```

Open `http://localhost:8000` and choose `weekly_focus_coach` from the agent list. The interface accepts text messages. Local agent and Gemini setup is required for a live conversation.

## Available Agent Tools

| Tool | What it does |
|---|---|
| `set_weekly_goal` | Creates a time based or milestone based weekly goal. |
| `log_activity` | Records activity time and accomplishment separately. |
| `correct_activity` | Edits a prior log without adding a duplicate. |
| `update_goal_progress` | Sets the user's reported completed milestone IDs. |
| `get_weekly_metrics` | Returns individual progress bars, logs, and saved plans. |
| `save_focus_plan` | Saves a selected action after the user confirms it. |

## Example Conversation

The demo scenario has six of eight focused work outcomes completed and one of four interview preparation milestones completed, with two days left. The main goal's progress remains visible even though many smaller tasks are done.

The following is an illustrative prompt sequence (Gemini wording can vary):

```text
User: My primary goal is interview prep. I planned four milestones and finished only "Practice arrays". My other goal was to finish eight focused work outcomes; six are done. I have two days left.
User: I spent 45 minutes practicing arrays this evening. I have only 20 minutes free today. What should I do next?
Coach: Focused work is 6 of 8 outcomes complete. Interview prep is 1 of 4, so your primary goal still needs attention. With 20 minutes, try one short practice answer. Would you like me to save that plan?
User: Yes.
```

The dialogue above illustrates intended behavior; it is not a verbatim transcript from a verified live Gemini run.

## How to Test

Run the deterministic suite without a model API key:

```bash
source .venv/bin/activate
PYTHONPATH=. python -m unittest discover -s weekly_focus_coach/tests -v
```

Tests cover JSON round trips and file permissions, weekly goal limits, separate time and outcome progress, per goal bars, corrections, approval before saving, malformed data preservation, and invalid dates and IDs.

To check the agent entry point and tool wiring, run `adk web --port 8000` and try the example prompts. A live Gemini conversation requires a valid API key and network access.

Live smoke test: on 2026-10-24, the local ADK web interface returned `READY` for a simple prompt using `gemini-3.5-flash-lite`. The longer example conversation above is illustrative, not a verbatim test transcript.

## Known Limitations

- Local JSON storage supports a single local user and one running application process; it is not a shared or concurrent database.
- The agent requires Gemini network access for conversation; deterministic storage and calculations can be exercised offline.
- Progress is based on user reported entries and milestones; the agent does not independently assess interview readiness.
- The local ADK development interface is for testing and demonstration, not production deployment.
- Push reminders, voice input, accounts, a hosted API, and a web dashboard are not implemented.
- Tested on macOS with Python 3.12; other operating systems are unverified.

## Security & Privacy Considerations

- Store the Gemini key only in `weekly_focus_coach/.env`; do not commit it.
- Activity, goal, and plan data stay in the local JSON file. Gemini receives conversation messages and tool context needed to answer; do not enter sensitive personal information.
- The JSON data file is ignored by Git and written with owner-only file permissions.
- The agent has no arbitrary shell or file tools. Its Python functions can only operate on the configured focus data store.
- The local ADK development interface should only be exposed on a trusted development machine.

## Future Improvements

- [ ] Accept spoken check-ins and let users correct extracted logs.
- [ ] Add scheduled, opt-in reminders that prompt users to begin a small next action.
- [ ] Provide a web API and dashboard using the same goal and progress models.
- [ ] Support other goal units and explicit weekly milestone templates.
- [ ] Replace single user JSON storage with a database for a hosted multi-user service.
- [ ] Add user selected commitment blocks and available time schedules.

## License

This project is contributed under the repository's open-source license.
