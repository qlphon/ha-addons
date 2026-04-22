# qlphon Home Assistant Add-ons

Custom add-on repository for Home Assistant OS.

## Add-ons in this repository

| Add-on | Description |
|--------|-------------|
| [FreeRADIUS + daloRADIUS](./freeradius) | FreeRADIUS 3 with the daloRADIUS web UI, backed by SQLite. Single container, no external DB. |

## Installation

1. In Home Assistant, open **Settings → Add-ons → Add-on Store**.
2. Click the three-dot menu in the top right and choose **Repositories**.
3. Add `https://github.com/qlphon/ha-addons` and save.
4. The add-ons will appear at the bottom of the store.

## Images

Container images are published to GitHub Container Registry:

- `ghcr.io/qlphon/ha-addons/freeradius-amd64`

## License

MIT
