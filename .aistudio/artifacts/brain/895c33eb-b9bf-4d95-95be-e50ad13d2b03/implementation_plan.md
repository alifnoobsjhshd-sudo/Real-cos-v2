# Discord Bot Enhancement & Repository Migration

Implementation plan for importing the repository `Real-cos` and addressing all three requested tasks: persisting bot status/visibility across restarts, updating the help embed footer branding, and fixing the ticket creation flow with button-driven panel channel selection.

### User Review & Critical Decisions

> [!IMPORTANT]
> The following product decisions were confirmed during the clarification interview:
> - **Ticket Panel Channel Selection**: Users will configure the channel where the ticket panel embed is posted via an interactive button in the setup message that prompts the user in chat to type or mention the channel (`#tickets` or channel ID), instead of passing it as a slash/prefix command argument.
> - **Status & Visibility Persistence**: Both the custom activity and bot visibility status (`online`, `idle`, `dnd`, `invisible`) will be saved permanently to SQLite (`db/branding.db`), ensuring they survive restarts and prevent the background rotating presence loop from resetting them.
> - **Codebase Structure**: Preserves the complete Python Discord bot codebase natively.

---

### 1. Overview & Core Concept

- **What It Does**: Enhances the multi-feature Discord bot codebase with reliable presence persistence across reboots, updated project branding, and a streamlined, error-resilient ticket management system.
- **Target Audience**: Discord server administrators, staff teams, and members interacting with support tickets and bot commands.
- **Key Value**:
  - Eliminates status regression upon bot restart or periodic presence updates.
  - Consistent branding across all help embeds (`A project by ZENOX`).
  - Seamless ticket creation where members always receive prompt feedback, ticket channels are properly configured with send/embed permissions, and ticket setup requires zero command parameters.

---

### 2. User Experience & Visual Design

#### Flow 1: Bot Status & Visibility Customization
1. Admin runs `/global_customization` or the owner customization panel.
2. Under Bot Presence, admin selects status visibility (`🟢 Online`, `🟡 Idle`, `🔴 Do Not Disturb`, `⚫ Invisible`).
3. The selected visibility is immediately applied to Discord and committed to the database.
4. On bot reboots or background presence cycles, the saved status is read from SQLite and maintained automatically.

#### Flow 2: Help Embed Footer Branding
1. A user executes `>help` or `/help`.
2. The interactive help menu displays the bot categories and command descriptions.
3. The embed footer displays `A project by ZENOX` instead of `by ZENOX` across both the home page and category detail views.

#### Flow 3: Ticket Setup & Creation
1. Admin runs `/ticket setup` or `>ticket setup` without any mandatory channel argument.
2. The setup interactive embed displays a **Panel Channel** button alongside Title, Description, Banner, Thumbnail, and Staff Role buttons.
3. Clicking **Panel Channel** prompts the admin in chat to mention the destination channel (`#support`). The preview embed updates to reflect the selected channel.
4. When a user clicks a ticket category button on the panel, the bot:
   - Defers the interaction with an ephemeral response.
   - Creates the channel with comprehensive permission overwrites (`send_messages`, `embed_links`, `view_channel`, `read_message_history` for both bot and ticket creator).
   - Posts the ticket embed with action buttons and pings.
   - Reliably sends the user an ephemeral confirmation link to their new ticket channel.

---

### 3. Key Product Decisions & Trade-Offs

- **Decision 1: Storage for Bot Visibility**
  - *Chosen Approach*: Add `bot_status` table (`id INTEGER PRIMARY KEY, status TEXT`) inside `db/branding.db`, alongside the existing `bot_activity` table.
  - *Why*: Groups bot appearance and branding data cleanly in SQLite with asynchronous `aiosqlite` access.
  - *Alternatives Considered*: Storing in a flat text file or `.env` — rejected because `aiosqlite` is already integrated for bot branding and activities.

- **Decision 2: Ticket Channel Selection UX**
  - *Chosen Approach*: Button in `PanelEmbedSetupView` triggering an interactive chat prompt (`_prompt`) that parses channel mentions (`<#id>`) and raw numerical IDs.
  - *Why*: Directly aligns with user requirement while maintaining consistent UX with the existing Title, Description, and Staff Role buttons in the setup panel.
  - *Alternatives Considered*: Channel select menu — user explicitly preferred the chat prompt option.

- **Decision 3: Permission Overwrite Hardening in Ticket Channels**
  - *Chosen Approach*: Explicitly grant `send_messages=True`, `embed_links=True`, `read_message_history=True`, `attach_files=True`, `manage_channels=True`, and `manage_messages=True` to `guild.me`.
  - *Why*: Resolves the root cause of the silent ticket failure where Discord category permissions denied sending messages to `@everyone`, causing `ch.send()` to fail with 403 Forbidden.

---

### 4. Technical Architecture & Data Strategy

```
┌──────────────────────────────────────────────────────────────┐
│                      Discord Gateway                         │
└──────────────┬───────────────────────────────┬───────────────┘
               │ Interaction / Slash           │ Events / Ready
               ▼                               ▼
┌──────────────────────────────┐ ┌─────────────────────────────┐
│       Ticket System          │ │      Core Bot (Zyrox)       │
│  cogs/commands/ticket.py     │ │        core/zyrox.py        │
│                              │ │                             │
│ • setup command (no channel) │ │ • setup_hook: loads saved   │
│ • PanelEmbedSetupView button │ │   visibility & activity     │
│ • Channel mention parser     │ │ • status_task: preserves    │
│ • Robust _create_ticket_flow │ │   saved visibility status   │
│ • Ephemeral followup reply   │ └──────────────┬──────────────┘
└──────────────┬───────────────┘                │
               │                                │
               │ Reads / Writes                 │ Reads / Writes
               ▼                                ▼
┌──────────────────────────────┐ ┌─────────────────────────────┐
│         ticket.db            │ │         branding.db         │
│  ticket_panels, categories,  │ │  bot_activity, bot_status   │
│  open_tickets, counts        │ │  (aiosqlite async helpers)  │
└──────────────────────────────┘ └─────────────────────────────┘
```

#### Files to Modify
1. **`utils/branding.py`**:
   - Create `bot_status` table in database initialization.
   - Implement `get_bot_status()`, `set_bot_status()`, and `clear_bot_status()`.
2. **`cogs/commands/botcustomization.py`**:
   - Update `select_status` to call `await set_bot_status(chosen)`.
   - Update `GlobalBotProfileView` initialization to preload saved status text.
3. **`core/zyrox.py`**:
   - In `setup_hook()`, retrieve and apply the saved status on startup.
   - In `status_task()`, pass `status=saved_status` so periodic presence changes do not overwrite the visibility setting.
4. **`utils/help.py`**:
   - Update embed footer text to `"A project by ZENOX"` in `HelpView.home` and command embed builders.
5. **`cogs/commands/ticket.py`**:
   - Update `@ticket.command(name="setup")` signature to remove the required `channel` argument.
   - Update `SetupLandingView` to handle optional/None channel.
   - Add `Panel Channel` button to `PanelEmbedSetupView` with chat prompt to set `self.panel_channel`.
   - Update preview embed to display the selected panel channel.
   - In `_create_ticket_flow`:
     - Fix `guild.me` and `user` permission overwrites so the bot has explicit `send_messages` and `embed_links`.
     - Sanitize banner and thumbnail URLs to prevent HTTP 400 Bad Request errors.
     - Add error handling and guarantee ephemeral user confirmation via `inter.followup.send`.
