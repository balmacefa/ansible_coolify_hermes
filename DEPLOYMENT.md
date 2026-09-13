# Deploying Hermes on Coolify

Step-by-step guide for deploying Hermes AI Agent with Web UI on a Coolify instance.

## Prerequisites

- Coolify instance (v4.x+) with Docker support
- SSH access to Coolify host
- At least one LLM provider API key:
  - **OpenRouter** (recommended): https://openrouter.ai
  - **Anthropic**: https://console.anthropic.com
  - **OpenAI**: https://platform.openai.com
  - **Google**: https://console.cloud.google.com
- A public domain to bind the service
- ~2GB free RAM and disk space

## Deployment Steps

### Option 1: Via Coolify Web UI (Recommended)

1. **Create New Service**
   - Open Coolify dashboard
   - Go to **Applications** → **Create**
   - Choose **One-Click Service**
   - Search for **"Hermes Agent"** or **"Hermes with WebUI"**

2. **Configure Service**
   - Set a descriptive name (e.g., `hermes-prod` or `hermes-dev`)
   - Choose target host/server
   - Set the environment in the **Configuration** tab

3. **Set Environment Variables**
   - **At least one LLM provider** (choose based on cost/preference):
     - For budget: `OPENROUTER_API_KEY=sk-or-...`
     - For Claude: `ANTHROPIC_API_KEY=sk-ant-...`
     - For GPT: `OPENAI_API_KEY=sk-...`
     - For Gemini: `GOOGLE_API_KEY=...`
   - **Security**: `SERVICE_PASSWORD_HERMESWEBUI=<strong_random_password>`
   - Leave `SERVICE_URL_HERMESWEBUI_8787` empty (Coolify will populate it)

4. **Configure Domain Routing**
   - Go to **Storages** tab → skip (volumes are auto-created)
   - Go to **Domains** tab
   - Click **Add Domain**
   - Enter your domain (e.g., `hermes.example.com`)
   - Set port to `8787`
   - Coolify's Traefik will auto-route HTTPS

5. **Deploy**
   - Click **Save & Deploy**
   - Monitor deployment in the **Logs** tab
   - Wait for both services to show ✅ (healthy)
   - This typically takes 1-2 minutes

6. **Access**
   - Open `https://hermes.example.com`
   - Login with your `SERVICE_PASSWORD_HERMESWEBUI`

---

### Option 2: Via Docker Compose (Manual)

If deploying directly on the Coolify host:

```bash
# 1. Create project directory
mkdir -p /data/hermes
cd /data/hermes

# 2. Copy docker-compose.yml from this repo
curl -o docker-compose.yml https://raw.githubusercontent.com/balmacefa/ansible_coolify_hermes/main/docker-compose.yml

# 3. Create .env file
cp .env.example .env
# Edit .env and set your LLM keys
nano .env

# 4. Start services
docker-compose up -d

# 5. Check status
docker-compose ps
docker-compose logs -f hermes-webui
```

Access via `http://localhost:8787` or bind to your domain via Traefik.

---

### Option 3: Via Ansible (Playbook)

If integrating into this ansible_coolify repository:

```bash
# From ansible_coolify root
ansible-playbook playbooks/services/deploy_hermes.yml \
  -e 'target_host=coolify_servers' \
  -e 'hermes_domain=hermes.example.com' \
  -e 'openrouter_api_key=sk-or-...' \
  -e 'hermes_webui_password=strong_password'
```

(Playbook template to be added to `playbooks/services/` if desired)

---

## Post-Deployment Configuration

### 1. First Login
- Navigate to `https://hermes.example.com`
- Use the password you set in `SERVICE_PASSWORD_HERMESWEBUI`
- Complete any initial setup prompts

### 2. Configure Agent Behavior
In the Hermes WebUI settings:
- Set default model provider (if using OpenRouter, choose a cost-effective model)
- Configure memory retention and archival
- Set task scheduling parameters if needed

