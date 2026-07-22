# Troubleshooting Guide

## Installation

### `command not found: ai-dev-os`
**Cause:** The install directory is not in PATH.
**Solution:** Add the install directory to PATH. Default locations:
- Linux/macOS: `export PATH="$HOME/.ai-dev-os/bin:$PATH"`
- Windows: Add `%USERPROFILE%\.ai-dev-os\bin` to the system PATH
**Ref:** [Installation Guide](GETTING_STARTED.md#installation)

### `ai-dev-os doctor` fails
**Cause:** Missing dependencies (Docker, Python 3.10+, Node.js 18+).
**Solution:** Run `ai-dev-os doctor --verbose` to see which checks fail. Install missing dependencies via your package manager.
**Ref:** [System Requirements](GETTING_STARTED.md#requirements)

### Port conflict on startup
**Cause:** Port 8080 (UI) or 50051 (gRPC) is already in use.
**Solution:**
```bash
ai-dev-os ui --port 8081
ai-dev-os config set daemon.port 50052
```
**Ref:** [Configuration Reference](CONFIG.md)

## Model / Provider

### `model not found`
**Cause:** The model name is incorrect or not pulled locally.
**Solution:**
- For Ollama: `ollama pull <model-name>`
- For cloud: verify the model name matches the provider's API. Run `ai-dev-os models list` to see available models.
**Ref:** [Model Configuration](MODELS.md)

### `authentication error`
**Cause:** Missing or invalid API key.
**Solution:**
```bash
ai-dev-os config set OPENAI_API_KEY=<key>
ai-dev-os config set ANTHROPIC_API_KEY=<key>
```
Or set the corresponding environment variable. Verify with `ai-dev-os doctor --provider`.
**Ref:** [Provider Setup](PROVIDERS.md)

### `rate limited` errors
**Cause:** Exceeded provider API rate limit.
**Solution:**
- Reduce concurrency: `ai-dev-os config set agent.max_concurrency 1`
- Add delay: `ai-dev-os config set provider.retry_delay 2000`
- Upgrade your provider tier.
**Ref:** [Provider Limits](PROVIDERS.md#rate-limits)

### Slow model responses
**Cause:** Large model on CPU, insufficient RAM, or provider throttling.
**Solution:**
- Use a smaller quantized model (e.g., `q4_k_m` instead of `q8_0`)
- Enable GPU acceleration: `ai-dev-os config set ollama.gpu true`
- Run `ai-dev-os doctor --benchmark` to identify bottlenecks
**Ref:** [Performance Tuning](PERFORMANCE.md)

## Run Failures

### `budget exhausted`
**Cause:** The run exceeded its token or cost budget.
**Solution:**
```bash
ai-dev-os config set agent.budget.tokens 1000000
ai-dev-os config set agent.budget.cost_usd 5.00
```
Restart the run. Use `ai-dev-os run --budget unlimited` for unbounded tasks.
**Ref:** [Budget Configuration](CONFIG.md#budgets)

### `guardian veto: policy violation`
**Cause:** The Guardian agent blocked a tool call that violated a safety policy.
**Solution:**
1. Check the Guardian log: `ai-dev-os logs <run-id> --filter guardian`
2. Adjust policies in `~/.ai-dev-os/policies.yaml`
3. Run with `--policy relaxed` if appropriate
**Ref:** [Safety & Guardian](SAFETY.md)

### Planner loop detected
**Cause:** The planner agent generated the same plan more than 3 times.
**Solution:**
- Refine your task description with more specific requirements
- Increase the loop limit: `ai-dev-os config set agent.planner.max_retries 5`
- Break the task into smaller subtasks
**Ref:** [Agent Orchestration](ORCHESTRATION.md)

### `context too long`
**Cause:** The conversation or code context exceeds the model's context window.
**Solution:**
- Reduce context: `ai-dev-os config set agent.max_context 32000`
- Enable context summarization: `ai-dev-os config set agent.summarize true`
- Use `ai-dev-os run --fresh` to start with an empty context
**Ref:** [Context Management](CONTEXT.md)

## Data

### Memory corruption or vector store errors
**Cause:** Corrupted SQLite or vector index files.
**Solution:**
```bash
ai-dev-os memory repair
ai-dev-os memory reset --force   # Last resort: clears all memory
```
Always back up memory before resetting: `cp -r ~/.ai-dev-os/memory ~/backups/`.
**Ref:** [Memory System](MEMORY.md)

### Database migration errors
**Cause:** Version mismatch between old data and new binary.
**Solution:**
```bash
ai-dev-os db migrate
ai-dev-os db migrate --force
```
If migration fails, downgrade: `ai-dev-os version downgrade <previous-version>`.
**Ref:** [Database Guide](DATABASE.md)

### Export fails silently
**Cause:** Disk full or insufficient permissions on output directory.
**Solution:**
- Check disk space: `df -h ~/.ai-dev-os/`
- Verify write permissions on the export target
- Try a different output path
**Ref:** [Export Commands](CLI.md#export)

## Performance

### High memory usage
**Cause:** Large models loaded in memory, or memory leak in a plugin.
**Solution:**
- Use `ai-dev-os top` to inspect memory per agent
- Set `agent.max_memory_mb 2048` to limit per-agent usage
- Reload: `ai-dev-os agent restart <name>`
**Ref:** [Resource Management](PERFORMANCE.md)

### Slow startup
**Cause:** Loading many plugins or a large vector store on startup.
**Solution:**
- Disable unused plugins: `ai-dev-os plugins disable <name>`
- Reduce memory index size: `ai-dev-os config set memory.index_size 1000`
- Use lazy loading: `ai-dev-os config set plugins.lazy true`
**Ref:** [Performance Tuning](PERFORMANCE.md)

### Worker timeout
**Cause:** A tool or agent exceeded the execution timeout.
**Solution:**
```bash
ai-dev-os config set agent.timeout 300    # Increase timeout in seconds
ai-dev-os config set agent.retry_on_timeout true
```
**Ref:** [Worker Configuration](WORKERS.md)

## Network

### Proxy configuration
**Cause:** Network behind a corporate proxy.
**Solution:**
```bash
ai-dev-os config set http_proxy http://proxy:8080
ai-dev-os config set https_proxy http://proxy:8080
ai-dev-os config set no_proxy localhost,127.0.0.1
```
**Ref:** [Network Configuration](NETWORK.md)

### SSL / certificate errors
**Cause:** Self-signed or corporate CA certificates not trusted.
**Solution:**
```bash
ai-dev-os config set ssl.verify false     # Only for testing
ai-dev-os config set ssl.ca_bundle /path/to/ca.crt
```
**Ref:** [Network Configuration](NETWORK.md#ssl)

### Offline mode not working
**Cause:** A plugin or provider is configured to require internet.
**Solution:**
- Ensure all active providers are set to local: `ai-dev-os config set provider.default ollama`
- Disable telemetry: `ai-dev-os config set telemetry.enabled false`
- Run `ai-dev-os doctor --offline` to validate the offline setup
**Ref:** [Offline Mode](OFFLINE.md)

---

**Related Documents:** [FAQ](FAQ.md), [Getting Started](GETTING_STARTED.md), [Support](SUPPORT.md)
