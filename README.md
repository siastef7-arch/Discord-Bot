# Rift Quests

A custom League of Legends quest bot for Discord, using Node.js, TypeScript, discord.js, Riot Account-V1 / Match-V5, PostgreSQL and Prisma. Progression belongs to a Discord server and is separate from a player's League competitive rank.

Members link a Riot ID, choose Top / Jungle / Mid / ADC / Support, and play eligible matches. Everyone starts at Bronze. Completing a Bronze quest awards Silver; the path continues through Challenger. Completing the Challenger quest is a final achievement and keeps the Challenger role. Members choose the next quest explicitly; a game cannot advance several ranks.

## Requirements

- Install **Node.js 24.17.0 or newer** from [nodejs.org](https://nodejs.org/en/download). The installer includes npm. Confirm `node --version` and `npm --version` in a new terminal.
- Install [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/) (Windows: enable WSL 2 and Linux containers), or Docker Engine and the Compose plugin on Linux. Start Docker and check `docker compose version`.
- Create a Discord application and have permission to manage your server.
- Obtain a Riot API key. Never paste keys into Discord commands or commit them.

## Discord setup

1. Open the [Discord Developer Portal](https://discord.com/developers/applications), create an application, and open its **Bot** page. Create/reset the bot token and store it only in your local `.env`.
2. Copy the **Application ID** to `DISCORD_CLIENT_ID`. Enable Developer Mode in Discord settings, then copy your server ID into `DISCORD_GUILD_ID`.
3. Leave **Message Content**, **Server Members**, and **Presence** privileged intents disabled. This bot uses only `Guilds`; role management fetches individual members through REST.
4. Under Installation / OAuth2 URL Generator, select **Guild Install**, scopes `bot` and `applications.commands`, and permissions **Manage Roles**, **View Channels**, **Send Messages**. Do not grant Administrator. Use the generated URL to install into your server.
5. Create nine progression roles: Bronze, Silver, Gold, Platinum, Emerald, Diamond, Master, Grandmaster, Challenger. Create five separate position roles: Top, Jungle, Mid, ADC, Support. These should be cosmetic roles without elevated permissions.
6. Move the bot's highest role **above all 14 roles** in Server Settings → Roles. Managed integration roles and `@everyone` cannot be reward roles.
7. Copy the 14 role IDs into `config/server.json` using the example file. Names are for humans; the bot uses IDs. Choose a text announcement channel or keep its ID `null` to disable announcements. Give the bot View Channel and Send Messages there.
8. After startup, run `/admin sync-roles`. It validates all mappings, Manage Roles, and role hierarchy, then queues repairs. Its errors explain which role needs correction.

`/admin` is restricted both in command registration and in the handler to **Manage Server** (`ManageGuild`). Discord administrators implicitly have this permission.

## Riot access and regions

Sign in to the [Riot Developer Portal](https://developer.riotgames.com/) with your Riot account. Copy the development API key into `RIOT_API_KEY`. Development keys expire every 24 hours; renew the key and restart the bot when needed. For a small private community, apply for the appropriate personal access; public deployment requires approved production access. Submit a working prototype and its user flow through the portal, then replace the environment value with the production key and restart. No source changes are needed. See [Riot API key guidance](https://developer.riotgames.com/docs/portal).

`/link region:europe game-name:Example tag-line:EUW` resolves Account-V1 to a permanent PUUID. Supported routing values are `europe`, `americas`, `asia`, `sea`. You may instead provide a platform: EUW/EUW1, EUNE/EUN1, NA/NA1, BR1, LA1, LA2, KR, JP1, TR1, RU, ME1, OC1, PH2, SG2, TH2, TW2, VN2. The platform is saved separately; Account and Match APIs use regional hosts. Routing availability can change with regional migrations. `RIOT_PLATFORM_ROUTES` accepts a JSON map such as `{"OC1":"sea"}` to override defaults, and direct region selection is always available. Hosts are allowlisted, never supplied directly by users.

**Simple Riot ID lookup does not prove account ownership.** The linked account is visibly marked `unverified`, and a PUUID can be claimed only once per server. `AccountVerifier` in `src/riot.ts` is replaceable; the default `UnverifiedAccountVerifier` deliberately does no ownership check. A future RSO implementation should verify the authenticated PUUID and Discord session before accepting the link. RSO requires Riot production access. Server operators must resolve disputed claims; do not treat these cosmetic roles as an identity or access-control system.

## Quick start: Docker

Run these commands from the repository root in PowerShell (use `cp` instead of `Copy-Item` on Linux/macOS):

```powershell
Copy-Item .env.example .env
Copy-Item config/server.example.json config/server.json
```

Edit `.env` with your credentials, server/application IDs, and a strong database password. Edit `config/server.json` with your server and role IDs. Set both `POSTGRES_PASSWORD` and the password in local `DATABASE_URL` consistently. Use a URL-safe password, or percent-encode it in a database URL; Compose interpolates `POSTGRES_PASSWORD` directly into its URL, so a URL-safe password is simplest.

```powershell
docker compose up -d db
docker compose build bot
docker compose run --rm bot npm run db:migrate
docker compose run --rm bot npm run configure
docker compose run --rm bot npm run db:seed
docker compose run --rm bot npm run register
docker compose up -d bot
docker compose logs -f bot
```

The config directory is mounted read-only into the container, so configuration and seed commands use your current JSON files without rebuilding. The Compose bot service migrates on startup. The database is bound to localhost and persists in the `postgres_data` volume. `docker compose down` retains it; adding `-v` destroys the stored progression.

## Local development

After editing `.env` and `config/server.json`:

```powershell
npm ci
npm run db:generate
docker compose up -d db
npm run db:migrate
npm run configure
npm run db:seed
npm run register
npm run dev
```

With npm 12's install-script policy, if prompted, review and approve the `@prisma/engines`, `prisma`, and `esbuild` package scripts using `npm install-scripts approve <package>`. Prisma generation can download its engine when first run. The repository does not require secrets for generation, build, lint, or tests.

`DATABASE_URL` uses `localhost` for local Node execution; Compose overrides it to `db`. Set `DISCORD_GUILD_ID` for immediate server-scoped command registration. Omit it only when registering global commands for a production multi-server bot; global updates can take time to propagate. Avoid registering both global and guild copies of the same commands. `register` replaces this application's commands in the selected scope, not commands belonging to other bots.

To configure another server, use a separate server JSON with that server's IDs, `npm run configure -- path/to/server.json`, and set `DISCORD_GUILD_ID` when seeding that server. The bot needs to be installed there. Server settings, quests, members and role mappings are isolated by server ID.

## Commands

| Command                           | Behavior                                                                                             |
| --------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `/link region game-name tag-line` | Resolve and link a Riot ID, with a 60-second global member cooldown                                  |
| `/unlink`                         | Cancel active quest, clear link, remove configured roles; keep earned rank and deduplication history |
| `/profile`                        | Account, verification status, rank, position and quest progress                                      |
| `/quest list`                     | Five configured paths at your current rank, status and IDs                                           |
| `/quest select position`          | Start the enabled path; switching position discards current active progress                          |
| `/quest active`                   | Active quest requirements and filters                                                                |
| `/progress`                       | Detailed values against targets                                                                      |
| `/check-match`                    | Queue a check; default 60-second persisted cooldown; see `/progress` for results                     |
| `/leaderboard`                    | Top 10 linked members in this server, sorted by rank then link time                                  |
| `/admin quest create json`        | Create a previously missing rank/position path                                                       |
| `/admin quest edit json`          | Replace an existing path definition using full JSON                                                  |
| `/admin quest enable id`          | Allow selection                                                                                      |
| `/admin quest disable id`         | Disallow new selections; existing active snapshots continue                                          |
| `/admin quest assign member id`   | Assign an enabled quest at the member's current rank, replacing active progress                      |
| `/admin sync-roles`               | Validate settings, queue role repair for all known members                                           |

Replies are private to the caller; promotion announcements are public if enabled. Use the web portal, not slash commands, to manage credentials.

## Quest configuration

`config/quests.json` contains **45 unique paths** (9 ranks × 5 positions). All five Bronze examples are enabled. The remaining 40 are editable disabled starting points; enable them after reviewing their balance. With every path seeded, use `edit` rather than `create`. Seed again with `npm run db:seed` after editing JSON; it replaces those quest definitions, including their enabled state, so export/preserve any admin changes first. You can import a smaller JSON array with `npm run db:seed -- path/to/quests.json`.

Example complete quest object for JSON import or the admin `json` option:

```json
{
  "name": "Bronze Trial — Jungle",
  "rank": "BRONZE",
  "position": "JUNGLE",
  "enabled": true,
  "filters": {
    "minDuration": 900,
    "maxDeaths": 8,
    "champions": ["Amumu", "Sejuani"]
  },
  "requirements": [
    { "type": "GAMES_PLAYED", "target": 3, "aggregation": "TOTAL" },
    { "type": "WINS", "target": 1, "aggregation": "TOTAL" },
    { "type": "ASSISTS", "target": 15, "aggregation": "TOTAL" },
    { "type": "KDA", "target": 3, "aggregation": "SINGLE" }
  ]
}
```

Every requirement must complete. `TOTAL` adds the metric from eligible matches, capped at the target. `SINGLE` retains the best one-game value. Separate SINGLE requirements may be achieved in different matches. `filters` apply to the **entire match**: failing champion, duration, or maximum-death restrictions contributes nothing to any requirement. Omit filters to remove restrictions. Duration is seconds, and champion names are Riot internal names compared case-insensitively.

Available metrics: `GAMES_PLAYED`, `WINS`, `KILLS`, `DEATHS`, `ASSISTS`, `KDA`, `CS`, `VISION_SCORE`, `CHAMPION_DAMAGE`, `DAMAGE_TAKEN`, `GOLD_EARNED`, `MULTIKILLS`, `TURRET_TAKEDOWNS`, `OBJECTIVES`, `DRAGON_KILLS`, `BARON_KILLS`, `DURATION`.

- CS = lane minions + neutral minions.
- KDA = (kills + assists) / max(1, deaths). TOTAL KDA sums per-game ratios; use SINGLE for a one-game KDA target.
- DEATHS is a numeric minimum target like other metrics; use `filters.maxDeaths` for a maximum-deaths condition.
- MULTIKILLS sums Riot double/triple/quadra/penta event counters as reported; it does not deduplicate overlapping event categories.
- OBJECTIVES counts participant dragon + baron kills, not every team objective. Other objective metrics can be added to the registry.
- Missing numeric fields contribute zero. Extend the `metrics` registry in `src/rules.ts` to add another metric without changing the worker or generic evaluator.

Selecting a quest snapshots its rules and start time. Later edits do not change active requirements. Only games **started after selection** can count. Starting a quest during a game does not make that game eligible. Disabled higher ranks produce a helpful message until a manager enables a path.

## Match processing and reliability

The worker uses Account-V1 PUUID identity and paginated Match-V5 history, sorts new games chronologically, accepts only map 11 and configured queues (default 420/440), finds the participant by PUUID, and uses `teamPosition`: TOP→Top, JUNGLE→Jungle, MIDDLE→Mid, BOTTOM→ADC, UTILITY→Support. Unknown/missing positions and other positions never count.

Each member/account/match has a database unique key. Serializable transactions and member row locks atomically write progress, processed-match history, completion, audit event, rank and a durable reward. Conflicts retry. A crash before commit rolls everything back; after commit the match is already recorded. A relink cannot make an old processed match count again. Unlinking retains rank to prevent reset/relink rank farming but discards active progress. Unreferenced Riot account display records are removed; processed PUUIDs remain for deduplication.

Polling advances a cursor only after the batch succeeds. Every scan overlaps 48 hours for delayed indexing and pages in batches of 100; extended downtime is covered from the previous cursor. Matches indexed more than 48 hours late are outside the overlap. Riot controls how much historical data remains available. Only one check per member runs at once.

All Riot calls share one serialized limiter (at least 1.2 seconds between requests) and honor application/method limit headers, 429 Retry-After, bounded exponential backoff and request timeouts. A single API key must not be shared with another deployment: independent processes would not share the limiter. The bot holds a PostgreSQL session advisory lock so only one instance runs against its database. Keep one replica; large communities need a shared external limiter and partitioned polling. Cooldowns survive restarts.

Roles are reconciled from committed rank/position, removing other mapped rank/position roles and preserving unrelated roles. Role failures retry without replaying quest credit. Rewards survive restarts. Announcements have **at-least-once** delivery: a crash after Discord accepts a message but before the database acknowledges it may duplicate the announcement. Quest credit and rank progression remain exactly once. A missing channel or bad hierarchy keeps rewards pending for repair.

To change role IDs after members exist, first plan an operator migration: remove retired roles from affected members, update mappings transactionally, and mark members `roleDirty`. The CLI blocks accidental remapping that would orphan old reward roles. Do not delete progression records merely to rename roles; role names can be changed in Discord without changing IDs.

## Validation and deployment

```powershell
npm run db:generate
npm run typecheck
npm run lint
npm test
npm run build
npm run format:check
```

Tests use mocked Riot HTTP responses, rule and eligibility cases, duplicate-processing checks, role/permission checks, and a PostgreSQL-compatible PGlite runtime to execute the actual migration and verify persistence constraints/rollback. They require no live credentials. Docker and real Discord/Riot connectivity require a configured deployment and are separate from automated validation.

For a server deployment, copy this repository to a host running Docker, provision `.env` there, create server JSON, and follow the Docker quick start. Build and run the pinned lockfile, keep one bot replica, back up PostgreSQL, and use `docker compose up -d --build bot` for updates. The container runs as a non-root user. Development tooling remains available in the image for migration, registration and configuration commands. Never expose the database publicly. Use environment injection or a secret manager in production; `.env` is ignored by Git and Docker builds. `.env.example` contains placeholders only.

Inspect `docker compose logs bot` for sanitized status messages. Detailed secrets and raw Discord/Riot request objects are never logged. To diagnose pending roles, run `/admin sync-roles`; for expired Riot keys, renew the environment key and recreate the bot container. When redeploying changed environment values, use `docker compose up -d --force-recreate bot`.

## Official references

Consulted during implementation on 2026-09-14:

- [Riot League documentation: Riot IDs, routing and RSO](https://developer.riotgames.com/docs/lol)
- [Riot API reference: Account-V1 and Match-V5](https://developer.riotgames.com/apis)
- [Riot key types and rate limits](https://developer.riotgames.com/docs/portal)
- [Discord application commands](https://docs.discord.com/developers/interactions/application-commands)
- [Discord role permissions](https://docs.discord.com/developers/platform/server-and-channel-management)
- [discord.js runtime requirements](https://discord.js.org/docs/packages/discord.js/main)
- [Stable Prisma v7 CLI](https://docs.prisma.io/docs/cli/v7)

Prisma CLI is explicitly pinned to stable 7.10.0, matching Prisma Client, because the registry's `latest` CLI tag pointed to an 8.0 release candidate. `package-lock.json` pins the complete tested dependency tree.

Rift Quests is not endorsed by Riot Games and does not reflect the views or opinions of Riot Games or anyone officially involved in producing or managing Riot Games properties. Riot Games and all associated properties are trademarks or registered trademarks of Riot Games, Inc.
