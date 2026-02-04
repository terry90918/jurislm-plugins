# Hetzner Cloud Plugin

Hetzner Cloud MCP server for Claude Code.

## Features

- List, create, and manage Hetzner Cloud servers
- Create, attach, detach, and resize volumes
- Manage firewall rules
- Create and manage SSH keys
- View available images, server types, and locations
- Power on/off and reboot servers

## Requirements

Set your Hetzner Cloud API token as an environment variable:

```bash
export HCLOUD_TOKEN="your_hetzner_api_token"
```

Get your API token from [Hetzner Cloud Console](https://console.hetzner.cloud/) → Project → Security → API Tokens.

## Credits

Uses [hetzner-mcp-server](https://github.com/nityeshaga/hetzner-mcp-server) npm package.
