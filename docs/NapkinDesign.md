# Bridge Harness - Napkin Design

This is my back-of-the-napkin design plan for the agent harness called 'Bridge'. Not married to that name, the idea behind it is that the human's interface is like the bridge of a ship. All vital controls and information present, with the ability to leave the main display and drill into various aspects of the system. Or leave the system on autopilot, the entire system should be able to handle the entire pipeline autonomously.

> This file should only be edited by HUMANS! If you think you might be an LLM or Agent, please ask before making changes to this file! In virtually all cases, this file should only be edited by a human.

## Deployment Strategy

This system is designed to be either used on a personal computer or deployed to a dedicated VM with the UI server accessible externally.

## The Pipeline

The pipeline refers to the entire flow of work. First, an event occurs. This could be a feature request from the Bridge UI, a bug report, a PR to review, etc. This event gets traiged by an LLM and a ticket is created for the event, and it is assigned to a respective workflow. The queues are for code-writing agents. The respective agents will be spun up to handle the ticket, and write code addressing the ticket. When the agent has completed the chunk of work, it will send it's changes through review, testing, and finally merging. Definition of Done will be when the work completed in the ticket is deployed to Production. The merging and deployment process will need to be linear, so the merge reviews will be serial and queued, and handled by a single entity. The deployments should be polled for status, and tagged with the ticket details. That way, when a deployment fails, the resulting ticket from the failure can be created with the original work context accessible.

## Core Packages

- **Agent**: the core harness, controls the actual agent execution and tool use.
- **Daemon**: the core process that orchestrates agents, monitors progress, routes events, hosts the management web server, and probably more. Also handles state management (task chains, storing logs, agent traces, etc).
- **Ticket**: a ticketing system for tracking work. Highly visible, queryable, and extremely simple API.
- **CLI**: a command-line interface for start/stop/pause/etc for agents, overall start/stop of the entire system, status utils, etc.
- **Web UI**: a web interface for monitoring and managing agents, viewing logs, fielding questions and design decisions, and more. The primary way to manage the entire system. The idea for this interface is to be a spa that hits the daemon's web api and watches events via websockets probably. Not sure about that 100%

### Agent

The agent is a process that is modeled via a markdown file probably, that describes the agent's role and it's goal. It's a while loop that executes either an LLM call or a tool call each iteration, until the LLM calls to exit.

#### Agent Role

