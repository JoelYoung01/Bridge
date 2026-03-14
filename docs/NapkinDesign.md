# Bridge Harness - Napkin Design

This is my back-of-the-napkin design plan for the agent harness called 'Bridge'. Not married to that name, the idea behind it is that the human's interface is like the bridge of a ship. All vital controls and information present, with the ability to leave the main display and drill into various aspects of the system. Or leave the system on autopilot, the entire system should be able to handle the entire pipeline autonomously.

## The Pipeline

The pipeline refers to the entire flow of work. First, an event occurs. This could be a feature request from the Bridge UI, a bug report, a PR to review, etc. This event gets traiged by an LLM and tagged, eventually landing in it's respective queue to be handled by a specific type of role as a ticket. The queues are for code-writing agents. The respective agents will be spun up to handle the ticket, and write code addressing the ticket. When the agent has completed the chunk of work, it will send it's changes through review, testing, and finally merging. Definition of Done will be when the work completed in the ticket is deployed to Production. The merging and deployment process will need to be linear, so the merge reviews will be serial and queued, and handled by a single entity. The deployments should be polled for status, and tagged with the ticket details. That way, when a deployment fails, the resulting ticket from the failure can be created with the original work context accessible.

## Core Packages

- Agent: the core harness, controls the actual agent execution and tool use.
- Daemon: the core process that orchestrates agents, monitors progress, routes events, hosts the management web server, and probably more. Also handles state management (task chains, storing logs, agent traces, etc).
- CLI: a command-line interface for start/stop/pause/etc for agents, overall start/stop of the entire system, status utils, etc.
- Web UI: a web interface for monitoring and managing agents, viewing logs, fielding questions and design decisions, and more. The primary way to manage the entire system. The idea for this interface is to be a spa that hits the daemon's web api and watches events via websockets probably. Not sure about that 100%

### Agent

The agent is a process that is modeled via a markdown file probably, that describes the agent's role and it's goal. It's a while loop that executes either an LLM call or a tool call each iteration, until the LLM calls to exit.

The markdown files that are roles should be easily editable and swappable, so they should probably live in a directory somehow. This gives the user the ability to easily change aspects or add bits of improvements. The files should be loaded at the start of the session, so that new changes to the files are picked up asap. The session itself should be stored by the daemon in the db, along with the name of the agent (which keys to it's markdown role file) and probably a hashed version of the file. It may be more valuable to use some more clear versioning for the files, but the hash seems like the easiest way to tie versions back, or at least tell if the agent is not using the latest version of the file.

The agents also have access to some tools, the agent loop will handle the tool calls. The tools that it needs that I can think of off the top of my head are:

- read files
- write files
- run terminal commands
- pull websites (parse html to markdown first)
- screenshot websites
- ask questions
- end session

I think the majority of agents will be making code changes. Their session will start with a checked out worktree of the project they are assigned too, and an onboarding file aka the project's AGENTS.md file. They should get their own branch as well, which uses their name and their hashed session id.

The agent's session will also usually be part of a greater workflow, so they will need to load context from previous sessions if they exist.

I want the agent workflows to be easily editable. I have a couple ideas for how this could be run.
The first is to have a markdown file that describes the workflow in clearly defined steps, with maybe the context needed between the steps. This file would then be loaded by a supervisor agent that essentially delegates the tasks out to a subagent, and then sleeps until the subagent reports done. It would then check the results from the subagent, and then decide to either send the task step back or move on to the next step.
The second way I can think of to do this is to use a yaml file that describes the workflow in a similar way, but uses a much more structured approach, so that it can be easily validated  after a user makes changes. This would allow for a more opinionated approach, but also might pigeonhole the system into a less extensible way. We would probably still use a manager agent to handle the handoffs between stages so the flexibility is still there, just not infinitely extensible and does not leave room for the manager to make judgement calls.
Each step is well defined in a concise paragraph to clearly define the goal and any context necessary to achieve the goal. There should be well defined expected outputs, or at least a way to verify / evaluate the results.

Everything that the agent does will be traced, probably in a database? I would imagine that's the easiest way to handle searching and callback. I am not sure if the best way to structure this is to do a per-message row in a table that links to a session and whatnot, like a linked list essentially, or if we should just serialize the entire session as a single text column and just do a row per session. The single session text field will be simpler, but I am not sure about performance since the row will be constantly updated, vs the row per message where it's just inserting row after row. I plan on using SQLite so I am not sure the most performant and wise option. This is open ended and will require a bit of research and decision.

### Daemon

The daemon is a core process that handles all the deterministic parts of this system. There will be a database for storage of stuff like agent traces
