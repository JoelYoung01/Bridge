This is my back-of-the-napkin design plan for the agent harness called 'Bridge'. Not married to that name.

Core Packages

- Agent: the core harness, controls the actual agent execution and tool use.
- Daemon: the core process that orchestrates agents, monitors progress, routes events, hosts the management web server, and probably more. Also handles state management (task chains, storing logs, agent traces, etc).
- CLI: a command-line interface for start/stop/pause/etc for agents, overall start/stop of the entire system, status utils, etc.
- Web UI: a web interface for monitoring and managing agents, viewing logs, fielding questions and design decisions, and more. The primary way to manage the entire system. The idea for this interface is to be a spa that hits the daemon's web api and watches events via websockets probably. Not sure about that 100%

## Agent

The agent is a process that is modeled via a markdown file probably, that describes the agent's role and it's goal. It's a while loop that executes either an LLM call or a tool call each iteration, until the LLM calls to exit.

The agent will be spun
