# Hermes AI Agent with Web UI

Self-hosted Hermes autonomous AI agent with persistent memory, scheduling, and a web-based chat UI. Coolify-compatible Docker Compose deployment.

## Overview

**Hermes** is an autonomous AI agent that combines:
- **Persistent Memory**: Maintains context and learns from interactions
- **Scheduling**: Can schedule and execute tasks
- **Self-Hosted Web UI**: Browser-based interface for chat and control
- **Multi-Provider LLM Support**: OpenRouter, Anthropic, OpenAI, Google APIs

This deployment runs two coordinated services:
1. **hermes-agent**: Core autonomous agent with memory and task execution
2. **hermes-webui**: Web interface for interacting with the agent

## Services

### hermes-agent
- **Image**: `nousresearch/hermes-agent` (SHA256 pinned)
- **Purpose**: Core agent runtime with memory, scheduling, and autonomy
- **Port**: 8000 (internal)
- **Volumes**:
  - `hermes-home`: Agent persistent state and memory
  - `hermes-agent-src`: Agent source code and plugins

### hermes-webui
- **Image**: `ghcr.io/nesquena/hermes-webui:0.51.92`
- **Purpose**: Web-based chat and control interface
- **Port**: `8787`
- **Depends On**: `hermes-agent` (with health check)
- **Volumes**:
  - `hermes-home`: Shared state with agent
  - `hermes-agent-src`: Read-only access to agent
  - `hermes-workspace`: Workspace for agent tasks

## Configuration

### Environment Variables

Required (at least one LLM provider):
- `OPENROUTER_API_KEY`: OpenRouter API key (recommended for cost)
- `ANTHROPIC_API_KEY`: Anthropic Claude API key
- `OPENAI_API_KEY`: OpenAI API key
- `GOOGLE_API_KEY`: Google API key

Hermes-specific:
- `SERVICE_PASSWORD_HERMESWEBUI`: Web UI password/authentication token
- `SERVICE_URL_HERMESWEBUI_8787`: Public URL (set by Coolify routing)

### Default Ports
- **8787**: Hermes Web UI (HTTP)

## Deployment on Coolify

### Prerequisites
1. At least one LLM provider API key (see Environment Variables above)
2. Coolify instance with Docker support
3. ~2GB RAM + storage for persistent volumes

### Steps

1. **Create Hermes Service in Coolify**
   - Go to Applications → Create → One-Click Service
   - Select "Hermes Agent with WebUI" or manually paste this `docker-compose.yml`

2. **Configure Environment Variables**
   - Set at least one LLM provider key (OPENROUTER recommended for cost-effective testing)
   - Set `SERVICE_PASSWORD_HERMESWEBUI` for web UI authentication
   - Coolify automatically sets `SERVICE_URL_HERMESWEBUI_8787`

3. **Configure Domain/Public URL**
   - Bind Hermes to a public domain (e.g., `hermes.example.com`)
   - Coolify's Traefik proxy automatically routes traffic

4. **Start the Service**
   - Coolify deploys and monitors both containers
   - Wait for health checks to pass on both services

## Usage

### Web UI Access
```
https://hermes.example.com/
```
Login with the `SERVICE_PASSWORD_HERMESWEBUI` you configured.

### Persistent Data
All agent state, memory, and chat history is stored in the `hermes-home` volume and persists across restarts.

### Task Execution
Hermes can schedule and execute tasks autonomously. Check the web UI's task dashboard for:
- Scheduled tasks
- Execution history
- Memory/context snapshots

## Health Checks

Both services include health checks:

- **hermes-agent**: Checks for `.hermes` home directory initialization
- **hermes-webui**: HTTP GET to `/health` endpoint

If health checks fail, check logs via Coolify's monitoring dashboard.

## Backing Up

To backup agent memory and configuration:
```bash
docker cp <hermes_container>:/home/hermes/.hermes ./hermes_backup
docker cp <hermes_container>:/workspace ./workspace_backup
```

Restore:
```bash
docker cp ./hermes_backup <hermes_container>:/home/hermes/.hermes
docker cp ./workspace_backup <hermes_container>:/workspace
```

## Troubleshooting

### WebUI won't start
- Verify `hermes-agent` is healthy (check logs)
- Check LLM provider API keys are valid
- Ensure sufficient disk space for volumes

### Memory/State Lost
- Verify `hermes-home` volume is persisted (not ephemeral)
- Check Docker volume is not pruned during cleanup

### Slow Performance
- Consider resource constraints; Hermes benefits from >2GB RAM
- Check LLM provider rate limits
- Review agent logs for errors

## References

- [Hermes WebUI GitHub](https://github.com/nesquena/hermes-webui)
- [Nous Research](https://github.com/nesquena)
- [OpenRouter (API provider)](https://openrouter.ai)
- [Anthropic Claude](https://console.anthropic.com)
- [OpenAI](https://platform.openai.com)

## License

Hermes is provided by Nous Research. See the [Hermes repository](https://github.com/nesquena/hermes-webui) for license details.

---

**Last Updated**: 2026-09-13
**Coolify Compatibility**: v4.x
**Images**: Pinned by SHA256 for reproducibility
