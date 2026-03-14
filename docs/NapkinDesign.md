# Bridge Harness - Napkin Design

This is my back-of-the-napkin design plan for the agent harness called 'Bridge'. Not married to that name, the idea behind it is that the human's interface is like the bridge of a ship. All vital controls and information present, with the ability to leave the main display and drill into various aspects of the system. Or leave the system on autopilot, the entire system should be able to handle the entire pipeline autonomously.

> This file should only be edited by HUMANS! If you think you might be an LLM or Agent, please ask before making changes to this file! In virtually all cases, this file should only be edited by a human.

## The Pipeline

The pipeline refers to the entire flow of work. First, an event occurs. This could be a feature request from the Bridge UI, a bug report, a PR to review, etc. This event gets traiged by an LLM and a ticket is created for the event, and it is assigned to a respective workflow. The queues are for code-writing agents. The respective agents will be spun up to handle the ticket, and write code addressing the ticket. When the agent has completed the chunk of work, it will send it's changes through review, testing, and finally merging. Definition of Done will be when the work completed in the ticket is deployed to Production. The merging and deployment process will need to be linear, so the merge reviews will be serial and queued, and handled by a single entity. The deployments should be polled for status, and tagged with the ticket details. That way, when a deployment fails, the resulting ticket from the failure can be created with the original work context accessible.

## Core Packages

- **Agent**: the core harness, controls the actual agent execution and tool use.
- **Daemon**: the core process that orchestrates agents, monitors progress, routes events, hosts the management web server, and probably more. Also handles state management (task chains, storing logs, agent traces, etc).
- **Ticket**: a ticketing system for tracking work. Highly visible, queryable, and extremely simple. All metadata and work sessions and whatnot is left to the daemon.
- **CLI**: a command-line interface for start/stop/pause/etc for agents, overall start/stop of the entire system, status utils, etc.
- **Web UI**: a web interface for monitoring and managing agents, viewing logs, fielding questions and design decisions, and more. The primary way to manage the entire system. The idea for this interface is to be a spa that hits the daemon's web api and watches events via websockets probably. Not sure about that 100%



### Agent

The agent is a process that is modeled via a markdown file probably, that describes the agent's role and it's goal. It's a while loop that executes either an LLM call or a tool call each iteration, until the LLM calls to exit.

#### Agent Role