### 3. Test LLM Connection
Send a test message via the chat interface:
- Verify the response completes
- Check Coolify logs if it fails:
  ```bash
  docker logs <hermes_webui_container_id>
  ```

---

## Monitoring & Troubleshooting

### Check Service Health in Coolify

1. Go to service → **Logs** tab
2. View real-time logs for both `hermes-agent` and `hermes-webui`
3. Health check status shown in the service dashboard

### Common Issues

#### WebUI Returns 502/Connection Refused
**Cause**: `hermes-agent` not ready or failed to initialize  
**Solution**:
```bash
docker logs <container_id>
# Check for permission errors or API key issues
```

#### High Memory Usage
**Cause**: Large agent state or concurrent tasks  
**Solution**:
- Increase Coolify host allocation
- Archive old memories in WebUI settings
- Reduce concurrent task limits

#### LLM API Errors
**Cause**: Invalid API key, rate limit, or quota exceeded  
**Solution**:
- Verify API key in Coolify dashboard
- Check provider dashboard for quota/limits
- If using OpenRouter, ensure account has credits

#### Persistent Volume Lost
**Cause**: Docker prune or manual volume deletion  
**Solution**:
```bash
# Restore from backup if available
docker cp hermes_backup:/home/hermes/.hermes <container>:/home/hermes/.hermes
# Restart container
docker restart <container>
```

---

## Backing Up

### Automated Backup (Recommended)

Use Coolify's built-in backup feature:
1. Go to service → **Backup** tab
2. Click **Create Backup**
3. Backup is encrypted and stored

### Manual Backup

```bash
# SSH into Coolify host
docker cp <hermes_webui_container>:/home/hermeswebui/.hermes ./hermes_state_backup
docker cp <hermes_webui_container>:/workspace ./hermes_workspace_backup

# Encrypt with SOPS if part of ansible_coolify
./manage_secrets.sh --encrypt-file hermes_state_backup
./manage_secrets.sh --encrypt-file hermes_workspace_backup
```

### Restore

```bash
docker cp ./hermes_state_backup <hermes_webui_container>:/home/hermeswebui/.hermes
docker restart <hermes_webui_container>
```

---

## Performance Tips

1. **Model Selection**: Use OpenRouter's cost-effective models for testing:
   - `openrouter/auto` (cheapest)
   - `mistral/mistral-7b-instruct` (fast)
   - `anthropic/claude-3-sonnet` (balanced)

2. **Memory Management**: Archive old conversations in WebUI to keep memory fresh

3. **Task Limits**: Set reasonable concurrent task limits to avoid resource exhaustion

4. **Resource Allocation**: Allocate at least:
   - 2GB RAM
   - 5GB disk (for volumes)
   - 1 CPU core minimum

---

## Security Considerations

- **Web UI Password**: Use a strong, random password (20+ chars)
- **API Keys**: Never commit `.env` to git; use Coolify's secret management
- **HTTPS**: Always access via HTTPS (Coolify auto-enables via Traefik)
- **Workspace Access**: Hermes can write to `/workspace` volume — monitor for malicious tasks
- **Rate Limiting**: Coolify/Traefik can add rate limits if needed

---

## Integration with Ansible Coolify

To add Hermes to ansible_coolify's service management:

1. Initialize this repo as a git submodule:
   ```bash
   git submodule add https://github.com/balmacefa/ansible_coolify_hermes vendor/hermes-agent
   ```

2. Create playbook: `playbooks/services/deploy_hermes.yml`

3. Include standard patterns (dry_run gate, no_log for secrets, encrypted backups)

See `CLAUDE.md` for playbook conventions.

---

## References

- [Hermes GitHub](https://github.com/nesquena/hermes-webui)
- [Coolify Documentation](https://coolify.io/docs)
- [OpenRouter API](https://openrouter.ai/docs)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)

---

**Last Updated**: 2026-09-13
