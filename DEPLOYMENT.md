# Deployment

## Pushing dashboard.yaml to Home Assistant

The Plants dashboard (`/dashboard-plants/`) is in **storage mode** in HA, which means
it can be updated programmatically via the WebSocket API — no manual copy-paste needed.

**Important:** The HA Lovelace REST API (`/api/lovelace/config`) returns 404 for this
instance. Always use the WebSocket API below. Also, always target the **local** instance
(`homeassistant.lan.candi-home.com:8123`) not the Cloudflare-fronted URL, which blocks
WebSocket connections.

### Prerequisites

```bash
pip3 install pyyaml websockets
```

### Deploy script

```python
import asyncio, json, yaml, websockets, pathlib

TOKEN = "YOUR_LONG_LIVED_ACCESS_TOKEN"  # HA Profile → Security → Long-Lived Access Tokens
URL = "ws://homeassistant.lan.candi-home.com:8123/api/websocket"
DASHBOARD_YAML = pathlib.Path("dashboard.yaml")

config = yaml.safe_load(DASHBOARD_YAML.read_text())

async def main():
    async with websockets.connect(URL) as ws:
        await ws.recv()  # auth_required
        await ws.send(json.dumps({"type": "auth", "access_token": TOKEN}))
        await ws.recv()  # auth_ok

        await ws.send(json.dumps({
            "id": 1,
            "type": "lovelace/config/save",
            "url_path": "dashboard-plants",
            "config": config
        }))
        result = json.loads(await ws.recv())
        print("Result:", json.dumps(result, indent=2))

asyncio.run(main())
```

Run from the repo root. A `"success": true` response means the dashboard updated
immediately — no HA restart needed, changes appear live in the browser.

### Fetching the current live config

Replace `lovelace/config/save` with `lovelace/config` (no body changes needed) to
read the current dashboard back from HA. Useful for syncing after manual edits in the UI.

### Full-width cards in a sections view

To make a card span the full width of a sections-view dashboard, you need **both**:

1. `column_span: 4` on the enclosing `type: grid` section (matches `max_columns`)
2. `grid_options: {columns: full}` on the card itself

Either one alone is not enough.

### Listing all dashboards (useful for finding url_path)

```python
await ws.send(json.dumps({"id": 1, "type": "lovelace/dashboards/list"}))
```