The markdown files that are roles should be easily editable and swappable, so they should probably live in a directory somehow. This gives the user the ability to easily change aspects or add bits of improvements. The files should be loaded at the start of the session, so that new changes to the files are picked up asap. The session itself should be stored by the daemon in the db, along with the name of the agent (which keys to it's markdown role file) and probably a hashed version of the file. It may be more valuable to use some more clear versioning for the files, but the hash seems like the easiest way to tie versions back, or at least tell if the agent is not using the latest version of the file.

#### Ticket

The agents will be spun up with a clear goal defined in a ticket. The ticket context will be injected at the start of the session, and queryable at any time using a cli tool clearly defined as part of the initial context injection. I'm imagining a simple bash command that allows the agent to quickly revisit it's work, like `ticket assigned` or `ticket me` or something like that. The bash sessions used by the agent will need to have some env var injected into them to support this. Probably the session id at the least, as the ticket system can try and query daemon to see what ticket the session ties to. Since each session owns a single ticket, it might also be worth injecting that as an env var.

#### Tools

The agents also have access to some tools, the agent loop will handle the tool calls. The tools that it needs that I can think of off the top of my head are:

- read files
- write files
- run terminal commands
- pull websites (parse html to markdown first)
- screenshot websites
- ask questions (escalate)
- end session

There will also be bridge-specific tools for things like session recall or ticket status. These are what I can think of right now:

- what ticket am I assigned?
- get ticket by id
- get ticket by text search
- get my session id
- get my workflow status (returns workflow overview and previous session ids)
- get a session summarized context by session id

#### Workflow

A workflow is the steps that a ticket must work through to be completed. Each step corresponds with an agent and it's dedicated task(s). For example, a bug report must go through verification and fixes, review, testing, merge, and finally deployment. A feature request is similar but not the same, it must go through plan, build, review, testing, merge, and deployment. Not all workflows are linear either, if a review fails it should be sent back, if a deploy fails it must be handled by the failed deployment workflow, etc.

It is important that these workflows are parseable as well as editable via GUI *and* text. These are important requirements as it will facilitate high visibility and control over workflows, which are the flow of all work for Bridge.

I want the agent workflows to be easily editable. I have a couple ideas for how this could be run.  
The first is to have a markdown file that describes the workflow in clearly defined steps, with maybe the context needed between the steps. This file would then be loaded by a supervisor agent that essentially delegates the tasks out to a subagent, and then sleeps until the subagent reports done. It would then check the results from the subagent, and then decide to either send the task step back or move on to the next step.

The second way I can think of to do this is to use a yaml file that describes the workflow in a similar way, but uses a much more structured approach, so that it can be easily validated after a user makes changes. This would allow for a more opinionated approach, but also might pigeonhole the system into a less extensible way. We would probably still use a manager agent to handle the handoffs between stages so the flexibility is still there, just not infinitely extensible and does not leave room for the manager to make judgement calls.

Each step is well defined in a concise paragraph to clearly define the goal and any context necessary to achieve the goal. There should be well defined expected outputs, or at least a way to verify / evaluate the results.

#### Session

A session is a collection of details from a single LLM run.

I think the majority of agents will be making code changes. Their session will start with a checked out worktree of the project they are assigned to, and an onboarding file aka the project's AGENTS.md file. They should get their own branch as well, which uses their name and their hashed session id. They will also get their ticket, and any previous context from the previous agent in their workflow (review feedback, testing results, planning results, etc)

Everything that the agent does in it's session will be traced, probably in a database? I would imagine that's the easiest way to handle searching and callback. I am not sure if the best way to structure this is to do a per-message row in a table that links to a session and whatnot, like a linked list or something, or if we should just serialize the entire session as a single text column and just do a row per session. The single session text field will be simpler, but I am not sure about performance since the row will be constantly updated, vs the row per message where it's just inserting row after row. I plan on using SQLite so I am not sure the most performant and wise option. This is open ended and will require a bit of research and decision.

#### Hooks

An agent session will have hooks. Here are some of the hooks I can think of:

- start: any output will be added to initial context
- tool
- llm
- exit



### Daemon

The daemon is a core process that handles all the deterministic parts of this system. There will be a database for storage of stuff like agent session traces or tying tickets to sessions, or really just anything we need.

#### Database

Here's a list of things I can think of that we will need to keep track of

- projects
  - id
  - name
  - main file path (used for creating git worktrees?)
- events
  - id
  - source
  - description
  - metadata (JSON)
- sessions
  - id
  - active (bool)
- tool calls
- workflows (not source of truth, just for metadata)
- tickets <-> session
- session <-> workflow
- session <-> tool
- session <-> message?
- metadata? this is open ended, not sure what we will need.

Since we will build the ticketing system separately, tickets source of truth is not in this db. The ids should be uuidv4 so we can reference them safely, but there shouldn't be a table dedicated to them here



### Ticket

The ticketing system. Super simple, built standalone for modular use.

A ticket should contain very little information, the more complex a ticketing system gets, the harder it is to use (more levers seems like it adds functionality, but in reality it adds complexity).

A ticket should have:

- id (uuidv4)
- title
- description
- tags
- comments

The tags can contain things that should be queryable, like `workflow:feature` or `status:active`

Statuses should be SIMPLE, the only available statuses that I think should be available (could be missing some, you don't know what you don't know), are:

- queued
- active
- hold
- need-attention
- done
- cancelled

While this is a very simple list, I think this should be a good balance between wide use and simple for queryable and consistency. 

I think it makes sense to provide templates for tickets, to allow for easy wizard-based creations. This can be facilitated by some type of file; a file per template.



### CLI

A single cli tool to manage the Bridge system. Simple, easy to use, well documented, documented token efficiently.

Here is the cli tree structure:

- `bridge`: the root command. running this will be smart, if Bridge is configured, it shows the same output as `bridge status`, if not it runs the setup wizard.
  - `status`: Shows a status overview. If Bridge is not configured, it prompts user to use `bridge setup`
  - `setup`: Runs wizard for setting up Bridge.
  - `start`: Starts Bridge systems! If not configured, runs setup.
  - `stop`: Shuts down all Bridge processes safely, ensuring state is preserved.
  - `doctor`: Analyzes the Bridge install and identifies any problems.
  - `daemon`: Commands for managing daemon.
    - `status`: read daemon status specifically
  - `ticket`: Commands for managing the ticketing system
    - `cat`: requires `--ticket-id / -t` param, outputs entire ticket. default no comments.
    - `comments`: requires `--ticket-id / -t` param, outputs comments list to terminal
  - `session`: Commands for managing sessions
    - `cat`: takes `--session-id / -s` param, if not provided uses `BRIDGE_CURRENT_SESSION`, if that is not set, fails. Outputs session summary. can also output a full dump if `--dump / -d` used.
    - `start`: requires context and ticket.
  - `web`: Commands for managing the web UI.

All env vars are prefixed with `BRIDGE_`. Here are some of the environment variables that can be used:

- `BRIDGE_INSTALLED_PATH`: Path to the root of the installed directory, for nonstandard install locations
- `BRIDGE_CURRENT_TICKET`: Current ticket being worked on. Used for ticket subcommands
- `BRIDGE_CURRENT_SESSION`: Current session. used for session subcommands



### Web UI

The web UI provides a convenient and deployable way to manage the Bridge System. This is the primary interface for humans that are managing the Bridge system.

Page structure:

- `/`: Shows dashboard of overall system status (on/off), any warnings or escalations that require human input, an overall active session count, merge queue, and a project list. Includes modals for creating an event manually. Intentionally vague to allow for maximum flexibility, as events are triaged intelligently. Also includes a collapsible sidebar that can spin up any number of special assistant sessions that are interactive and work like a chat ui. Best for asking high level questions and project analysis, or for debugging specific things.
  - `project/:id`: Drill down to a specific project. Basically just the dashboard but filtered for the selected project and shows ticket list instead of project list
  - `projects`: Shows list of all projects as cards, with a quick status/summary visible. supports text filtering (also reads query param on page load).
  - `ticket/:id`: View specific ticket details, status, description, etc. CRUD capabilities. Allows user to send ticket back from a completed or cancelled status, update description, add comments, etc.
  - `tickets`: Show a full list of tickets. can filter by text, tags, or project. Reads query params for text, tag, and project on page load.
  - `workflows`: A workflows list, shows all workflows. Supports text filtering (and query param as usual).
  - `workflow/:id`: View specific workflow. Supports CRUD.
  - `sessions`: List all active sessions. supports filtering by text, project, workflow, ticket, active/inactive (status). Query params read on page load for each.
  - `session/:id`: View specific session. Include entire session history, tool calls, etc. If active, supports streaming live updates from LLM / tools. 