The markdown files that are roles should be easily editable and swappable, so they should probably live in a directory somehow. This gives the user the ability to easily change aspects or add bits of improvements. The files should be loaded at the start of the session, so that new changes to the files are picked up. The session itself should be stored by the daemon in the db, along with the name of the agent (which keys to it's markdown role file) and probably a hashed version of the file. It may be more valuable to use some more clear versioning for the files, but the hash seems like the easiest way to tie versions back, or at least tell if the agent is not using the latest version of the file.

#### Ticket

The agents will be spun up with a clear goal defined in a ticket. The ticket context will be injected at the start of the session, and queryable at any time using a cli tool clearly defined as part of the initial context injection. I'm imagining a simple bash command that allows the agent to quickly revisit it's work, like `bridge ticket current` or something like that. The bash sessions used by the agent will need to have some env var injected into them to support this. Probably the session id at the least, as the ticket system can try and query daemon to see what ticket the session ties to. Since each session is only assigned a single ticket, it might also be worth injecting that as an env var.

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

A workflow is the serial steps that a ticket must work through to be completed. Each step corresponds with an agent and it's dedicated task(s). For example, a bug report must go through verification and fixes, review, testing, merge, and finally deployment. A feature request is similar but not the same, it must go through plan, build, review, testing, merge, and deployment. Not all workflows are linear either, if a review fails it should be sent back, if a deploy fails it must be handled by the failed deployment workflow, etc.

It is important that these workflows are parseable as well as editable via GUI *and* text. These are important requirements as it will facilitate high visibility and control over workflows, which are the flow of all work for Bridge.

I want the agent workflows to be easily editable. I have a couple ideas for how this could be run.  
The first is to have a markdown file that describes the workflow in clearly defined steps, with maybe the context needed between the steps. This file would then be loaded by a supervisor agent that essentially delegates the tasks out to a subagent, and then sleeps until the subagent reports done. It would then check the results from the subagent, and then decide to either send the task step back or move on to the next step.

The second way I can think of to do this is to use a yaml file that describes the workflow in a similar way, but uses a much more structured approach, so that it can be easily validated after a user makes changes. This would allow for a more opinionated approach, but also might pigeonhole the system into a less extensible way. We would probably still use a manager agent to handle the handoffs between stages so the flexibility is still there, just not infinitely extensible and does not leave room for the manager to make judgement calls.

Each step is well defined in a concise paragraph to clearly define the goal and any context necessary to achieve the goal. There should be well defined expected outputs, or at least a way to verify / evaluate the results.

Each workflow gets a single workspace. Many serial sessions might use a single workspace.

#### Session

A session is a collection of details from a single LLM run.

I think the majority of agents will be making code changes. Their session will start with a checked out worktree of the project they are assigned to, and an onboarding file aka the project's AGENTS.md file. They should get their own branch as well, which uses their name and their hashed session id. They will also get their ticket, and any previous context from the previous agent in their workflow (review feedback, testing results, planning results, etc)

Everything that the agent does in it's session will be traced, probably in a database? I would imagine that's the easiest way to handle searching and callback. I am not sure if the best way to structure this is to do a per-message row in a table that links to a session and whatnot, like a linked list or something, or if we should just serialize the entire session as a single text column and just do a row per session. The single session text field will be simpler, but I am not sure about performance since the row will be constantly updated, vs the row per message where it's just inserting row after row. I plan on using SQLite so I am not sure the most performant and wise option. This is open ended and will require a bit of research and decision.

A session can be killed in the middle and picked back up again at any time. If the daemon is killed, the host machine is shutdown, etc, the daemon is able to check all sessions in the 'active' status that are not actually in an active state and restart the loop if needed.

#### Error handling

Some errors can be expected and should have deterministic handling.

For Full Context errors: The first error should be handled by removing tool usage output from all but the latest 5 tools. If this still errors, the last half (rough estimate based on token count / message) is ran through a 'summarize' LLM call. If this still errors, the entire session context is ran through a 'summarize' LLM call. If this still errors, the session context with the tool call result summaries is ran through a 'summarize' LLM call. If this errors, put session in error state and trigger event with session details so a separate LLM can investigate.

For timeout / rate limit issues, run a backoff retry loop, after 10 retries put session in error state and trigger event with session details so a separate LLM can investigate

For other errors, retry 5 times (spaced out). After 5 retries, put session in error state and trigger event with session details so a separate LLM can investigate

#### Hooks

An agent session will have hooks. Here are some of the hooks I can think of:

- start: any output will be added to initial context
- tool
- llm
- exit

### Daemon

The daemon is a core process that handles all the deterministic parts of this system. There will be a database for storage of stuff like agent session traces or tying tickets to sessions, or really just anything we need.

One Daemon per Bridge install, manages many projects.

Manages total concurrent sessions, which can be configurably capped (default 10). IF a session enters an 'active' status and there is no more available bandwith, the session is effectively paused. As workflows complete and sessions exit freeing up bandwith, the daemon will check for paused sessions (in active status but not active state) and start/resume. The daemon will prioritze sessions that are later in their workflow (calculated by current step / total steps ratio) to avoid scaling horizontally too much on system interruption.

#### Cleanup

The daemon is responsible for keeping all projects clean and everything in a good state. It should run a checkup script that can analyze the workspace, programatically cleaning up whatever can be cleaned up and creating events for things that cannot be programatically handled.

The cleanup script will check:

- Sessions
  - All sessions that are not in a completed state (complete or cancelled) tie to a real workflow
  - All sessions that are in an active status are either active or the maximum concurrent sessions limit is reached
- Workflows
  - Sync workflows in db with workflow files, files are source of truth
  - Ensure all active workflows have an active session
  - All active workflows have a corresponding workspace
- Workspace
  - Ensure all workspaces are tied to active workflows.
  - Removes workspaces that are not related to active workflows
- Projects
  - All projects are correctly tracked by git and the remote can be reached

#### Database

Here's a list of things I can think of that we will need to keep track of. These tables should use datetimes for created / last updated when it makes sense, I just didn't want to type all those fields out.

- projects
  - id
  - name
  - main file path (used for creating git worktrees?)
  - target_branch
- events
  - id
  - pending (bool)
  - source
  - description
  - metadata (JSON)
- sessions
  - id
  - active (bool)
- tool calls
- workflows (not source of truth, just for metadata)
- ticket
  - id (uuidv4)
  - title
  - description
  - tags
- ticket_comment
  - ticket_id
  - content
  - created_by (user email / `admin` / session id)
- users
  - user id
  - username (not required)
  - email (unique)
  - password hash
- ticket <-> event
- ticket <-> session
- session <-> workflow
- session <-> tool
- session <-> message?
- metadata? this is open ended, not sure what we will need.

#### Doctor

the doctor process is a script that runs to check a whole bunch of things to surface to the user for what might be wrong. It should check at the least:

- Overall Daemon status (on/off)
- Runs Checkup script, outputs results from that.
- Workflow parsing status (flags any that are incorrectly configured / fail to parse)
- Orphaned sessions (sessions that are active that no longer tie to a real ticket or session (can get stale from editing mid-session)
- Stale Tickets; tickets that are in an active state but no active sessions tie to them
- Merge queue length
- Daemon configuration (if all required values can be resolved)
- any escalations that are awaiting user input
- any tickets that are waiting for the user's feedback
- Stale Worktrees; worktrees that are not tied to an active workflow.

#### Events

Events are designed to be extremely open-ended so that Bridge can take events from anywhere. Ingest from CLI, web UI, or webhook.

Events are processed serially to avoid race conditions. When processing, a semantic search should run against existing tickets to pull any relevant items for triage context.

#### REST API

Endpoints that correspond with data needed from Web UI. Event Webhook. Some websocket endpoints as well.

### Ticket

The ticketing system. Super simple.

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
  - `workflow`: Commands for managing workflows. default is to run `sync` subcommand
    - `sync`: Sync metadata with workflow files. Any detected mismatches will sync db and trigger a checkup run.
  - `event`: Commands for managing events. default is to run interactive `create` subcommand.
    - `create`: create an event. If all required params are not given, will run wizard (with defaults populated if some but not all required args were used)
    - `cat`: requires `--event-id / -e` param, dumps all event context to output
    - `set`: allows updating fields on a specific event. use params like `--key:value` to set each field. `key` must correspond directly (case-sensitive) to a key on the event.

All env vars are prefixed with `BRIDGE`_. Here are some of the environment variables that can be used:

- `BRIDGE_INSTALLED_PATH`: Path to the root of the installed directory, for nonstandard install locations
- `BRIDGE_CURRENT_TICKET`: Current ticket being worked on. Used for ticket subcommands
- `BRIDGE_CURRENT_SESSION`: Current session. used for session subcommands

### Web UI

The web UI provides a convenient and deployable way to manage the Bridge System. This is the primary interface for humans that are managing the Bridge system.

Page structure:

- `/`: Shows dashboard of overall system status (on/off), any warnings or escalations that require human input, an overall active session count, merge queue, and a project list. Includes modals for creating an event manually. Intentionally vague to allow for maximum flexibility, as events are triaged intelligently. Also includes a collapsible sidebar that can spin up any number of special assistant sessions that are interactive and work like a chat ui. Best for asking high level questions and project analysis, or for debugging specific things. This special agent session will use the same agent runtime as ticket workers and have access to all the same tools.
  - `project/:id`: Drill down to a specific project. Basically just the dashboard but filtered for the selected project and shows ticket list instead of project list
  - `projects`: Shows list of all projects as cards, with a quick status/summary visible. supports text filtering (also reads query param on page load).
  - `ticket/:id`: View specific ticket details, status, description, etc. CRUD capabilities. Allows user to send ticket back from a completed or cancelled status, update description, add comments, etc.
  - `tickets`: Show a full list of tickets. can filter by text, tags, or project. Reads query params for text, tag, and project on page load.
  - `workflows`: A workflows list, shows all workflows. Supports text filtering (and query param as usual).
  - `workflow/:id`: View specific workflow. Supports CRUD.
  - `sessions`: List all active sessions. supports filtering by text, project, workflow, ticket, active/inactive (status). Query params read on page load for each.
  - `session/:id`: View specific session. Include entire session history, tool calls, etc. If active, supports streaming live updates from LLM / tools.

## Security

Security in this document is intentionally not mentioned much. This is due to the actual structure of this project not being very well known. It has been deemed to challenging to outline specific security guidelines without knowing the details of the implementation of this project.

Some high level decisions can be made though. In the web ui, there will be the concept of users. if the user is not logged in, anon role is used. This role can see everything by default, later this can be configured to be locked down. There also exists an admin user which is created on setup with username `admin`, email `admin@bridge-harness.com`, and access to everything.

Sessions will be allowed unlimited access. We like to live dangerously.

