# OpenClaw Railway Template

## Security Notice

> **This template exposes your OpenClaw gateway to the public internet.** **Please read the [OpenClaw security documentation](https://docs.openclaw.ai/gateway/security) before deploying** to understand the risks and recommended configuration. If you only use chat channels (Telegram, Discord, Slack, WhatsApp) and don't need the gateway dashboard, you can remove the public endpoint from Railway after setup.

<img width="1860" height="2624" alt="CleanShot 2026-02-23 at 21 59 06@2x" src="https://github.com/user-attachments/assets/2605d44c-4319-4e92-838c-3caa726b9595" />

## What you get

- **OpenClaw Gateway + Control UI** (served at `/` and `/openclaw`)
- A friendly **Setup Wizard** at `/setup` (protected by a password)
- Optional **Web Terminal** at `/tui` for browser-based TUI access
- Persistent state via **Railway Volume** (so config/credentials/memory survive redeploys)

## How it works (high level)

- The container runs a wrapper web server.
- The wrapper protects `/setup` with `SETUP_PASSWORD`.
- During setup, the wrapper runs `openclaw onboard --non-interactive ...` inside the container, writes state to the volume, and then starts the gateway.
- After setup, **`/` is OpenClaw**. The wrapper reverse-proxies all traffic (including WebSockets) to the local gateway process.

## Getting chat tokens (so you don't have to scramble)

### Telegram bot token

1. Open Telegram and message **@BotFather**
2. Run `/newbot` and follow the prompts
3. BotFather will give you a token that looks like: `123456789:AA...`
4. Paste that token into `/setup`

### Discord bot token

1. Go to the Discord Developer Portal: https://discord.com/developers/applications
2. **New Application** → pick a name
3. Open the **Bot** tab → **Add Bot**
4. Copy the **Bot Token** and paste it into `/setup`
5. Invite the bot to your server (OAuth2 URL Generator → scopes: `bot`, `applications.commands`; then choose permissions)

### WhatsApp (WhatsApp Web)

1. In `/setup` step **Channels**, enable **WhatsApp** and optionally set an E.164 allowlist (e.g. `+15551234567`) and group policy.
2. Run setup. The template installs the [`@openclaw/whatsapp`](https://docs.openclaw.ai/channels/whatsapp) plugin and writes `channels.whatsapp` to your config.
3. After the gateway starts, open **Open OpenClaw UI** (`/openclaw`) from the setup page and complete **WhatsApp login** there — scan the QR code with your phone (same flow as OpenClaw desktop).
4. Default DM policy is **pairing**. To approve a pending DM user, use **Approve Channel Access** on `/setup` and choose channel **whatsapp** (see [OpenClaw pairing](https://docs.openclaw.ai/channels/whatsapp#approve-first-pairing-request-if-using-pairing-mode)).

OpenClaw recommends a **dedicated phone number** for the bot when possible; see the [WhatsApp channel documentation](https://docs.openclaw.ai/channels/whatsapp) for policies, groups, and troubleshooting.

## Web Terminal (TUI)

The template includes an optional web-based terminal that runs `openclaw tui` in your browser.

### Enabling

Set `ENABLE_WEB_TUI=true` in your Railway Variables. The terminal is **disabled by default**.

Once enabled, access it at `/tui` or via the "Open Terminal" button on the setup page.

### Security

The web TUI implements multiple security layers:

| Control | Description |
|---------|-------------|
| **Opt-in only** | Disabled by default, requires explicit `ENABLE_WEB_TUI=true` |
| **Password protected** | Uses the same `SETUP_PASSWORD` as the setup wizard |
| **Single session** | Only 1 concurrent TUI session allowed at a time |
| **Idle timeout** | Auto-closes after 5 minutes of inactivity (configurable via `TUI_IDLE_TIMEOUT_MS`) |
| **Max duration** | Hard limit of 30 minutes per session (configurable via `TUI_MAX_SESSION_MS`) |

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_WEB_TUI` | `false` | Set to `true` to enable |
| `TUI_IDLE_TIMEOUT_MS` | `300000` (5 min) | Closes session after inactivity |
| `TUI_MAX_SESSION_MS` | `1800000` (30 min) | Maximum session duration |

## Local testing

```bash
docker build -t openclaw-railway-template .

docker run --rm -p 8080:8080 \
  -e PORT=8080 \
  -e SETUP_PASSWORD=test \
  -e ENABLE_WEB_TUI=true \
  -e OPENCLAW_STATE_DIR=/data/.openclaw \
  -e OPENCLAW_WORKSPACE_DIR=/data/workspace \
  -v $(pwd)/.tmpdata:/data \
  openclaw-railway-template

# Setup wizard: http://localhost:8080/setup (password: test)
# Web terminal: http://localhost:8080/tui (after setup)
```

## FAQ

**Q: How do I access the setup page?**

A: Go to `/setup` on your deployed instance. When prompted for credentials, use the generated `SETUP_PASSWORD` from your Railway Variables as the password. The username field is ignored—you can leave it empty or enter anything.

**Q: I see "gateway disconnected" or authentication errors in the Control UI. What should I do?**

A: Go back to `/setup` and click the "Open OpenClaw UI" button from there. The setup page passes the required auth token to the UI. Accessing the UI directly without the token will cause connection errors.

**Q: I don't see the TUI option on the setup page.**

A: Make sure `ENABLE_WEB_TUI=true` is set in your Railway Variables and redeploy. The web terminal is disabled by default.

**Q: How do I approve pairing for Telegram, Discord, or WhatsApp?**

A: Go to `/setup` and use the "Approve Channel Access" dialog to approve pending pairing requests. Pick the channel (telegram, discord, or whatsapp) and enter the code from the user’s chat request.

**Q: I see "pairing required" when opening the Control UI. How do I fix it?**

A: New browsers/devices need a one-time approval from the gateway. Go to `/setup`, click "Manage Devices" in the Devices section, and click "Approve Latest Request". Refresh the Control UI and it should connect. Local connections (127.0.0.1) are auto-approved; remote connections (LAN, public URL) require explicit approval.

**Q: How do I change the AI model after setup?**

A: **Easiest:** go to `/setup` (after the instance is configured) and use the **Model** section — enter `provider/model` (e.g. `anthropic/claude-sonnet-4-20250514`) or an alias, then **Apply model**. The wrapper runs `openclaw models set` and restarts the gateway.

**From Railway without a shell:** set **`OPENCLAW_MODEL_ON_BOOT`** to the same `provider/model` string in Railway Variables and redeploy. On boot, the wrapper runs `models set` **only when that variable’s value changes** (it remembers the last applied value in `${OPENCLAW_STATE_DIR}/.wrapper-last-model-env`). Changing the model from `/setup` does not update that file, so a stale env value won’t keep overwriting your manual change until you change or clear the variable.

**CLI fallbacks:** Web terminal at `/tui` (`ENABLE_WEB_TUI=true`), or from your laptop with the [Railway CLI](https://docs.railway.com/develop/cli): `railway run openclaw models set provider/model-id` (linked to the same service/environment). Use `openclaw models list --all` where you have a shell to list ids.

**Q: How do I access configuration after the initial setup?**

A: Visit `/setup` on your deployed instance at any time — it works both before and after setup. Once configured, the setup page shows your current status along with management tools: **default model** (Apply model), device approval, health checks (Run Doctor), data export, and a reset option. You'll need your `SETUP_PASSWORD` to access it.

**Q: My config seems broken or I'm getting strange errors. How do I fix it?**

A: Go to `/setup` and click the "Run Doctor" button. This runs `openclaw doctor --repair` which performs health checks on your gateway and channels, creates a backup of your config, and removes any unrecognized or corrupted configuration keys.

## Screenshots

## Setup

<img width="2110" height="2032" alt="CleanShot 2026-02-23 at 21 57 59@2x" src="https://github.com/user-attachments/assets/28640eec-fa35-42f2-ba56-cb1fbb9525de" />

## TUI

<img width="2510" height="608" alt="CleanShot 2026-02-23 at 22 08 20@2x" src="https://github.com/user-attachments/assets/61147ec2-ddd5-4b5b-b9ac-0dd81a1ae4c7" />

## Device approval

<img width="1712" height="1376" alt="CleanShot 2026-02-23 at 21 59 21@2x" src="https://github.com/user-attachments/assets/f30ab683-dbc2-4980-ace7-152265e00c79" />

## Support

Need help? [Request support on Railway Station](https://station.railway.com/all-templates/d0880c01-2cc5-462c-8b76-d84c1a203348)
