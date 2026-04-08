---
name: tippy-automation
description: Develop automation tools for Tippy.club
metadata:
  author: tippyclub
  version: "1.0.0"
  argument-hint: <file-or-pattern>
---

# Tippy Automation Skill

Build automation scripts for Tippy.club that find odds and send them as tips to channels.

## Setup

Install the Tippy SDK globally or use npx:

```bash
npm install -g @tippyclub/sdk@latest
# or use directly:
npx @tippyclub/sdk@latest <command>
```

**Requirement:** The Tippy desktop app must be running on the local machine. All CLI commands connect to it — if it's not running, commands will fail to connect.

## Core Workflow

Automation scripts follow this pattern:

1. **List profiles** — find which profile (Chrome tab / bookmaker account) to use
2. **Search odds** — query Bet365 for odds matching a text search
3. **List channels** — find the target channel to send the tip to
4. **Send tip** — post the found odd as a tip to the channel

## CLI Reference

### Profiles

Each profile corresponds to a Chrome tab running a bookmaker account.

```bash
# List all profiles
tippy profiles list
```

Output includes profile IDs. You need a `profileId` to run bookmaker commands.

### Bet365

```bash
# Search for odds by text (requires a profileId)
tippy bet365 search-odds-by-text --profile-id <profileId> --text "Manchester United"

# Or pass the full request as JSON
tippy bet365 search-odds-by-text --json '{"profileId": "abc123", "text": "Manchester United"}'
```

Note: Only Bet365 search is available currently. Other bookmakers are not yet implemented.

### Channels

```bash
# List channels (to find the channelId to send tips to)
tippy channels list
tippy channels list --limit 10

# Send a tip to a channel
tippy channels send-tip --channel-id <channelId> --json '{"odd": {...}}'

# Send with explicit source
tippy channels send-tip --channel-id <channelId> --source SEND_TIP_SOURCE_SDK --json '{"odd": {...}}'
```

The `--source` flag defaults to `SEND_TIP_SOURCE_MINIAPP` when unspecified. Use `SEND_TIP_SOURCE_SDK` for automation scripts.

### Telegram (optional)

```bash
# Stream incoming chat messages
tippy telegram subscribe
```

## Example Automation Script

Write automation scripts in Node.js using the `@tippyclub/sdk` package, which exports the RPC client directly.

```bash
# Install the SDK in your project, depending on the project's package manager
npm install --save @tippyclub/sdk
# or
yarn add @tippyclub/sdk
# or
bun add @tippyclub/sdk
```

```js
import { client } from '@tippyclub/sdk';
import { BetPlatform, SendTipSource } from '@tippyclub/sdk/gen/tippy/v1/channels_pb';

async function main() {
  // 1. Pick the first available profile
  const { profiles } = await client.profiles.list({});
  const profileId = profiles[0].id;

  // 2. Search for odds on Bet365
  const { odds } = await client.bet365.searchOddsByText({
    profileId,
    text: 'Champions League',
  });
  const odd = odds[0];

  // 3. Find the target channel
  const { channels } = await client.channels.list({ limit: 10 });
  const channelId = channels[0].id;

  // 4. Send the tip
  await client.channels.sendTip({
    channelId,
    source: SendTipSource.SDK,
    bet: {
      platform: BetPlatform.BET365,
      betItems: [
        {
          title: odd.title,
          odds: odd.odds,
          // ... populate remaining fields from the odd
        },
      ],
      betSlip: odd.betSlip,
    },
  });
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

## Notes

- All CLI commands output JSON
- If a command fails, it exits with code 1 and prints the error message to stderr
- `channels send-tip` requires the caller to be an admin of the target channel
- Streaming commands (`telegram subscribe`, `bet365 dummy-watch-live-odds`) emit one JSON object per line and run until interrupted
