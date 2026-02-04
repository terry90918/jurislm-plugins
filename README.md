# JurisLM Plugins Official

A Claude Code Plugin Marketplace for cloud infrastructure management.

## Installation

Add the marketplace:

```
/plugin marketplace add terry90918/claude-plugins
```

Install plugins:

```
/plugin install hetzner@jurislm-plugins-official
```

## Available Plugins

### hetzner

Hetzner Cloud MCP server for managing cloud infrastructure through Claude Code.

**Features:**
- List, create, and manage servers
- Create, attach, detach, and resize volumes
- Manage firewall rules
- Create and manage SSH keys
- View available images, server types, and locations
- Power on/off and reboot servers

**Setup:**

1. Get your API token from [Hetzner Cloud Console](https://console.hetzner.cloud/) → Project → Security → API Tokens

2. Add to `~/.zshenv`:
   ```bash
   export HETZNER_API_TOKEN="your_token_here"
   ```

3. Restart Claude Code

## License

MIT
