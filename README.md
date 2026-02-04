# Terry's Claude Code Plugins

A collection of Claude Code plugins for cloud infrastructure management.

## Installation

```bash
/plugin marketplace add terryc/claude-plugins
```

Then install individual plugins:

```bash
/plugin install hetzner
```

## Available Plugins

### hetzner

Hetzner Cloud MCP server for Claude Code.

**Features:**
- List, create, and manage Hetzner Cloud servers
- Create, attach, detach, and resize volumes
- Manage firewall rules
- Create and manage SSH keys
- View available images, server types, and locations
- Power on/off and reboot servers

**Requirements:**
- Set `HCLOUD_TOKEN` environment variable with your Hetzner API token

## License

MIT
