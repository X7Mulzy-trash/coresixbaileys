# <div align='center'>Baileys - Javascript WhatsApp Web API</div>

<div align='center'>

![NPM Downloads](https://img.shields.io/npm/dw/baron-baileys-v2?label=npm&color=%23CB3837)
![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/Barons-Team/baron-baileys-v2)
![GitHub License](https://img.shields.io/github/license/Barons-Team/baron-baileys-v2)
![GitHub Repo stars](https://img.shields.io/github/stars/Barons-Team/baron-baileys-v2)
![GitHub forks](https://img.shields.io/github/forks/Barons-Team/baron-baileys-v2)

</div>

#### Liability and License Notice

Baileys and its maintainers cannot be held liable for misuse of this application, as stated in the [MIT license](https://github.com/Barons-Team/baron-baileys-v2/LICENSE).
The maintainers of Baileys do not in any way condone the use of this application in practices that violate the Terms of Service of WhatsApp. The maintainers of this application call upon the personal responsibility of its users to use this application in a fair way, as it is intended to be used.

##

- Baileys does not require Selenium or any other browser to be interface with WhatsApp Web, it does so directly using a **WebSocket**.
- Not running Selenium or Chromimum saves you like **half a gig** of ram :/
- Baileys supports interacting with the multi-device & web versions of WhatsApp.

## Performance — Rust WASM Core ⚡

All cryptographic and binary encoding operations run in a **compiled Rust/WebAssembly module** ([whatsapp-rust-bridge](https://github.com/7ucg/whatsapp-rust-bridge)), giving significantly faster throughput compared to pure-JS alternatives — especially under high message volume.

**What runs in Rust:**

| Operation | Was | Now |
|-----------|-----|-----|
| AES-256-GCM / CBC / CTR | `node:crypto` | Rust WASM |
| HMAC-SHA256 / SHA256 / MD5 | `node:crypto` | Rust WASM |
| HKDF key derivation | `baron-util-js-hkdf` | Rust WASM |
| Curve25519 (ECDH, sign, verify) | `libsignal-node` JS | Rust WASM |
| Signal SessionCipher / Builder / Record | `libsignal-node` JS | Rust WASM |
| WABinary encode (`encodeBinaryNode`) | Pure JS | Rust WASM |
| LT-Hash (app state integrity) | Pure JS | Rust WASM |
| Group cipher (encrypt / decrypt) | Pure JS crypto | Rust WASM |

**Dependencies removed:** `libsignal-node`, `baron-util-js-hkdf` — replaced entirely by the Rust bridge.

**No configuration needed.** The WASM module is bundled and initialized automatically on startup. Works seamlessly with the built-in Antiban middleware.

> **Server deployment:** No Rust toolchain required — the compiled `.wasm` binary ships with the package.

## Antiban Protection ⚠ (Test Phase)

Built-in anti-ban middleware is automatically applied to every socket. See [ANTIBAN.md](./ANTIBAN.md) for usage, configuration, and diagnostics.

## Install

Use the stable version:

```
yarn add baron-baileys-v2
```

Use the edge version (no guarantee of stability, but latest fixes + features)

```
yarn add github:Barons-Team/baron-baileys-v2
```

Then import your code using:

```ts
import makeWASocket from 'baron-baileys-v2'
```

## What's New (Usernames)

Full implementation of the WhatsApp username protocol, reverse-engineered from WhatsApp 2.26.17.2. See [USERNAME.md](./USERNAME.md) for full documentation and setup instructions.

- **`checkUsername(username, includeSuggestions?)`** — check whether a `@username` is available before setting it. Returns `{ available, suggestions, rejectionReasons, suggestionsEligible }`. Confirmed from `C1568872p.A01()` and `C164057Wg.java` (`xwa2_username_check` data path).
- **`setUsername(username, options?)`** — set your own username via `w:mex` `UsernameSet`. Options: `source` (`USER_INPUT` / `SUGGESTION` / `FB` / `IG`), `sessionId`, `pin`.
- **`deleteUsername()`** — unset your username. Sends `username: null` which triggers the server-side delete path (confirmed `C1568872p.java:24`).
- **`getMyUsername()`** — fetch your current username via `w:mex` `UsernameGet`.
- **`setUsernamePin(pin)`** — protect your username with a PIN, or delete it by passing `null`. Uses `w:mex` operation `UsernamePinSet` (confirmed `MexUsernamePinProtocolApi.java`).
- **`findUserByUsername(username, pin?)`** — look up a contact's JID by their `@username` via USync. Supports PIN-protected usernames. Returns `{ jid, contact }`.
- **`fetchContactUsernames(...jids)`** — batch-fetch the username of multiple contacts by JID via USync `UsernameProtocol`.
- **`USERNAME_QUERY_IDS`** — object holding the `w:mex` query IDs (must be filled in from a live session capture — see [USERNAME.md](./USERNAME.md#setup-query-ids)).
- **`USERNAME_CHECK_RESULT`**, **`USERNAME_SOURCE`** — enums confirmed from APK (`EnumC141106Vn`, `C1568872p.java`).

## What's New (Interop — BirdyChat & Haiket)

Full support for the WhatsApp DMA Interoperability Protocol (`w:interop`). Allows messaging with users on third-party platforms directly from a WhatsApp account. See [INTEROP.md](./INTEROP.md) for full documentation.

- **Automatic init on connect** — integrators are fetched, TOS accepted, and opt-in sent in parallel with other init queries. No manual setup required.
- **`fetchIntegrators()`** — returns all available integrators with `id`, `name`, `status` (`active` | `onboarding` | `removed`), `identifierType` (`email` | `pn` | `username`), `optedIn`, and `features`.
- **`resolveInteropUser(externalId, integratorId)`** — look up a single user by email (BirdyChat) or phone number (Haiket). Returns their interop JID (`12-…@interop`) or an error object.
- **`resolveInteropUsers([...])`** — batch lookup of up to 256 users in a single IQ request.
- **`optInIntegrators(ids)` / `optOutIntegrators(ids)`** — opt in or out of specific integrators.
- **`getReachabilitySettings()` / `setReachabilitySettings(users, enabled)`** — interop presence/reachability subscription mechanism (replaces XMPP presence for interop contacts).
- **`blockInteropUser(jid)` / `unblockInteropUser(jid)`** — block/unblock via the dedicated `w:interop` blocklist (separate from the regular WA blocklist).
- **`reportInteropSpam(jid)`** — report an interop contact as spam.
- **`trustInteropContact(jid)`** — mark an interop JID as `trusted_contact` in privacy tokens (called automatically after the first send in WA).
- **`isInteropUser(jid)`** — new JID utility exported from `WABinary`, returns `true` for `@interop` JIDs.
- **Incoming message support** — `decodeMessageNode` now correctly handles messages received from interop JIDs (`@interop`) as `chat` type, routed through the normal `messages.upsert` event.
- **`profilePictureUrl`** — works out of the box for interop JIDs (no TC token attached, matching the WA protocol).
- **Constants** `INTEGRATOR_BIRDYCHAT` (12) and `INTEGRATOR_HAIKET` (13) exported directly from the socket.

## What's New (Identity + Meta AI + History Sync + New Message Types)

- **TC Token (Privacy Token) system — full upstream parity**
  - Tokens are now stored under LID instead of PN where applicable (AB prop 14303).
  - Expiry is enforced using a rolling 28-day / 4-bucket window; expired tokens are cleared on send.
  - Fire-and-forget token issuance after every outgoing 1:1 message, with in-flight deduplication via `inFlightTcTokenIssuance`.
  - Token re-issuance triggers automatically when a peer's device identity changes (`reissueTcTokenAfterIdentityChange`).
  - Privacy token notifications (`privacy_token`) are now processed via `storeTcTokensFromIqResult` with `sender_lid` support.
  - TC tokens from history sync are stored and indexed (`storeTcTokensFromHistorySync`).
  - A persistent `__index` key tracks all storage JIDs for daily cross-session pruning (`pruneExpiredTcTokens`).
  - Profile picture IQs and presence subscribes are gated by server-assigned AB props (`serverProps.profilePicPrivacyToken`, `serverProps.privacyTokenOn1to1`).
  - New exports: `readTcTokenIndex`, `buildMergedTcTokenIndexWrite`, `isTcTokenExpired`, `shouldSendNewTcToken`, `resolveTcTokenJid`, `resolveIssuanceJid`, `storeTcTokensFromIqResult`, `TC_TOKEN_INDEX_KEY`.
- **App state sync — missing-key handling (Blocked state)**
  - Collections blocked on a missing app state sync key are parked in `blockedCollections` instead of being retried indefinitely.
  - When the key arrives via `APP_STATE_SYNC_KEY_SHARE`, blocked collections are re-synced automatically.
  - New utilities: `isMissingKeyError`, `isAppStateSyncIrrecoverable`, `MAX_SYNC_ATTEMPTS`, `ensureLTHashStateVersion` (exported from `chat-utils`).
- **Server AB props (`serverProps`) on the socket**
  - `chats.makeChatsSocket` now fetches and exposes `serverProps` containing three live AB flags:
    - `privacyTokenOn1to1` (prop 10518) — whether tctoken is required on all 1:1 messages
    - `profilePicPrivacyToken` (prop 9666) — whether profile-picture IQs require a tctoken
    - `lidTrustedTokenIssueToLid` (prop 14303) — whether tokens are issued to LID or PN
- **History sync completion tracking**
  - `historySyncStatus` tracks `initialBootstrapComplete` and `recentSyncComplete` in-memory.
  - `messaging-history.status` event fires with `{ syncType, status: 'complete'|'paused', explicit }` for `INITIAL_BOOTSTRAP` and `RECENT` sync types.
  - A 120 s paused-timeout fires a `status: 'paused'` event if progress stalls mid-RECENT sync (`HISTORY_SYNC_PAUSED_TIMEOUT_MS`).
  - On reconnection with existing data (`accountSyncCounter > 0`), the 20 s AwaitingInitialSync wait is skipped.
- **Username support**
  - `USyncUsernameProtocol` — new USync protocol for username lookups.
  - `USyncQuery.withUsernameProtocol()` and `USyncUser.withUsername()` / `withUsernameKey()` added.
  - `USyncContactProtocol.getUserElement` supports `username`, `usernameKey`, and `lid` lookups in addition to phone.
  - Group metadata now includes `ownerUsername`, `subjectOwnerUsername`, `descOwnerUsername`, and `username` per participant.
  - `authorUsername` propagated through group-participant events (`group-participants.update`, `groups.update`, `group.join-request`).
  - `participantUsername` and `remoteJidUsername` exposed on message keys.
  - `username` field carried through history sync, sync-action, and lidContactAction contacts events.
- **Server error codes**
  - `SERVER_ERROR_CODES` exported from `decode-wa-message`: `MissingTcToken: '463'`, `SmaxInvalid: '479'`.
  - ACK error handler distinguishes 463 (account restricted / missing tctoken) and 479 (stale session / SMAX_INVALID) with specific log messages and no-retry behaviour for 463.
- **Identity change handler**
  - `ctx.onBeforeSessionRefresh?.(from)` callback fires before `assertSessions`, enabling fire-and-forget tctoken re-issuance in parallel with the session refresh.
- **Call latency**
  - `relaylatency` call status now reads `latency` / `latency_ms` / `latency-ms` attributes and sets `call.latencyMs`.
- **Rich AI Response Message support**
  - Send Meta AI-style rich responses via `richResponse` with optional syntax-highlighted code blocks.
  - Built-in tokenizer supports JavaScript, TypeScript, and Python.
  - Uses `botForwardedMessage` → `richResponseMessage` → `unifiedResponse` proto chain.
  - Extended: `table` (rows/cells), `latex` (expressions), `map` (lat/lng/annotations), `imageUrl`, `imageUrls` (grid).
- **New WA 2.3000+ message types**
  - `statusNotification` — status add-yours / reshare / question-answer-reshare events.
  - `statusQuestionAnswer` — user answered a status question.
  - `questionResponse` — direct response to a question message.
  - `statusQuoted` — quote a status with a custom type.
  - `statusStickerInteraction` — react to a status with a sticker.
  - `newsletterFollowerInvite` — invite a user to follow a newsletter.
  - `messageHistoryNotice` — notify about message history metadata.
- **Scheduled Calls** (`scheduledCall` / `editScheduledCall`)
  - Create scheduled voice/video calls with `scheduledCall: { title, isVideo, scheduledAt }`.
  - Cancel scheduled calls with `editScheduledCall: { key }`.
  - Enums: `ScheduledCallType` (VOICE, VIDEO), `ScheduledCallEditType` (CANCEL).
- **Event Invitations** (`eventInvite`)
  - Send event invite messages referencing an existing event by ID.
  - `eventInvite: { id, title, startDate, endDate, text, thumbnail }`.
- **Comment Messages** (`comment`)
  - Thread comments on any message via `comment: { targetMessageKey, message }`.
- **Split Payment** (`splitPayment`)
  - Bill-splitting with `splitPayment: { totalAmount, description, participants: [{ jid, amount }] }`.
  - Per-participant status tracked via `SplitPaymentStatus` enum (PENDING, PAID).
- **P2P Payment Reminders** (`p2pPaymentReminder`)
  - Recurring payment reminders with `p2pPaymentReminder: { amount, frequency, state, description, receiverJid }`.
  - Enums: `P2PPaymentReminderFrequency` (WEEKLY, BIWEEKLY, MONTHLY, CUSTOM), `P2PPaymentReminderState`.
- **Conditional Reveal / Scheduled Messages** (`conditionalReveal`)
  - Send a message that reveals at a scheduled time: `conditionalReveal: { revealKeyId, messageSecret }`.
  - Type via `ConditionalRevealType` enum (SCHEDULED_MESSAGE).
- **Call Log Messages** (`callLog`)
  - Log a call event with outcome, duration, and participant list.
  - `callLog: { isVideo, outcome, durationSecs, callType, participants }`.
  - Enums: `CallLogOutcome` (CONNECTED, MISSED, FAILED, REJECTED…), `CallLogType` (REGULAR, SCHEDULED_CALL, VOICE_CHAT).
- **Status Mention** (`statusMention`)
  - Wrap a message as a status mention: `statusMention: { message: { ... } }`.
- **New FutureProof message wrappers**
  - `question` / `questionMessage` — send a question prompt message.
  - `questionReply` / `questionReplyMessage` — reply to a question message.
  - `statusAddYours` — "Add yours" status interaction.
  - `eventCoverImage` — event cover image wrapper.
  - `spoilerMessage` / `spoiler` — spoiler-tagged content.
  - `lottieStickerMessage` / `lottieSticker` — Lottie animated sticker.
  - `groupStatusV2` / `groupStatusMessageV2` — improved group status update.
  - `newsletterAdminProfile` / `newsletterAdminProfileMessage` — admin profile update.
  - `newsletterAdminProfileV2` / `newsletterAdminProfileMessageV2` — admin profile update v2.
  - `botTask` / `botTaskMessage` — bot task wrapper.
- **New socket methods**
  - `sock.renameAIThread(jid, title)` — rename an AI chat thread via app-state sync.
  - `sock.pinThreadMessage(jid, messageId, pinned?)` — pin/unpin a message in a thread.
  - `sock.updatePrivateProcessingSetting(enabled)` — toggle AI end-to-end private processing.
  - `sock.updateBioPrivacy(value)` — set who can see your About/bio (`'all'|'contacts'|'none'`).
  - `sock.blockBot(jid)` / `sock.unblockBot(jid)` — block/unblock a bot.
  - `sock.getBotProfile(jid)` — fetch bot details (name, personaId, description).
- **New sync action processors** — `aiThreadRenameAction`, `threadPinAction`, `privateProcessingSettingAction`, `settingsSyncAction`, `musicUserIdAction`, `avatarUpdatedAction` now emit `settings.update` or `chats.update` events.
- **New enum: `PrivateProcessingStatus`** — `UNDEFINED`, `ENABLED`, `DISABLED`.
- **Round 9 — Image/Video Intelligence, Bot Metadata, Business Context**
  - **`aiGenerated: true`** on images → sets `imageSourceType = AI_GENERATED`.
  - **`aiModified: true`** on images → sets `imageSourceType = AI_MODIFIED`.
  - **`imageSourceType`** shortcut → explicitly set IMAGE source type on images.
  - **`qrUrl`** on images → embed a QR code URL in an image message.
  - **`videoContentUrl`** on text messages → attach a direct video URL to a link preview.
  - **`endCardTiles`** → add video end-card tiles (username, caption, thumbnail) to extended text.
  - **`musicMetadata`** → attach `EmbeddedMusic` (title, author, songId, isExplicit, timing) to link previews.
  - **`businessInteractionPills`** → Business CTA pill buttons (CATALOG, CALL, BOOK_APPOINTMENT, BESTSELLERS, MENU, OFFERS…) with entry point.
  - **`dataSharingContext`** → Meta media disclosure flags for business messages.
  - **`botSession`** → populate `BotMetadata.sessionMetadata` (sessionId + sessionSource) when messaging bots.
  - **`botReminder`** → create/update/delete bot reminders via `BotMetadata.reminderMetadata`.
  - **`botPlugin`** → attach search/reels plugin metadata via `BotMetadata.pluginMetadata` (provider: BING/GOOGLE/SUPPORT).
  - **`status.psa`** event → emitted when a StatusPSA (public service announcement) is received.
  - **9 new enums**: `ImageSourceType`, `BotSessionSource`, `BotReminderAction`, `BotReminderFrequency`, `BotPluginType`, `BotPluginSearchProvider`, `BizInteractionPillType` (11 values), `PeerDataOperationRequestType` (14 values), `SocialMediaPostType` (REEL, LIVE_VIDEO, LONG_VIDEO, SINGLE_IMAGE, CAROUSEL).
- **Round 8 — Carousel, Bot Capabilities, MessageContextInfo, Store**
  - **Carousel message** (`carousel`) — interactive card carousel with multiple `InteractiveMessage` cards, horizontal scroll or album layout.
  - **`botCapabilities`** — advertise client capabilities to bots (RICH_RESPONSE_TABLE, RICH_RESPONSE_LATEX, RICH_RESPONSE_MAPS, etc.) via `messageContextInfo.botMetadata.capabilityMetadata`.
  - **`messageAssociation`** — link a message to a parent (album, HD video, motion photo, poll, sticker annotation, etc.) via `MessageContextInfo.messageAssociation`.
  - **`threadId`** — attach an AI thread ID (AI_THREAD or VIEW_REPLIES) to a message via `MessageContextInfo.threadId`.
  - **`featureEligibilities`** — control per-message features: `cannotBeReactedTo`, `cannotBeRanked`, `canBeReshared`, `canReceiveMultiReact`.
  - **`ChatThemeSetting` extended** — full wallpaper type support: `solidColor` (colorLight/colorDark), `stockImage` (id/dimLevel), `defaultWallpaper` (isDoodleEnabled), `customImage`.
  - **`make-in-memory-store` upgraded** — handles 10 new events: `settings.update`, `labels.reorder`, `call.scheduled`, `call.schedule-cancelled`, `payment.split`, `message.comment`, `poll.add-option`, `chats.lock`, `messaging-history.status`.
  - **New enums**: `BotCapabilityType` (62 values), `AssociationType` (21 values), `ThreadType`, `CarouselCardType`.
- **Round 7 — Sync Actions, Contacts, Protocol, Context**
  - **20+ new `processSyncAction` handlers**: `recentEmojiWeights`, `stickerAction`, `userStatusMute`, `chatAssignment`, `callLogAction`, `deleteCallLog`, `labelReorder`, `paymentInfo`, `noteEdit`, `favoritesAction`, `usernameChatStartMode`, `aiFeatures`, `statusPostOptIn`, `subscriptionsSyncV2`, `newsletterSavedInterests`, `interactiveMessageAction`, `outContact`, `businessBroadcastList`, `customerData`, `autoOrganizeBusinessChat`, `chatLockSettings`, `agentAction`, `nuxAction`.
  - **New chatModify ops**: `muteStatus`, `favorite`, `reorderLabel`, `deleteCallLog`, `noteEdit`.
  - **New socket methods**: `muteContactStatus(jid)`, `toggleFavorite(jid)`, `reorderLabels(ids)`, `deleteCallLog(callId)`, `setChatNote(jid, note)`, `deleteChatNote(jid)`.
  - **New ProtocolMessage handlers** (incoming): `REQUEST_WELCOME_MESSAGE` → `bot.welcome-request`; `BOT_MEMU_ONBOARDING_MESSAGE` → `bot.memu-onboarding`; `STATUS_MENTION_MESSAGE` → `status.mention`; `AI_PSI_METADATA` → `bot.psi-metadata`; `AI_QUERY_FANOUT` → `bot.query-fanout`; `AI_MEDIA_COLLECTION_MESSAGE` → `bot.media-collection`.
  - **New message types**: `aiMediaCollection` / `aiMediaCollectionMessage` — start an AI media collection session.
  - **New contextInfo shortcuts**: `forwardOrigin` (CHAT/STATUS/CHANNELS/META_AI/UGC), `statusAudienceMetadata` (closeFriends, listName).
  - **Video flag**: `aiGenerated: true` sets `VideoMessage.videoSourceType = AI_GENERATED`.
  - **New WebMessageInfo processing**: `quarantinedMessage` → `message.quarantined`; `interactiveMessageAdditionalMetadata.isGalaxyFlowCompleted` → `galaxy.flow.completed`; `ephemeralExpirationTimestamp` → `chat.ephemeralExpirationTimestamp`.
  - **New enums**: `ForwardOrigin`, `StatusAudienceType`, `VideoSourceType`, `PollType`.
- **Round 6 — Polls, Utilities, Communities, Events**
  - **Quiz Poll support**: `poll.type: 'quiz'` with `correctAnswer`, `hideParticipantName`, `allowAddOption`, `endTime`.
  - **Poll V5 / V6**: select via `poll.v5: true` or `poll.v6: true`.
  - **`pollResultSnapshotMessageV3`**: `pollResult.v3: true` sends the V3 snapshot proto.
  - **`afterReadDuration`**: message flag — sets `contextInfo.afterReadDuration` + `messageContextInfo.messageAddOnDurationInSecs` (like a timed-read message).
  - **`getAggregateVotesInPollMessage`** — now handles V4/V5/V6 poll options.
  - **New utilities exported**:
    - `isScheduledMessage(msg)` — true if message has a `scheduledMessageMetadata`
    - `getScheduledMessageTime(msg)` — returns the reveal `Date` or `null`
    - `getMessagePaymentInfo(msg)` — extracts `PaymentInfo` from `WebMessageInfo`
    - `getMessageCommentMetadata(msg)` — extracts comment thread metadata
    - `getMessageAddOns(msg)` — returns `messageAddOns[]` array (reactions, pins, etc.)
    - `getPollCorrectAnswer(pollMsg)` — returns correct answer string for quiz polls
  - **New Community methods**: `communityUpdatePicture`, `communityRemovePicture`, `communitySettingUpdate`, `communityDeactivate`, `communityGetInviteInfo`, `communityToggleEphemeral`.
  - **New incoming event handlers** in `process-message.js`:
    - `poll.add-option` — when someone adds a poll option
    - `call.scheduled` — when a scheduled call is created
    - `call.schedule-cancelled` — when a scheduled call is cancelled
    - `payment.split` — split payment message received
    - `payment.reminder` — P2P payment reminder received
- **Round 5 — Messaging, Chat Management, Newsletter**
  - `bcall` / `bcallMessage` — broadcast call message (audio or video) with sessionId/masterKey.
  - `liveLocationUpdate` — update a live location with new coordinates/accuracy/speed.
  - `stopLiveLocation` — revoke/stop a live location share.
  - Incoming: `keepInChatMessage`, `encReactionMessage`, `commentMessage`, `encCommentMessage`, `bcallMessage` all emit proper events.
  - `markChatAsUnread(jid)` — explicitly mark a chat as unread.
  - `setChatEphemeral(jid, durationSecs)` — set per-chat disappearing message timer.
  - `silenceChat(jid, silent, until?)` — mute a chat permanently or until a timestamp.
  - `fetchBotProfiles(jids)` — batch-fetch bot profiles via USyncBotProfileProtocol.
  - Newsletter: `newsletterUpdateCategory`, `newsletterUpdateSettings`, `newsletterPromoteAdmin`, `newsletterViewStats`, `newsletterSendPost`, `newsletterPinMessage`, `newsletterUnpinMessage`.
  - `USyncBotProfileProtocol` now publicly exported from `baron-baileys-v2`.
- **New enums**
  - `BCallMediaType` — `UNKNOWN`, `AUDIO`, `VIDEO`
  - `SecretEncType` — `UNKNOWN`, `EVENT_EDIT`, `MESSAGE_EDIT`, `MESSAGE_SCHEDULE`, `POLL_EDIT`, `POLL_ADD_OPTION`
  - `MediaRetryResultType` — `GENERAL_ERROR`, `SUCCESS`, `NOT_FOUND`, `DECRYPTION_ERROR`
  - `StatusAttributionType` — 12 values (`RESHARE`, `EXTERNAL_SHARE`, `MUSIC`, `STATUS_MENTION`, `GROUP_STATUS`, `RL_ATTRIBUTION`, `AI_CREATED`, …)
- **Round 4 — Protocol-level message types**
  - `keepInChat` / `keepInChatMessage` — keep or un-keep a message for all (KeepInChatMessage proto).
  - `botFeedback` / `botFeedbackMessage` — thumbs-up/down on a bot response (ProtocolMessage.BOT_FEEDBACK_MESSAGE).
  - `pollAddOption` / `pollAddOptionMessage` — add a new option to an existing poll.
  - `chatTheme` / `chatThemeMessage` — change the chat color scheme (ProtocolMessage.CHAT_THEME_SETTING).
  - `stopGeneration` — cancel an in-progress AI generation (ProtocolMessage.STOP_GENERATION_MESSAGE).
  - `unscheduleMessage` — cancel a scheduled message (ProtocolMessage.MESSAGE_UNSCHEDULE).
- **New ProtocolMessage event handlers** (incoming messages now emit events):
  - `bot.feedback` — when a bot feedback message is received.
  - `cloud.thread.control` — Cloud API thread control notification (CONTROL_PASSED / CONTROL_TAKEN).
  - `chats.update` with `chatThemeSetting` — chat theme changed.
  - `bot.stop-generation` — AI generation stopped remotely.
  - `media.notify` — media notification (express-path URL update).
  - `reminder.update` — reminder message received.
  - `messages.update` with `scheduledMessageMetadata: null` — message unscheduled.
- **New group methods**
  - `sock.groupGetPastParticipants(jid)` — fetch list of past participants (left/removed) with reason & timestamp.
  - `sock.groupCreateSubgroup(communityJid, subject, participants?)` — create a subgroup inside a community.
  - `sock.groupLinkToCommunity(communityJid, groupJid)` — link an existing group to a community.
  - `sock.groupUnlinkFromCommunity(communityJid, groupJid)` — unlink a group from a community.
  - `sock.groupSetMemberLabel(jid, participantJid, label)` — tag a group member with a label.
  - `sock.groupGetLinkedSubgroups(communityJid)` — list all subgroups linked to a community.
- **New enums**
  - `KeepType` — `UNKNOWN`, `KEEP_FOR_ALL`, `UNDO_KEEP_FOR_ALL`
  - `BotFeedbackKind` — 14 values (POSITIVE, NEGATIVE_GENERIC, NEGATIVE_HELPFUL, NEGATIVE_ACCURATE, …)
  - `CloudAPIThreadControl` — `UNKNOWN`, `CONTROL_PASSED`, `CONTROL_TAKEN`, `INFO`
  - `MessageAddOnType` — `UNDEFINED`, `REACTION`, `EVENT_RESPONSE`, `POLL_UPDATE`, `PIN_IN_CHAT`
  - `ProtocolMessageType` — full enum of all 35 protocol message types
- **Extended History Sync** — `messaging-history.set` now includes:
  - `globalSettings` — font size, wallpaper, notification prefs, disappearing mode from `HistorySync.globalSettings`
  - `callLogRecords` — call history records (`CallLogRecord[]`) from `HistorySync.callLogRecords`
  - `recentStickers` — recent sticker metadata (`StickerMetadata[]`)
  - `pastParticipants` — past group participants with leave reason/time
  - `statusV3Messages` — status/story messages from initial bootstrap
  - `inlineContacts` — inline contact data with LID-PN mappings auto-extracted
- **New message types**
  - `tapLink: { title, url, buttonTitle }` — action-link message (tap to open URL)
  - `citation: { title, subtitle, text }` — AI citation / source reference
  - `embeddedMusic: { title, author }` — music attribution in text messages
- **New contextInfo flags**
  - `isSpoiler: true` — marks message content as spoiler (applies to any message type)
  - `isQuestion: true` — marks message as a question
- **New socket methods (chat management)**
  - `sock.updateChatLock(jid, locked)` — lock/unlock a chat (requires device password)
  - `sock.updateChatWallpaper(jid, wallpaper)` — set per-chat wallpaper (pass `null` to remove)
  - `sock.updateChatMediaVisibility(jid, visibility)` — set auto-download: `'default'|'on'|'off'`
- **New enums**
  - `PaymentInfoStatus` — full payment status (PROCESSING, SENT, COMPLETE, REFUNDED, EXPIRED…)
  - `PaymentInfoTxnStatus` — transaction status (32 values, INIT through REVERSAL_PENDING)
  - `VideoQuality` — `UNDEFINED`, `LOW`, `MID`, `HIGH`
  - `MediaVisibility` — `DEFAULT`, `OFF`, `ON`
  - `HistorySyncType` — `INITIAL_BOOTSTRAP`, `INITIAL_STATUS_V3`, `FULL`, `RECENT`, `PUSH_NAME`, `NON_BLOCKING_DATA`, `ON_DEMAND`
- **JID display normalization is now built in**
  - Runtime prefers PN JIDs (`@s.whatsapp.net`) over LID JIDs (`@lid`) in display/event paths when mapping data is available.
  - This includes message identity fields, mentions (`contextInfo.mentionedJid`), quote/reply participants, and call/event payload identity fields.
- **Meta AI `msmsg` decryption is now deterministic**
  - Decoding now uses a bounded strategy pipeline for known private/group Meta AI message families.
  - Broad brute-force loops were removed for better reliability and predictable behavior.
- **One-time initial full history sync is supported**
  - With fresh auth state, full history sync can run on first successful connect.
  - After completion, auth state persists it as done so reconnects do not repeat full sync automatically.
- **Optional LID/PN diagnostics**
  - Set `LID_PN_DEBUG=1` to trace LID -> PN normalization source selection.

# Index

- [What's New](#whats-new-identity--meta-ai--history-sync--new-message-types)
- [Connecting Account](#connecting-account)
  - [Connect with QR-CODE](#starting-socket-with-qr-code)
  - [Connect with Pairing Code](#starting-socket-with-pairing-code)
  - [Receive Full History](#receive-full-history)
- [Important Notes About Socket Config](#important-notes-about-socket-config)
  - [Caching Group Metadata (Recommended)](#caching-group-metadata-recommended)
  - [Improve Retry System & Decrypt Poll Votes](#improve-retry-system--decrypt-poll-votes)
  - [Receive Notifications in Whatsapp App](#receive-notifications-in-whatsapp-app)

- [Save Auth Info](#saving--restoring-sessions)
- [Handling Events](#handling-events)
  - [Core Events You Should Handle (Recommended)](#core-events-you-should-handle-recommended)
  - [Production Event Handling Checklist](#production-event-handling-checklist)
  - [Debugging Tips](#debugging-tips)
  - [Example to Start](#example-to-start)
  - [Decrypt Poll Votes](#decrypt-poll-votes)
  - [Summary of Events on First Connection](#summary-of-events-on-first-connection)
- [Implementing a Data Store](#implementing-a-data-store)
- [Whatsapp IDs Explain](#whatsapp-ids-explain)
- [Utility Functions](#utility-functions)
- [Sending Messages](#sending-messages)
  - [Non-Media Messages](#non-media-messages)
    - [Text Message](#text-message)
    - [Quote Message](#quote-message-works-with-all-types)
    - [Mention User](#mention-user-works-with-most-types)
    - [Forward Messages](#forward-messages)
    - [Location Message](#location-message)
    - [Live Location Message](#live-location-message)
    - [Contact Message](#contact-message)
    - [Reaction Message](#reaction-message)
    - [Pin Message](#pin-message)
    - [Keep Message](#keep-message)
    - [Poll Message](#poll-message)
    - [Poll Result Message](#poll-result-message)
    - [Call Message](#call-message)
    - [Event Message](#event-message)
    - [Order Message](#order-message)
    - [Product Message](#product-message)
    - [Payment Message](#payment-message)
    - [Payment Invite Message](#payment-invite-message)
    - [Admin Invite Message](#invite-admin-message)
    - [Group Invite Message](#group-invite-message)
    - [Share Phone Number Message](#share-phone-number-message)
    - [Request Phone Number Message](#request-phone-number-message)
    - [Buttons Reply Message](#buttons-reply-message)
    - [Buttons Message](#buttons-message)
    - [Buttons List Message](#buttons-list-message)
    - [Buttons Product List Message](#buttons-product-list-message)
    - [Buttons Cards Message](#buttons-cards-message)
    - [Buttons Template Message](#buttons-template-message)
    - [Buttons Interactive Message](#buttons-interactive-message)
    - [Buttons Interactive Message PIX](#buttons-interactive-message-pix)
    - [Buttons Interactive Message PAY](#buttons-interactive-message-PAY)
    - [Status Mentions Message](#status-mentions-message)
    - [Send Album Message](#send-album-message)
    - [Shop Message](#shop-message)
    - [Collection Message](#collection-message)
    - [Sticker Pack Message](#Sticker-Pack-Message)
    - [Rich AI Response Message](#rich-ai-response-message)
    - [Rich Composer Methods](#rich-composer-methods)
      - [Send Table](#send-table)
      - [Send List](#send-list)
      - [Send Code Block](#send-code-block)
      - [Send Latex](#send-latex)
      - [Send Latex Image](#send-latex-image)
      - [Send Latex Inline Image](#send-latex-inline-image)
      - [Send Rich Message](#send-rich-message)
      - [Capture & Resend Unified Response](#capture--resend-unified-response)
    - [Status Notification Message](#status-notification-message)
    - [Status Question Answer Message](#status-question-answer-message)
    - [Question Response Message](#question-response-message)
    - [Status Quoted Message](#status-quoted-message)
    - [Status Sticker Interaction Message](#status-sticker-interaction-message)
    - [Newsletter Follower Invite Message](#newsletter-follower-invite-message)
    - [Message History Notice](#message-history-notice)
  - [Sending with Link Preview](#sending-messages-with-link-previews)
  - [Media Messages](#media-messages)
    - [Gif Message](#gif-message)
    - [Video Message](#video-message)
    - [Audio Message](#audio-message)
    - [Image Message](#image-message)
    - [Ptv Video Message](#ptv-video-message)
    - [ViewOnce Message](#view-once-message)
- [Modify Messages](#modify-messages)
  - [Delete Messages (for everyone)](#deleting-messages-for-everyone)
  - [Edit Messages](#editing-messages)
- [Manipulating Media Messages](#manipulating-media-messages)
  - [Thumbnail in Media Messages](#thumbnail-in-media-messages)
  - [Downloading Media Messages](#downloading-media-messages)
  - [Re-upload Media Message to Whatsapp](#re-upload-media-message-to-whatsapp)
- [Reject Call](#reject-call)
- [Send States in Chat](#send-states-in-chat)
  - [Reading Messages](#reading-messages)
  - [Update Presence](#update-presence)
- [Modifying Chats](#modifying-chats)
  - [Archive a Chat](#archive-a-chat)
  - [Mute/Unmute a Chat](#muteunmute-a-chat)
  - [Mark a Chat Read/Unread](#mark-a-chat-readunread)
  - [Delete a Message for Me](#delete-a-message-for-me)
  - [Delete a Chat](#delete-a-chat)
  - [Star/Unstar a Message](#starunstar-a-message)
  - [Disappearing Messages](#disappearing-messages)
  - [Clear Messages](#clear-messages)
- [User Querys](#user-querys)
  - [Check If ID Exists in Whatsapp](#check-if-id-exists-in-whatsapp)
  - [Query Chat History (groups too)](#query-chat-history-groups-too)
  - [Fetch Status](#fetch-status)
  - [Fetch Profile Picture (groups too)](#fetch-profile-picture-groups-too)
  - [Fetch Bussines Profile (such as description or category)](#fetch-bussines-profile-such-as-description-or-category)
  - [Fetch Someone's Presence (if they're typing or online)](#fetch-someones-presence-if-theyre-typing-or-online)
- [Change Profile](#change-profile)
  - [Change Profile Status](#change-profile-status)
  - [Change Profile Name](#change-profile-name)
  - [Change Display Picture (groups too)](#change-display-picture-groups-too)
  - [Remove display picture (groups too)](#remove-display-picture-groups-too)
- [Groups](#groups)
  - [Create a Group](#create-a-group)
  - [Add/Remove or Demote/Promote](#addremove-or-demotepromote)
  - [Change Subject (name)](#change-subject-name)
  - [Change Description](#change-description)
  - [Change Settings](#change-settings)
  - [Leave a Group](#leave-a-group)
  - [Get Invite Code](#get-invite-code)
  - [Revoke Invite Code](#revoke-invite-code)
  - [Join Using Invitation Code](#join-using-invitation-code)
  - [Get Group Info by Invite Code](#get-group-info-by-invite-code)
  - [Query Metadata (participants, name, description...)](#query-metadata-participants-name-description)
  - [Join using groupInviteMessage](#join-using-groupinvitemessage)
  - [Get Request Join List](#get-request-join-list)
  - [Approve/Reject Request Join](#approvereject-request-join)
  - [Get All Participating Groups Metadata](#get-all-participating-groups-metadata)
  - [Toggle Ephemeral](#toggle-ephemeral)
  - [Change Add Mode](#change-add-mode)
- [Privacy](#privacy)
  - [Block/Unblock User](#blockunblock-user)
  - [Get Privacy Settings](#get-privacy-settings)
  - [Get BlockList](#get-blocklist)
  - [Update LastSeen Privacy](#update-lastseen-privacy)
  - [Update Online Privacy](#update-online-privacy)
  - [Update Profile Picture Privacy](#update-profile-picture-privacy)
  - [Update Status Privacy](#update-status-privacy)
  - [Update Read Receipts Privacy](#update-read-receipts-privacy)
  - [Update Groups Add Privacy](#update-groups-add-privacy)
  - [Update Default Disappearing Mode](#update-default-disappearing-mode)
- [Broadcast Lists & Stories](#broadcast-lists--stories)
  - [Send Broadcast & Stories](#send-broadcast--stories)
  - [Query a Broadcast List's Recipients & Name](#query-a-broadcast-lists-recipients--name)
- [Writing Custom Functionality](#writing-custom-functionality)
  - [Enabling Debug Level in Baileys Logs](#enabling-debug-level-in-baileys-logs)
  - [How Whatsapp Communicate With Us](#how-whatsapp-communicate-with-us)
  - [Register a Callback for Websocket Events](#register-a-callback-for-websocket-events)

## Connecting Account

WhatsApp provides a multi-device API that allows Baileys to be authenticated as a second WhatsApp client by scanning a **QR code** or **Pairing Code** with WhatsApp on your phone.

> [!NOTE]
> **[Here](#example-to-start) is a simple example of event handling**

> [!TIP]
> **You can see all supported socket configs [here](https://baileys.whiskeysockets.io/types/SocketConfig.html) (Recommended)**

### Starting socket with **QR-CODE**

> [!TIP]
> You can customize browser name if you connect with **QR-CODE**, with `Browser` constant, we have some browsers config, **see [here](https://baileys.whiskeysockets.io/types/BrowsersMap.html)**

```ts
import makeWASocket from 'baron-baileys-v2'

const sock = makeWASocket({
	// can provide additional config here
	browser: Browsers.ubuntu('My App'),
	printQRInTerminal: true
})
```

If the connection is successful, you will see a QR code printed on your terminal screen, scan it with WhatsApp on your phone and you'll be logged in!

### Starting socket with **Pairing Code**

> [!IMPORTANT]
> Pairing Code isn't Mobile API, it's a method to connect Whatsapp Web without QR-CODE, you can connect only with one device, see [here](https://faq.whatsapp.com/1324084875126592/?cms_platform=web)

The phone number can't have `+` or `()` or `-`, only numbers, you must provide country code

```ts
import makeWASocket from 'baron-baileys-v2'

const sock = makeWASocket({
	// can provide additional config here
	printQRInTerminal: false //need to be false
})

if (!sock.authState.creds.registered) {
	const number = 'XXXXXXXXXXX'
	const code = await sock.requestPairingCode(number) // or await sock.requestPairingCode(number, 'CODEOTPS') custom your pairing code
	console.log(code)
}
```

### Receive Full History

1. Set `syncFullHistory` as `true` if you want to force full history sync.
2. Baileys, by default, use chrome browser config
   - If you'd like to emulate a desktop connection (and receive more message history), this browser setting to your Socket config:

```ts
const sock = makeWASocket({
	...otherOpts,
	// can use Windows, Ubuntu here too
	browser: Browsers.macOS('Desktop'),
	syncFullHistory: true
})
```

Behavior note:

- With fresh auth state, full history sync can run once on first successful connect.
- That completion is persisted in auth creds, so later reconnects/restarts with the same auth folder do not repeat full sync automatically.
- You can still explicitly force it again by setting `syncFullHistory: true`.

## Important Notes About Socket Config

### Caching Group Metadata (Recommended)

- If you use baileys for groups, we recommend you to set `cachedGroupMetadata` in socket config, you need to implement a cache like this:

  ```ts
  const groupCache = new NodeCache({ stdTTL: 5 * 60, useClones: false })

  const sock = makeWASocket({
  	cachedGroupMetadata: async jid => groupCache.get(jid)
  })

  sock.ev.on('groups.update', async ([event]) => {
  	const metadata = await sock.groupMetadata(event.id)
  	groupCache.set(event.id, metadata)
  })

  sock.ev.on('group-participants.update', async event => {
  	const metadata = await sock.groupMetadata(event.id)
  	groupCache.set(event.id, metadata)
  })
  ```

### Improve Retry System & Decrypt Poll Votes

- If you want to improve sending message, retrying when error occurs and decrypt poll votes, you need to have a store and set `getMessage` config in socket like this:
  ```ts
  const sock = makeWASocket({
  	getMessage: async key => await getMessageFromStore(key)
  })
  ```

### Receive Notifications in Whatsapp App

- If you want to receive notifications in whatsapp app, set `markOnlineOnConnect` to `false`
  ```ts
  const sock = makeWASocket({
  	markOnlineOnConnect: false
  })
  ```

## Saving & Restoring Sessions

You obviously don't want to keep scanning the QR code every time you want to connect.

So, you can load the credentials to log back in:

```ts
import makeWASocket, { useMultiFileAuthState } from 'baron-baileys-v2'

const { state, saveCreds } = await useMultiFileAuthState('auth_info_baileys')

// will use the given state to connect
// so if valid credentials are available -- it'll connect without QR
const sock = makeWASocket({ auth: state })

// this will be called as soon as the credentials are updated
sock.ev.on('creds.update', saveCreds)
```

> [!IMPORTANT]
> `useMultiFileAuthState` is a utility function to help save the auth state in a single folder, this function serves as a good guide to help write auth & key states for SQL/no-SQL databases, which I would recommend in any production grade system.

> [!NOTE]
> When a message is received/sent, due to signal sessions needing updating, the auth keys (`authState.keys`) will update. Whenever that happens, you must save the updated keys (`authState.keys.set()` is called). Not doing so will prevent your messages from reaching the recipient & cause other unexpected consequences. The `useMultiFileAuthState` function automatically takes care of that, but for any other serious implementation -- you will need to be very careful with the key state management.

## Handling Events

- Baileys uses the EventEmitter syntax for events.
  They're all nicely typed up, so you shouldn't have any issues with an Intellisense editor like VS Code.

> [!IMPORTANT]
> **The events are [these](https://baileys.whiskeysockets.io/types/BaileysEventMap.html)**, it's important you see all events

You can listen to these events like this:

```ts
const sock = makeWASocket()
sock.ev.on('messages.upsert', ({ messages }) => {
	console.log('got messages', messages)
})
```

Built-in note: manual `@lid` mention rewrite handlers are no longer required for most bots.
Baileys now normalizes identity fields and mention arrays across receive/send/event paths where LID -> PN mapping data is available.
For troubleshooting, set `LID_PN_DEBUG=1`.

### Core Events You Should Handle (Recommended)

Use this list as your production baseline.

#### `connection.update`

- **When:** connection state changes (`connecting`, `open`, `close`) and QR updates.
- **Why:** reconnect strategy and connection diagnostics.
- **Handler:**

```ts
sock.ev.on('connection.update', update => {
	const { connection, lastDisconnect } = update
	console.log(connection, lastDisconnect?.error)
})
```

- **Pitfall:** reconnect blindly without checking logout reason.

#### `creds.update`

- **When:** auth credentials/keys rotate.
- **Why:** persistence of auth state.
- **Handler:**

```ts
sock.ev.on('creds.update', saveCreds)
```

- **Pitfall:** not saving key updates causes delivery/decryption issues.

#### `messages.upsert`

- **When:** new messages arrive.
- **Why:** primary inbound message processing.
- **Handler:**

```ts
sock.ev.on('messages.upsert', event => {
	for (const message of event.messages) {
		// process every message
	}
})
```

- **Pitfall:** handling only `event.messages[0]`.

#### `messages.update`

- **When:** message status/content update arrives (poll votes, receipts, edits metadata).
- **Why:** keep local state in sync and decrypt poll updates.
- **Handler:**

```ts
sock.ev.on('messages.update', updates => {
	for (const update of updates) {
		// update local message state
	}
})
```

- **Pitfall:** skipping updates makes message state inconsistent.

#### `messages.delete`

- **When:** message deletion events are emitted.
- **Why:** remove/hide messages in local store/UI.
- **Handler:**

```ts
sock.ev.on('messages.delete', item => {
	// delete by key from storage
})
```

- **Pitfall:** leaving deleted messages in local caches.

#### `message-receipt.update`

- **When:** read/delivery receipts update.
- **Why:** accurate read/delivered indicators.
- **Handler:**

```ts
sock.ev.on('message-receipt.update', receipts => {
	// apply receipt state
})
```

- **Pitfall:** assuming only one receipt per event.

#### `messaging-history.set`

- **When:** history chunk is pushed (initial sync/replay).
- **Why:** initial backfill and reconnect recovery.
- **Handler:**

```ts
sock.ev.on('messaging-history.set', data => {
	// merge chats/contacts/messages idempotently
})
```

- **Pitfall:** non-idempotent writes causing duplicates after reconnect.

#### `chats.upsert`

- **When:** new chat records are added.
- **Why:** create chats in local store.
- **Handler:**

```ts
sock.ev.on('chats.upsert', chats => {
	// insert new chats
})
```

- **Pitfall:** assuming all chats already exist before messages.

#### `chats.update`

- **When:** chat metadata changes (mute/archive/unread/etc.).
- **Why:** keep chat state accurate.
- **Handler:**

```ts
sock.ev.on('chats.update', updates => {
	// patch chat records
})
```

- **Pitfall:** overwriting full records instead of patching.

#### `chats.delete`

- **When:** chats are removed.
- **Why:** remove chat from local state.
- **Handler:**

```ts
sock.ev.on('chats.delete', deletions => {
	// remove chat ids
})
```

- **Pitfall:** stale chat references in dependent tables.

#### `contacts.upsert`

- **When:** new contacts are synced.
- **Why:** create contact directory entries.
- **Handler:**

```ts
sock.ev.on('contacts.upsert', contacts => {
	// insert contacts
})
```

- **Pitfall:** discarding partial payload fields you need later.

#### `contacts.update`

- **When:** contact fields change (name/status/device flags).
- **Why:** keep identity/profile metadata fresh.
- **Handler:**

```ts
sock.ev.on('contacts.update', updates => {
	// patch contacts
})
```

- **Pitfall:** treating update payload as full contact object.

#### `groups.update`

- **When:** group properties change (subject/announce/restrict/etc.).
- **Why:** maintain current group metadata.
- **Handler:**

```ts
sock.ev.on('groups.update', updates => {
	// patch group metadata cache/store
})
```

- **Pitfall:** not invalidating stale group cache.

#### `group-participants.update`

- **When:** members join/leave/promote/demote.
- **Why:** maintain participant roster and roles.
- **Handler:**

```ts
sock.ev.on('group-participants.update', event => {
	// apply participant delta by action
})
```

- **Pitfall:** ignoring action type and applying wrong mutation.

#### `call`

- **When:** call offer/accept/reject/timeout events are emitted.
- **Why:** missed-call handling and call-related UX/workflows.
- **Handler:**

```ts
sock.ev.on('call', calls => {
	for (const call of calls) {
		// handle call status transitions
	}
})
```

- **Pitfall:** assuming only direct calls; group calls also emit call events.

### Production Event Handling Checklist

- Always persist auth changes from `creds.update`.
- Make handlers idempotent to survive reconnect/history replay.
- Process all messages in `event.messages`, not only index `0`.
- Configure `getMessage` if you rely on retry/poll decryption paths.
- Cache group metadata for group-heavy bots.

### Debugging Tips

- Use structured logs around event entry/exit and reconnect causes.
- Enable `LID_PN_DEBUG=1` when tracking LID -> PN normalization behavior.
- Log disconnect reason codes from `connection.update` for recovery policy tuning.

### Example to Start

> [!NOTE]
> This example includes basic auth storage too

```ts
import makeWASocket, { DisconnectReason, useMultiFileAuthState } from 'baron-baileys-v2'
import { Boom } from '@hapi/boom'

async function connectToWhatsApp() {
	const { state, saveCreds } = await useMultiFileAuthState('./auth_info_baileys')
	const sock = makeWASocket({
		// can provide additional config here
		auth: state,
		printQRInTerminal: true
	})
	sock.ev.on('connection.update', update => {
		const { connection, lastDisconnect } = update
		if (connection === 'close') {
			const shouldReconnect = (lastDisconnect.error as Boom)?.output?.statusCode !== DisconnectReason.loggedOut
			console.log('connection closed due to ', lastDisconnect.error, ', reconnecting ', shouldReconnect)
			// reconnect if not logged out
			if (shouldReconnect) {
				connectToWhatsApp()
			}
		} else if (connection === 'open') {
			console.log('opened connection')
		}
	})
	sock.ev.on('messages.upsert', event => {
		for (const m of event.messages) {
			console.log(JSON.stringify(m, undefined, 2))

			console.log('replying to', m.key.remoteJid)
			await sock.sendMessage(m.key.remoteJid!, { text: 'Hello Word' })
		}
	})

	// to storage creds (session info) when it updates
	sock.ev.on('creds.update', saveCreds)
}
// run in main file
connectToWhatsApp()
```


> [!IMPORTANT]
> In `messages.upsert` it's recommended to use a loop like `for (const message of event.messages)` to handle all messages in array

### Decrypt Poll Votes

- By default poll votes are encrypted and handled in `messages.update`

```ts
import pino from 'pino'
import { makeInMemoryStore, getAggregateVotesInPollMessage } from 'baron-baileys-v2'

const logger = pino({ timestamp: () => `,"time":"${new Date().toJSON()}"` }).child({ class: '@Baron' })
logger.level = 'fatal'
const store = makeInMemoryStore({ logger })

async function getMessage(key) {
	if (store) {
		const msg = await store.loadMessage(key.remoteJid, key.id)
		return msg?.message
	}
	return {
		conversation: 'Baron'
	}
}

sock.ev.on('messages.update', async chatUpdate => {
	for (const { key, update } of chatUpdate) {
		if (update.pollUpdates && key.fromMe) {
			const pollCreation = await getMessage(key)
			if (pollCreation) {
				const pollUpdate = await getAggregateVotesInPollMessage({
					message: pollCreation,
					pollUpdates: update.pollUpdates
				})
				const toCmd = pollUpdate.filter(v => v.voters.length !== 0)[0]?.name
				if (toCmd == undefined) return
				console.log(toCmd)
			}
		}
	}
})
```

### Summary of Events on First Connection

1. When you connect first time, `connection.update` will be fired requesting you to restart sock
2. Then, history messages will be received in `messaging.history-set`

## Implementing a Data Store

- Baileys does not come with a defacto storage for chats, contacts, or messages. However, a simple in-memory implementation has been provided. The store listens for chat updates, new messages, message updates, etc., to always have an up-to-date version of the data.

> [!IMPORTANT]
> I highly recommend building your own data store, as storing someone's entire chat history in memory is a terrible waste of RAM.

It can be used as follows:

```ts
import makeWASocket, { makeInMemoryStore } from 'baron-baileys-v2'
// the store maintains the data of the WA connection in memory
// can be written out to a file & read from it
const store = makeInMemoryStore({})
// can be read from a file
store.readFromFile('./baileys_store.json')
// saves the state to a file every 10s
setInterval(() => {
	store.writeToFile('./baileys_store.json')
}, 10_000)

const sock = makeWASocket({})
// will listen from this socket
// the store can listen from a new socket once the current socket outlives its lifetime
store.bind(sock.ev)

sock.ev.on('chats.upsert', () => {
	// can use 'store.chats' however you want, even after the socket dies out
	// 'chats' => a KeyedDB instance
	console.log('got chats', store.chats.all())
})

sock.ev.on('contacts.upsert', () => {
	console.log('got contacts', Object.values(store.contacts))
})
```

The store also provides some simple functions such as `loadMessages` that utilize the store to speed up data retrieval.

## Whatsapp IDs Explain

- `id` is the WhatsApp ID, called `jid` too, of the person or group you're sending the message to.
  - It must be in the format `[country code][phone number]@s.whatsapp.net`
    - Example for people: `+19999999999@s.whatsapp.net`.
    - For groups, it must be in the format `123456789-123345@g.us`.
  - For broadcast lists, it's `[timestamp of creation]@broadcast`.
  - For stories, the ID is `status@broadcast`.

## Utility Functions

- `getContentType`, returns the content type for any message
- `getDevice`, returns the device from message
- `makeCacheableSignalKeyStore`, make auth store more fast
- `downloadContentFromMessage`, download content from any message

## Sending Messages

- Send all types of messages with a single function
  - **[Here](https://baileys.whiskeysockets.io/types/AnyMessageContent.html) you can see all message contents supported, like text message**
  - **[Here](https://baileys.whiskeysockets.io/types/MiscMessageGenerationOptions.html) you can see all options supported, like quote message**

  ```ts
  const jid: string
  const content: AnyMessageContent
  const options: MiscMessageGenerationOptions

  sock.sendMessage(jid, content, options)
  ```

### Non-Media Messages

#### Text Message

```ts
await sock.sendMessage(jid, { text: 'hello word' })
```

#### Quote Message (works with all types)

```ts
await sock.sendMessage(jid, { text: 'hello word' }, { quoted: message })
```

#### Mention User (works with most types)

- @number is to mention in text, it's optional

```ts
await sock.sendMessage(jid, {
	text: '@12345678901',
	mentions: ['12345678901@s.whatsapp.net']
})
```

#### Forward Messages

- You need to have message object, can be retrieved from [store](#implementing-a-data-store) or use a [message](https://baileys.whiskeysockets.io/types/WAMessage.html) object

```ts
const msg = getMessageFromStore() // implement this on your end
await sock.sendMessage(jid, { forward: msg, force: true or number }) // WA forward the message!
```

#### Location Message

```ts
await sock.sendMessage(jid, {
	location: {
		degreesLatitude: 24.121231,
		degreesLongitude: 55.1121221
	}
})
```

#### Live Location Message

```ts
await sock.sendMessage(jid, {
	location: {
		degreesLatitude: 24.121231,
		degreesLongitude: 55.1121221
	},
	live: true
})
```

#### Contact Message

```ts
const vcard =
	'BEGIN:VCARD\n' + // metadata of the contact card
	'VERSION:3.0\n' +
	'FN:Jeff Singh\n' + // full name
	'ORG:Ashoka Uni\n' + // the organization of the contact
	'TELtype=CELLtype=VOICEwaid=911234567890:+91 12345 67890\n' + // WhatsApp ID + phone number
	'END:VCARD'

await sock.sendMessage(id, {
	contacts: {
		displayName: 'Baron',
		contacts: [{ vcard }]
	}
})
```

#### Reaction Message

- You need to pass the key of message, you can retrieve from [store](#implementing-a-data-store) or use a [key](https://baileys.whiskeysockets.io/types/WAMessageKey.html) object

```ts
await sock.sendMessage(jid, {
	react: {
		text: '💖', // use an empty string to remove the reaction
		key: message.key
	}
})
```

#### Pin Message

- You need to pass the key of message, you can retrieve from [store](#implementing-a-data-store) or use a [key](https://baileys.whiskeysockets.io/types/WAMessageKey.html) object

- Time can be:

| Time | Seconds   |
| ---- | --------- |
| 24h  | 86.400    |
| 7d   | 604.800   |
| 30d  | 2.592.000 |

```ts
await sock.sendMessage(jid, {
	pin: {
		type: 1, // 2 to remove
		time: 86400,
		key: Key
	}
})
```

### Keep Message

```ts
await sock.sendMessage(jid, {
	keep: {
		key: Key,
		type: 1 // or 2
	}
})
```

#### Poll Message

```ts
await sock.sendMessage(
    jid,
    {
        poll: {
            name: 'My Poll',
            values: ['Option 1', 'Option 2', ...],
            selectableCount: 1,
            toAnnouncementGroup: false // or true
        }
    }
)
```

#### Poll Result Message

```ts
await sock.sendMessage(jid, {
	pollResult: {
		name: 'Hi',
		values: [
			['Option 1', 1000],
			['Option 2', 2000]
		]
	}
})
```

### Call Message

```ts
await sock.sendMessage(jid, {
	call: {
		name: 'Hay',
		type: 1 // 2 for video
	}
})
```

### Event Message

```ts
await sock.sendMessage(jid, {
	event: {
		isCanceled: false, // or true
		name: 'holiday together!',
		description: 'who wants to come along?',
		location: {
			degreesLatitude: 24.121231,
			degreesLongitude: 55.1121221,
			name: 'name'
		},
		startTime: number,
		endTime: number,
		extraGuestsAllowed: true // or false
	}
})
```

### Order Message

```ts
await sock.sendMessage(
    jid,
    {
        order: {
            orderId: '574xxx',
            thumbnail: 'your_thumbnail',
            itemCount: 'your_count',
            status: 'your_status', // INQUIRY || ACCEPTED || DECLINED
            surface: 'CATALOG',
            message: 'your_caption',
            orderTitle: "your_title",
            sellerJid: 'your_jid'',
            token: 'your_token',
            totalAmount1000: 'your_amount',
            totalCurrencyCode: 'IDR'
        }
    }
)
```

### Product Message

```ts
await sock.sendMessage(jid, {
	product: {
		productImage: {
			// for using buffer >> productImage: your_buffer
			url: your_url
		},
		productId: 'your_id',
		title: 'your_title',
		description: 'your_description',
		currencyCode: 'IDR',
		priceAmount1000: 'your_amount',
		retailerId: 'your_reid', // optional use if needed
		url: 'your_url', // optional use if needed
		productImageCount: 'your_imageCount',
		firstImageId: 'your_image', // optional use if needed
		salePriceAmount1000: 'your_priceSale',
		signedUrl: 'your_url' // optional use if needed
	},
	businessOwnerJid: 'your_jid'
})
```

### Payment Message

```ts
await sock.sendMessage(jid, {
	payment: {
		note: 'Hi!',
		currency: 'IDR', // optional
		offset: 0, // optional
		amount: '10000', // optional
		expiry: 0, // optional
		from: '628xxxx@s.whatsapp.net', // optional
		image: {
			// optional
			placeholderArgb: 'your_background', // optional
			textArgb: 'your_text', // optional
			subtextArgb: 'your_subtext' // optional
		}
	}
})
```

#### Payment Invite Message

```ts
await sock.sendMessage(id, {
	paymentInvite: {
		type: number, // 1 || 2 || 3
		expiry: 0
	}
})
```

### Admin Invite Message

```ts
await sock.sendMessage(jid, {
	adminInvite: {
		jid: '123xxx@newsletter',
		name: 'newsletter_name',
		caption: 'Please be my channel admin',
		expiration: 86400,
		jpegThumbnail: Buffer // optional
	}
})
```

### Group Invite Message

```ts
await sock.sendMessage(jid, {
	groupInvite: {
		jid: '123xxx@g.us',
		name: 'group_name',
		caption: 'Please Join My Whatsapp Group',
		code: 'code_invite',
		expiration: 86400,
		jpegThumbnail: Buffer // optional
	}
})
```

### Share Phone Number Message

```ts
await sock.sendMessage(jid, {
	sharePhoneNumber: {}
})
```

### Request Phone Number Message

```ts
await sock.sendMessage(jid, {
	requestPhoneNumber: {}
})
```

### Buttons Reply Message

```ts
// List
await sock.sendMessage(
    jid,
    {
        buttonReply: {
            name: 'Hii',
            description: 'description',
            rowId: 'ID'
       },
       type: 'list'
    }
)
// Plain
await sock.sendMessage(
    jid,
    {
        buttonReply: {
            displayText: 'Hii',
            id: 'ID'
       },
       type: 'plain'
    }
)

// Template
await sock.sendMessage(
    jid,
    {
        buttonReply: {
            displayText: 'Hii',
            id: 'ID',
            index: 'number'
       },
       type: 'template'
    }
)

// Interactive
await sock.sendMessage(
    jid,
    {
        buttonReply: {
            body: 'Hii',
            nativeFlows: {
                name: 'menu_options',
                paramsJson: JSON.stringify({ id: 'ID', description: 'description' })
                version: 1 // 2 | 3
            }
       },
       type: 'interactive'
    }
)
```

### Buttons Message

```ts
await sock.sendMessage(jid, {
	text: 'This is a button message!', // image: buffer or // image: { url: url } If you want to use images
	caption: 'caption', // Use this if you are using an image or video
	footer: 'Hello World!',
	buttons: [
		{
			buttonId: 'Id1',
			buttonText: {
				displayText: 'Button 1'
			}
		},
		{
			buttonId: 'Id2',
			buttonText: {
				displayText: 'Button 2'
			}
		},
		{
			buttonId: 'Id3',
			buttonText: {
				displayText: 'Button 3'
			}
		}
	]
})
```

### Buttons List Message

```ts
// Just working in a private chat
await sock.sendMessage(jid, {
	text: 'This is a list!',
	footer: 'Hello World!',
	title: 'Amazing boldfaced list title',
	buttonText: 'Required, text on the button to view the list',
	sections: [
		{
			title: 'Section 1',
			rows: [
				{
					title: 'Option 1',
					rowId: 'option1'
				},
				{
					title: 'Option 2',
					rowId: 'option2',
					description: 'This is a description'
				}
			]
		},
		{
			title: 'Section 2',
			rows: [
				{
					title: 'Option 3',
					rowId: 'option3'
				},
				{
					title: 'Option 4',
					rowId: 'option4',
					description: 'This is a description V2'
				}
			]
		}
	]
})
```

### Buttons Product List Message

```ts
// Just working in a private chat
await sock.sendMessage(jid, {
	text: 'This is a list!',
	footer: 'Hello World!',
	title: 'Amazing boldfaced list title',
	buttonText: 'Required, text on the button to view the list',
	productList: [
		{
			title: 'This is a title',
			products: [
				{
					productId: '1234'
				},
				{
					productId: '5678'
				}
			]
		}
	],
	businessOwnerJid: '628xxx@s.whatsapp.net',
	thumbnail: 'https://example.jpg' // or buffer
})
```

### Buttons Cards Message

```ts
await sock.sendMessage(jid, {
	text: 'Body Message',
	title: 'Title Message',
	subtile: 'Subtitle Message',
	footer: 'Footer Message',
	cards: [
		{
			image: { url: 'https://example.jpg' }, // or buffer
			title: 'Title Cards',
			body: 'Body Cards',
			footer: 'Footer Cards',
			buttons: [
				{
					name: 'quick_reply',
					buttonParamsJson: JSON.stringify({
						display_text: 'Display Button',
						id: 'ID'
					})
				},
				{
					name: 'cta_url',
					buttonParamsJson: JSON.stringify({
						display_text: 'Display Button',
						url: 'https://www.example.com'
					})
				}
			]
		},
		{
			video: { url: 'https://example.mp4' }, // or buffer
			title: 'Title Cards',
			body: 'Body Cards',
			footer: 'Footer Cards',
			buttons: [
				{
					name: 'quick_reply',
					buttonParamsJson: JSON.stringify({
						display_text: 'Display Button',
						id: 'ID'
					})
				},
				{
					name: 'cta_url',
					buttonParamsJson: JSON.stringify({
						display_text: 'Display Button',
						url: 'https://www.example.com'
					})
				}
			]
		}
	]
})
```

### Buttons Template Message

```ts
await sock.sendMessage(jid, {
	text: 'This is a template message!',
	footer: 'Hello World!',
	templateButtons: [
		{
			index: 1,
			urlButton: {
				displayText: 'Follow Me',
				url: 'https://whatsapp.com/channel/0029Vag9VSI2ZjCocqa2lB1y'
			}
		},
		{
			index: 2,
			callButton: {
				displayText: 'Call Me!',
				phoneNumber: '628xxx'
			}
		},
		{
			index: 3,
			quickReplyButton: {
				displayText: 'This is a reply, just like normal buttons!',
				id: 'id-like-buttons-message'
			}
		}
	]
})
```

### Buttons Interactive Message

```ts
await sock.sendMessage(jid, {
	text: 'This is an Interactive message!',
	title: 'Hiii',
	subtitle: 'There is a subtitle',
	footer: 'Hello World!',
	interactiveButtons: [
		{
			name: 'quick_reply',
			buttonParamsJson: JSON.stringify({
				display_text: 'Click Me!',
				id: 'your_id'
			})
		},
		{
			name: 'cta_url',
			buttonParamsJson: JSON.stringify({
				display_text: 'Follow Me',
				url: 'https://whatsapp.com/channel/0029Vag9VSI2ZjCocqa2lB1y',
				merchant_url: 'https://whatsapp.com/channel/0029Vag9VSI2ZjCocqa2lB1y'
			})
		},
		{
			name: 'cta_copy',
			buttonParamsJson: JSON.stringify({
				display_text: 'Click Me!',
				copy_code: 'https://whatsapp.com/channel/0029Vag9VSI2ZjCocqa2lB1y'
			})
		},
		{
			name: 'cta_call',
			buttonParamsJson: JSON.stringify({
				display_text: 'Call Me!',
				phone_number: '628xxx'
			})
		},
		{
			name: 'cta_catalog',
			buttonParamsJson: JSON.stringify({
				business_phone_number: '628xxx'
			})
		},
		{
			name: 'cta_reminder',
			buttonParamsJson: JSON.stringify({
				display_text: '...'
			})
		},
		{
			name: 'cta_cancel_reminder',
			buttonParamsJson: JSON.stringify({
				display_text: '...'
			})
		},
		{
			name: 'address_message',
			buttonParamsJson: JSON.stringify({
				display_text: '...'
			})
		},
		{
			name: 'send_location',
			buttonParamsJson: JSON.stringify({
				display_text: '...'
			})
		},
		{
			name: 'open_webview',
			buttonParamsJson: JSON.stringify({
				title: 'Follow Me!',
				link: {
					in_app_webview: true, // or false
					url: 'https://whatsapp.com/channel/0029Vag9VSI2ZjCocqa2lB1y'
				}
			})
		},
		{
			name: 'mpm',
			buttonParamsJson: JSON.stringify({
				product_id: '8816262248471474'
			})
		},
		{
			name: 'wa_payment_transaction_details',
			buttonParamsJson: JSON.stringify({
				transaction_id: '12345848'
			})
		},
		{
			name: 'automated_greeting_message_view_catalog',
			buttonParamsJson: JSON.stringify({
				business_phone_number: '628xxx',
				catalog_product_id: '12345'
			})
		},
		{
			name: 'galaxy_message',
			buttonParamsJson: JSON.stringify({
				mode: 'published',
				flow_message_version: '3',
				flow_token: '1:1307913409923914:293680f87029f5a13d1ec5e35e718af3',
				flow_id: '1307913409923914',
				flow_cta: 'Baron >\\<',
				flow_action: 'navigate',
				flow_action_payload: {
					screen: 'QUESTION_ONE',
					params: {
						user_id: '123456789',
						referral: 'campaign_xyz'
					}
				},
				flow_metadata: {
					flow_json_version: '201',
					data_api_protocol: 'v2',
					flow_name: 'Lead Qualification [en]',
					data_api_version: 'v2',
					categories: ['Lead Generation', 'Sales']
				}
			})
		},
		{
			name: 'single_select',
			buttonParamsJson: JSON.stringify({
				title: 'Click Me!',
				sections: [
					{
						title: 'Title 1',
						highlight_label: 'Highlight label 1',
						rows: [
							{
								header: 'Header 1',
								title: 'Title 1',
								description: 'Description 1',
								id: 'Id 1'
							},
							{
								header: 'Header 2',
								title: 'Title 2',
								description: 'Description 2',
								id: 'Id 2'
							}
						]
					}
				]
			})
		}
	]
})

// If you want to use an image
await sock.sendMessage(jid, {
	image: {
		url: 'https://example.jpg'
	},
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	interactiveButtons: [
		{
			name: 'quick_reply',
			buttonParamsJson: JSON.stringify({
				display_text: 'DisplayText',
				id: 'ID1'
			})
		}
	],
	hasMediaAttachment: false // or true
})

// If you want to use an video
await sock.sendMessage(jid, {
	video: {
		url: 'https://example.mp4'
	},
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	interactiveButtons: [
		{
			name: 'quick_reply',
			buttonParamsJson: JSON.stringify({
				display_text: 'DisplayText',
				id: 'ID1'
			})
		}
	],
	hasMediaAttachment: false // or true
})

// If you want to use an document
await sock.sendMessage(jid, {
	document: {
		url: 'https://example.jpg'
	},
	mimetype: 'image/jpeg',
	jpegThumbnail: await sock.resize('https://example.jpg', 320, 320),
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	interactiveButtons: [
		{
			name: 'quick_reply',
			buttonParamsJson: JSON.stringify({
				display_text: 'DisplayText',
				id: 'ID1'
			})
		}
	],
	hasMediaAttachment: false // or true
})

// If you want to use an location
await sock.sendMessage(jid, {
	location: {
		degressLatitude: -0,
		degressLongitude: 0,
		name: 'Hi'
	},
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	interactiveButtons: [
		{
			name: 'quick_reply',
			buttonParamsJson: JSON.stringify({
				display_text: 'DisplayText',
				id: 'ID1'
			})
		}
	],
	hasMediaAttachment: false // or true
})

// if you want to use an product
await sock.sendMessage(jid, {
	product: {
		productImage: {
			url: 'https://example.jpg'
		},
		productId: '836xxx',
		title: 'Title',
		description: 'Description',
		currencyCode: 'IDR',
		priceAmount1000: '283xxx',
		retailerId: 'Baron',
		url: 'https://example.com',
		productImageCount: 1
	},
	businessOwnerJid: '628xxx@s.whatsapp.net',
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	interactiveButtons: [
		{
			name: 'quick_reply',
			buttonParamsJson: JSON.stringify({
				display_text: 'DisplayText',
				id: 'ID1'
			})
		}
	],
	hasMediaAttachment: false // or true
})
```

### Buttons Interactive Message PIX

```ts
await sock.sendMessage(jid, {
	text: '', // This string is required. Even it's empty.
	interactiveButtons: [
		{
			name: 'payment_info',
			buttonParamsJson: JSON.stringify({
				payment_settings: [
					{
						type: 'pix_static_code',
						pix_static_code: {
							merchant_name: 'Baron',
							key: 'example@gmail.com',
							key_type: 'EMAIL' // PHONE || EMAIL || CPF || EVP
						}
					}
				]
			})
		}
	]
})
```

### Buttons Interactive Message PAY

```ts
await sock.sendMessage(jid, {
	text: '', // This string is required. Even it's empty.
	interactiveButtons: [
		{
			name: 'review_and_pay',
			buttonParamsJson: JSON.stringify({
				currency: 'IDR',
				payment_configuration: '',
				payment_type: '',
				total_amount: {
					value: '999999999',
					offset: '100'
				},
				reference_id: '45XXXXX',
				type: 'physical-goods',
				payment_method: 'confirm',
				payment_status: 'captured',
				payment_timestamp: Math.floor(Date.now() / 1000),
				order: {
					status: 'completed',
					description: '',
					subtotal: {
						value: '0',
						offset: '100'
					},
					order_type: 'PAYMENT_REQUEST',
					items: [
						{
							retailer_id: 'your_retailer_id',
							name: 'Baron >\\\<',
							amount: {
								value: '999999999',
								offset: '100'
							},
							quantity: '1'
						}
					]
				},
				additional_note: 'Baron >\\\<',
				native_payment_methods: [],
				share_payment_status: false
			})
		}
	]
})
```

### Status Mentions Message

```ts
const jids = [
    '123451679@g.us',
    '124848899@g.us',
    '111384848@g.us',
    '62689xxxx@s.whatsapp.net',
    '62xxxxxxx@s.whatsapp.net'
]
// Text
await sock.sendStatusMentions(
    {
      text: 'Hello Everyone',
      font: 2, // optional
      textColor: 'FF0000', // optional
      backgroundColor: '#000000' // optional
    },
    jids // Limit to 5 mentions per status
)

// Image
await sock.sendStatusMentions(
    {
      Image: { url: 'https://example.com/ruriooe.jpg' }, or image buffer
      caption: 'Hello Everyone ' // optional
    },
    jids // Limit to 5 mentions per status
)

// Video
await sock.sendStatusMentions(
    {
      video: { url: 'https://example.com/ruriooe.mp4' }, or video buffer
      caption: 'Hello Everyone ' // optional
    },
    jids // Limit to 5 mentions per status
)

// Audio
await sock.sendStatusMentions(
    {
      audio: { url: 'https://example.com/ruriooe.mp3' }, or audio buffer
      backgroundColor: '#000000', // optional
      mimetype: 'audio/mp4',
      ppt: true
    },
    jids // Limit to 5 mentions per status
)
```

### Send Album Message

```ts
await sock.sendAlbumMessage(
	jid,
	[
		{
			image: { url: 'https://example.jpg' },
			caption: 'Hello World'
		},
		{
			image: Buffer,
			caption: 'Hello World'
		},
		{
			video: { url: 'https://example.mp4' },
			caption: 'Hello World'
		},
		{
			video: Buffer,
			caption: 'Hello World'
		}
	],
	{
		quoted: message,
		delay: 2000
	}
)
```

### Shop Message

```ts
await sock.sendMessage(jid, {
	text: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	shop: {
		surface: 1, // 2 | 3 | 4
		id: 'https://example.com'
	},
	viewOnce: true
})

// Image
await sock.sendMessage(jid, {
	image: {
		url: 'https://example.jpg'
	},
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	shop: {
		surface: 1, // 2 | 3 | 4
		id: 'https://example.com'
	},
	hasMediaAttachment: false, // or true
	viewOnce: true
})

// Video
await sock.sendMessage(jid, {
	video: {
		url: 'https://example.jpg'
	},
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	shop: {
		surface: 1, // 2 | 3 | 4
		id: 'https://example.com'
	},
	hasMediaAttachment: false, // or true
	viewOnce: true
})

// Document
await sock.sendMessage(jid, {
	document: {
		url: 'https://example.jpg'
	},
	mimetype: 'image/jpeg',
	jpegThumbnail: await sock.resize('https://example.jpg', 320, 320),
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	shop: {
		surface: 1, // 2 | 3 | 4
		id: 'https://example.com'
	},
	hasMediaAttachment: false, // or true,
	viewOnce: true
})

// Location
await sock.sendMessage(jid, {
	location: {
		degressLatitude: -0,
		degressLongitude: 0,
		name: 'Hi'
	},
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	shop: {
		surface: 1, // 2 | 3 | 4
		id: 'https://example.com'
	},
	hasMediaAttachment: false, // or true
	viewOnce: true
})

// Product
await sock.sendMessage(jid, {
	product: {
		productImage: {
			url: 'https://example.jpg'
		},
		productId: '836xxx',
		title: 'Title',
		description: 'Description',
		currencyCode: 'IDR',
		priceAmount1000: '283xxx',
		retailerId: 'Baron',
		url: 'https://example.com',
		productImageCount: 1
	},
	businessOwnerJid: '628xxx@s.whatsapp.net',
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	shop: {
		surface: 1, // 2 | 3 | 4
		id: 'https://example.com'
	},
	hasMediaAttachment: false, // or true
	viewOnce: true
})
```

### Collection Message

```ts
await sock.sendMessage(jid, {
	text: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	collection: {
		bizJid: 'jid',
		id: 'https://example.com',
		version: 1
	},
	viewOnce: true
})

// Image
await sock.sendMessage(jid, {
	image: {
		url: 'https://example.jpg'
	},
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	collection: {
		bizJid: 'jid',
		id: 'https://example.com',
		version: 1
	},
	hasMediaAttachment: false, // or true
	viewOnce: true
})

// Video
await sock.sendMessage(jid, {
	video: {
		url: 'https://example.jpg'
	},
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	collection: {
		bizJid: 'jid',
		id: 'https://example.com',
		version: 1
	},
	hasMediaAttachment: false, // or true
	viewOnce: true
})

// Document
await sock.sendMessage(jid, {
	document: {
		url: 'https://example.jpg'
	},
	mimetype: 'image/jpeg',
	jpegThumbnail: await sock.resize('https://example.jpg', 320, 320),
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	collection: {
		bizJid: 'jid',
		id: 'https://example.com',
		version: 1
	},
	hasMediaAttachment: false, // or true,
	viewOnce: true
})

// Location
await sock.sendMessage(jid, {
	location: {
		degressLatitude: -0,
		degressLongitude: 0,
		name: 'Hi'
	},
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	collection: {
		bizJid: 'jid',
		id: 'https://example.com',
		version: 1
	},
	hasMediaAttachment: false, // or true
	viewOnce: true
})

// Product
await sock.sendMessage(jid, {
	product: {
		productImage: {
			url: 'https://example.jpg'
		},
		productId: '836xxx',
		title: 'Title',
		description: 'Description',
		currencyCode: 'IDR',
		priceAmount1000: '283xxx',
		retailerId: 'Baron',
		url: 'https://example.com',
		productImageCount: 1
	},
	businessOwnerJid: '628xxx@s.whatsapp.net',
	caption: 'Body',
	title: 'Title',
	subtitle: 'Subtitle',
	footer: 'Footer',
	collection: {
		bizJid: 'jid',
		id: 'https://example.com',
		version: 1
	},
	hasMediaAttachment: false, // or true
	viewOnce: true
})
```

### Sticker Pack Message

```ts
// I don't know why the sticker doesn't appear
await sock.sendMessage(jid, {
	stickerPack: {
		name: 'Hiii',
		publisher: 'Baron',
		description: 'Hello',
		cover: Buffer, // Image buffer
		stickers: [
			{
				data: { url: 'https://example.com/1234kjd.webp' },
				emojis: ['❤'], // optional
				accessibilityLabel: '' // optional
			},
			{
				data: Buffer,
				emojis: ['❤'], // optional
				accessibilityLabel: '' // optional
			}
		]
	}
})
```

### Rich AI Response Message

Send a WhatsApp AI-style rich response — the same format used by Meta AI bots — with an optional syntax-highlighted code block.
Uses `botForwardedMessage` → `richResponseMessage` → `unifiedResponse` (raw JSON bytes in the `data` field).

Token types produced by the built-in tokenizer: `KEYWORD`, `STR`, `NUMBER`, `METHOD`, `COMMENT`, `DEFAULT`.

WAProto types used: `AIRichResponseMessage` (field 97), `AIRichResponseUnifiedResponse`, `ForwardedAIBotMessageInfo`, `BotMessageSharingInfo`.

```js
// Text-only
await sock.sendMessage(jid, {
  richResponse: {
    text: 'aku hann universe'
  }
})

// Text + JS code block (auto-tokenized)
await sock.sendMessage(jid, {
  richResponse: {
    text: 'Here is a Hello World example:',
    code: 'console.log("Hello World")',
    language: 'javascript'   // default
  }
})

// Text + Table
await sock.sendMessage(jid, {
  richResponse: {
    text: 'Leaderboard:',
    table: {
      rows: [
        ['Name', 'Score'],   // header row
        ['Alice', '100'],
        ['Bob', '90']
      ]
    }
  }
})

// LaTeX equation
await sock.sendMessage(jid, {
  richResponse: {
    text: 'Einstein\'s equation:',
    latex: 'E = mc^2'
    // or multiple: latex: ['E = mc^2', 'F = ma']
  }
})

// Map with marker
await sock.sendMessage(jid, {
  richResponse: {
    map: {
      latitude: 48.8566,
      longitude: 2.3522,
      zoom: 12,
      title: 'Paris',
      annotations: [{ latitude: 48.8584, longitude: 2.2945, title: 'Eiffel Tower' }]
    }
  }
})

// Single inline image
await sock.sendMessage(jid, {
  richResponse: {
    text: 'Check this out:',
    imageUrl: 'https://example.com/image.jpg'
  }
})

// Grid of images
await sock.sendMessage(jid, {
  richResponse: {
    text: 'Gallery:',
    imageUrls: [
      'https://example.com/1.jpg',
      'https://example.com/2.jpg',
      'https://example.com/3.jpg'
    ]
  }
})

// Text + code + custom bot JID
await sock.sendMessage(jid, {
  richResponse: {
    text: 'Result:',
    code: 'const x = 42\nconsole.log(x)',
    botJid: '259786046210223@bot'
  }
})
```

### Rich Composer Methods

These methods are available directly on the socket and send rich AI-style content using the `botForwardedMessage` proto chain.

#### Send Table

```js
await sock.sendTable(
  jid,
  'User Stats',                        // title
  ['Name', 'Score', 'Rank'],           // headers
  [['Alice', '980', '#1'], ['Bob', '870', '#2']], // rows
  quoted,                              // optional quoted message
  { headerText: 'Top Players', footer: 'Updated daily' } // optional
)
```

#### Send List

```js
await sock.sendList(
  jid,
  'Shopping List',         // title
  ['Apples', 'Bread', 'Milk'], // items (string[] or string[][])
  quoted,                  // optional
  { footer: 'Remember to buy!' }
)
```

#### Send Code Block

```js
await sock.sendCodeBlock(
  jid,
  'console.log("Hello World")', // code string
  quoted,                       // optional
  {
    title: 'Example',
    language: 'javascript',     // default: 'javascript'
    footer: 'Run with Node.js'
  }
)
```

#### Send Latex

Send a LaTeX expression as text (no image rendering).

```js
await sock.sendLatex(
  jid,
  quoted,
  {
    text: 'Quadratic formula:',
    expressions: [
      {
        latexExpression: 'x = \\frac{-b \\pm \\sqrt{b^2-4ac}}{2a}',
        url: 'https://example.com/formula.png', // pre-rendered image url
        width: 300,
        height: 80
      }
    ],
    footer: 'Source: Wikipedia'
  }
)
```

#### Send Latex Image

Renders LaTeX to a PNG via your own `renderLatexToPng` function and uploads it.

```js
const renderLatexToPng = async (expr) => {
  // return { buffer: Buffer, width: number, height: number }
}

await sock.sendLatexImage(
  jid,
  quoted,
  {
    text: 'Euler\'s identity:',
    expressions: [{ latexExpression: 'e^{i\\pi}+1=0' }]
  },
  renderLatexToPng,
  waUploadToServer
)
```

#### Send Latex Inline Image

Same as above but renders each expression as an inline image block.

```js
await sock.sendLatexInlineImage(
  jid,
  quoted,
  {
    text: 'Inline formula:',
    expressions: [{ latexExpression: '\\sqrt{2}' }]
  },
  renderLatexToPng,
  waUploadToServer
)
```

#### Send Rich Message

Send a fully custom array of submessages using the rich response format.

```js
await sock.sendRichMessage(
  jid,
  [
    { messageType: 2, messageText: 'Header text' },
    { messageType: 5, codeMetadata: { codeLanguage: 'javascript', codeBlocks: [...] } }
  ],
  quoted, // optional
  options: {
  	botJid: '123@bot' // optional
  	mentions: [...] // optional
  }
)
```

#### Capture & Resend Unified Response

Capture the `unifiedResponse` payload from a received Meta AI message and forward it.

```js
// capture from an incoming message
const captured = sock.captureUnifiedResponse(message)

// resend to another jid
if (captured) {
  await sock.sendUnifiedResponse(jid, quoted, captured)
}
```

### Status Notification Message

> [!NOTE]
> Added in WA 2.3000+

Sent when a status add-yours / reshare / question-answer-reshare event fires.

```js
await sock.sendMessage(jid, {
  statusNotification: {
    responseMessageKey: { remoteJid: jid, id: 'MSG_ID' },
    originalMessageKey:  { remoteJid: jid, id: 'ORIG_ID' },
    type: 1  // 1=STATUS_ADD_YOURS, 2=STATUS_RESHARE, 3=STATUS_QUESTION_ANSWER_RESHARE
  }
})
// full proto key also accepted:
// statusNotificationMessage: { ... }
```

### Status Question Answer Message

> [!NOTE]
> Added in WA 2.3000+

User answered a status question.

```js
await sock.sendMessage(jid, {
  statusQuestionAnswer: {
    key:  { remoteJid: jid, id: 'MSG_ID' },
    text: 'My answer'
  }
})
// full proto key: statusQuestionAnswerMessage
```

### Question Response Message

> [!NOTE]
> Added in WA 2.3000+

Direct response to a question message.

```js
await sock.sendMessage(jid, {
  questionResponse: {
    key:  { remoteJid: jid, id: 'QUESTION_MSG_ID' },
    text: 'My response'
  }
})
// full proto key: questionResponseMessage
```

### Status Quoted Message

> [!NOTE]
> Added in WA 2.3000+

Quote a status with a custom type.

```js
await sock.sendMessage(jid, {
  statusQuoted: {
    type: 1,           // 1 = QUESTION_ANSWER
    text: 'Quoted text',
    thumbnail: Buffer, // optional
    originalStatusId: { remoteJid: jid, id: 'STATUS_MSG_ID' }
  }
})
// full proto key: statusQuotedMessage
```

### Status Sticker Interaction Message

> [!NOTE]
> Added in WA 2.3000+

React to a status with a sticker.

```js
await sock.sendMessage(jid, {
  statusStickerInteraction: {
    key:       { remoteJid: jid, id: 'STATUS_MSG_ID' },
    stickerKey: 'sticker-hash-key',
    type: 1    // 1 = REACTION
  }
})
// full proto key: statusStickerInteractionMessage
```

### Newsletter Follower Invite Message

> [!NOTE]
> Added in WA 2.3000+

Invite a user to follow a newsletter.

```js
await sock.sendMessage(jid, {
  newsletterFollowerInvite: {
    newsletterJid:  '120363xxxxxx@newsletter',
    newsletterName: 'My Channel',
    jpegThumbnail:  Buffer, // optional
    caption: 'Join my channel!'
  }
})
// full proto key: newsletterFollowerInviteMessageV2
```

### Message History Notice

> [!NOTE]
> Added in WA 2.3000+

Notify about message history metadata.

```js
await sock.sendMessage(jid, {
  messageHistoryNotice: {
    contextInfo: { ... }
    // messageHistoryMetadata is optional
  }
})
```

### Scheduled Call Message

Schedule a future voice or video call.

```js
const { ScheduledCallType } = require('baron-baileys-v2')

// Schedule a voice call in 1 hour
await sock.sendMessage(jid, {
  scheduledCall: {
    title: 'Team Standup',
    isVideo: false,                    // or callType: ScheduledCallType.VOICE
    scheduledAt: new Date(Date.now() + 3600_000)
  }
})

// Schedule a video call
await sock.sendMessage(jid, {
  scheduledCall: {
    title: 'Video Meeting',
    isVideo: true,
    scheduledAt: new Date('2026-06-10T14:00:00Z')
  }
})

// Cancel a scheduled call (by key of the scheduled call message)
await sock.sendMessage(jid, {
  editScheduledCall: {
    key: { id: 'SCHEDULED_CALL_MSG_ID', remoteJid: jid }
  }
})
```

### Event Invite Message

Send an invitation to an existing event by its ID.

```js
await sock.sendMessage(jid, {
  eventInvite: {
    id: 'EVT_ID_FROM_EVENT_MESSAGE',
    title: 'Birthday Party',
    startDate: new Date('2026-07-01T18:00:00Z'),
    endDate:   new Date('2026-07-01T22:00:00Z'),
    text: 'You are invited!',
    thumbnail: Buffer,   // optional jpegThumbnail
  }
})
// full proto key: eventInviteMessage
```

### Comment Message

Post a threaded comment on any message.

```js
await sock.sendMessage(jid, {
  comment: {
    targetMessageKey: { id: 'ORIGINAL_MSG_ID', remoteJid: jid },
    message: { conversation: 'Great post!' }
  }
})
// full proto key: commentMessage
```

### Split Payment Message

Split a bill among multiple participants.

```js
const { SplitPaymentStatus } = require('baron-baileys-v2')

await sock.sendMessage(jid, {
  splitPayment: {
    totalAmount:  { value: 3000, currencyCode: 'INR' },
    description:  'Dinner at the restaurant',
    participants: [
      { jid: 'alice@s.whatsapp.net', amount: { value: 1500, currencyCode: 'INR' } },
      { jid: 'bob@s.whatsapp.net',   amount: { value: 1500, currencyCode: 'INR' } }
    ]
  }
})
// full proto key: splitPaymentMessage
// Status enum: SplitPaymentStatus.PENDING / PAID
```

### P2P Payment Reminder

Send a recurring payment reminder notification.

```js
const { P2PPaymentReminderFrequency, P2PPaymentReminderState } = require('baron-baileys-v2')

await sock.sendMessage(jid, {
  p2pPaymentReminder: {
    amount:      { value: 500, currencyCode: 'INR' },
    description: 'Monthly rent',
    frequency:   'monthly',        // or P2PPaymentReminderFrequency.MONTHLY
    state:       'active',         // or P2PPaymentReminderState.ACTIVE
    receiverJid: 'bob@s.whatsapp.net',
    upiId:       'bob@upi',        // optional
    expiryTimestamp: Date.now() + 30 * 24 * 3600 * 1000
  }
})
// full proto key: p2PPaymentReminderNotification
```

### Conditional Reveal (Scheduled) Message

Send a message that is revealed at a specific time (scheduled reveal).

```js
await sock.sendMessage(jid, {
  conditionalReveal: {
    revealKeyId: 'my-reveal-key-123'   // optional, auto-generated if omitted
    // messageSecret is auto-generated
  }
})
// full proto key: conditionalRevealMessage
// Type: ConditionalRevealType.SCHEDULED_MESSAGE
```

### Call Log Message

Log a call event to a chat (shows as a call history entry).

```js
const { CallLogOutcome, CallLogType } = require('baron-baileys-v2')

await sock.sendMessage(jid, {
  callLog: {
    isVideo:      false,
    outcome:      'missed',          // or CallLogOutcome.MISSED
    durationSecs: 0,
    callType:     'regular',         // or CallLogType.REGULAR
    participants: [
      { jid: 'alice@s.whatsapp.net', outcome: 'missed' }
    ]
  }
})
// full proto key: callLogMesssage (note: WA proto typo, triple-s)
// Outcome values: connected, missed, failed, rejected, accepted_elsewhere, ongoing, silenced_by_dnd, silenced_unknown_caller
// CallType values: regular, scheduled_call, voice_chat
```

### Status Mention Message

Wrap a message as a status mention (appears in the status mention flow).

```js
await sock.sendMessage(jid, {
  statusMention: {
    message: { conversation: 'Check this out!' }
  }
})
// full proto key: statusMentionMessage (FutureProofMessage wrapper)
```

### FutureProof Wrappers (Question, Spoiler, etc.)

All these types use `FutureProofMessage { message: Message }` internally.

```js
// Question / interactive prompt
await sock.sendMessage(jid, {
  question: { conversation: 'What is the capital of France?' }
})

// Reply to a question
await sock.sendMessage(jid, {
  questionReply: { conversation: 'Paris!' }
})

// Spoiler-tagged content
await sock.sendMessage(jid, {
  spoiler: {
    message: { conversation: 'Darth Vader is Luke\'s father' }
  }
})

// "Add yours" status prompt
await sock.sendMessage(jid, {
  statusAddYours: {
    message: { conversation: 'Show your best photo!' }
  }
})

// Lottie sticker
await sock.sendMessage(jid, {
  lottieSticker: {
    message: { stickerMessage: { url: 'https://...', mimetype: 'image/webp' } }
  }
})

// Bot task wrapper
await sock.sendMessage(jid, {
  botTask: {
    message: { conversation: 'Translate this to English' }
  }
})
```

### AI Image / Video Source Flags

Mark images as AI-generated or AI-modified, and embed QR codes.

```js
const { ImageSourceType, VideoSourceType } = require('baron-baileys-v2')

// AI-generated image
await sock.sendMessage(jid, {
  image: { url: './ai-art.jpg' },
  caption: 'AI artwork',
  aiGenerated: true              // sets imageSourceType = AI_GENERATED
})

// AI-modified image
await sock.sendMessage(jid, {
  image: { url: './edited.jpg' },
  aiModified: true               // sets imageSourceType = AI_MODIFIED
})

// Custom imageSourceType
await sock.sendMessage(jid, {
  image: { url: './img.jpg' },
  imageSourceType: 'rasterized'  // or ImageSourceType.RASTERIZED_TEXT_STATUS
})

// Image with QR code URL embedded
await sock.sendMessage(jid, {
  image: { url: './qr-card.jpg' },
  qrUrl: 'https://wa.me/p/123456789/ABC'   // QR code destination
})
```

### Extended Text Enhancements (Music, Video, End-Cards)

Add rich metadata to text/link-preview messages.

```js
// Music attribution (EmbeddedMusic)
await sock.sendMessage(jid, {
  text: 'Now listening 🎵',
  musicMetadata: {
    title: 'Bohemian Rhapsody',
    author: 'Queen',
    songId: 'spotify:track:abc123',
    artistAttribution: '© 1975 Queen',
    isExplicit: false,
    startTimeMs: 30000        // start at 30s
  }
})

// Direct video URL for link preview
await sock.sendMessage(jid, {
  text: 'https://youtube.com/watch?v=abc',
  videoContentUrl: 'https://cdn.example.com/video.mp4'
})

// Video end-card tiles (profile cards shown after a video)
await sock.sendMessage(jid, {
  text: 'Watch this!',
  endCardTiles: [
    { username: '@alice', caption: 'Subscribe!', thumbnailUrl: 'https://img/thumb.jpg', profilePictureUrl: 'https://img/avatar.jpg' },
    { username: '@bob',   caption: 'More videos', thumbnailUrl: 'https://img/thumb2.jpg' }
  ]
})
```

### Business Interaction Pills

Add clickable CTA pills to messages from/about a business.

```js
const { BizInteractionPillType } = require('baron-baileys-v2')

await sock.sendMessage(jid, {
  text: 'Welcome! How can we help?',
  businessInteractionPills: {
    businessJid: 'business@s.whatsapp.net',
    entryPoint: 'p2p_link_share',
    pills: [
      { type: 'catalog',          url: 'https://wa.me/c/123456' },
      { type: 'call',             url: 'tel:+1234567890' },
      { type: 'book_appointment', url: 'https://calendly.com/biz' },
      { type: 'menu',             url: 'https://biz.com/menu' }
    ]
    // Pill types: view_business, chat, call, catalog, channel,
    //             book_appointment, offers, bestsellers, menu, about
  }
})
```

### Bot Metadata — Session, Reminder, Plugin

Populate `BotMetadata` fields when sending to or from bots.

```js
const { BotSessionSource, BotReminderAction, BotPluginSearchProvider } = require('baron-baileys-v2')

// Attach session metadata (how the user started this bot session)
await sock.sendMessage(botJid, {
  text: 'Hello!',
  botSession: {
    sessionId: 'my-session-id',
    source: 'user_input'           // or BotSessionSource.USER_INPUT
  }
})

// Create a bot reminder
await sock.sendMessage(botJid, {
  text: 'Remind me tomorrow',
  botReminder: {
    action: 'create',              // or BotReminderAction.CREATE
    name: 'Team standup',
    frequency: 'daily',
    timestamp: Date.now() + 86400000
  }
})

// Attach plugin metadata (search/reels bot)
await sock.sendMessage(botJid, {
  text: 'Search this',
  botPlugin: {
    provider: 'google',            // or BotPluginSearchProvider.GOOGLE
    pluginType: 'search',
    searchQuery: 'WhatsApp protocol',
    expectedLinksCount: 5
  }
})
```

### Data Sharing Context (Business Messages)

Attach Meta media disclosure flags to business messages.

```js
await sock.sendMessage(jid, {
  text: 'Business message',
  dataSharingContext: {
    showMmDisclosure: true,        // show Meta media disclosure
    flags: 1                       // SHOW_MM_DISCLOSURE_ON_CLICK
  }
})
```

### Carousel Message

Send an interactive carousel with multiple scrollable cards.

```js
const { CarouselCardType } = require('baron-baileys-v2')

await sock.sendMessage(jid, {
  carousel: {
    cardType: 'horizontal',        // or CarouselCardType.HSCROLL_CARDS
    cards: [
      {
        body: { text: 'Card 1' },
        nativeFlowMessage: {
          buttons: [{ name: 'quick_reply', buttonParamsJson: '{"display_text":"Choose","id":"card1"}' }]
        }
      },
      {
        body: { text: 'Card 2' },
        nativeFlowMessage: {
          buttons: [{ name: 'quick_reply', buttonParamsJson: '{"display_text":"Choose","id":"card2"}' }]
        }
      }
    ]
  }
})
```

### Bot Capabilities Advertisement

Tell bots what rich response features your client supports.

```js
const { BotCapabilityType } = require('baron-baileys-v2')

await sock.sendMessage(botJid, {
  text: 'Hello!',
  botCapabilities: [
    'RICH_RESPONSE_TABLE',     // or BotCapabilityType.RICH_RESPONSE_TABLE
    'RICH_RESPONSE_CODE',
    'RICH_RESPONSE_LATEX',
    'RICH_RESPONSE_MAPS',
    'RICH_RESPONSE_INLINE_IMAGE',
    'RICH_RESPONSE_GRID_IMAGE'
  ]
})
// The bot now knows to send tables, code blocks, LaTeX, maps, etc.
```

### Message Association (Albums, HD Video, etc.)

Link a message to a parent message for album grouping, HD uploads, etc.

```js
const { AssociationType } = require('baron-baileys-v2')

// Mark an image as part of an album
await sock.sendMessage(jid, {
  image: { url: './photo.jpg' },
  messageAssociation: {
    type: 'MEDIA_ALBUM',                // or AssociationType.MEDIA_ALBUM
    parentMessageKey: { id: 'ALBUM_MSG_ID', remoteJid: jid },
    messageIndex: 0                     // position in album
  }
})

// HD video dual upload
await sock.sendMessage(jid, {
  video: { url: './hd.mp4' },
  messageAssociation: {
    type: 'HD_VIDEO_DUAL_UPLOAD',
    parentMessageKey: { id: 'SD_VIDEO_MSG_ID', remoteJid: jid }
  }
})
```

### AI Thread ID

Attach a thread ID to a message for AI conversation tracking.

```js
const { ThreadType } = require('baron-baileys-v2')

await sock.sendMessage(botJid, {
  text: 'Continue our conversation',
  threadId: {
    type: 'ai',                         // or ThreadType.AI_THREAD
    key: { id: 'THREAD_MSG_ID', remoteJid: botJid }
  }
})
```

### Feature Eligibilities

Control per-message interaction features.

```js
await sock.sendMessage(jid, {
  text: 'This cannot be reacted to or reshared',
  featureEligibilities: {
    cannotBeReactedTo: true,
    cannotBeRanked: true,
    canBeReshared: false,
    canReceiveMultiReact: false
  }
})
```

### Enhanced Chat Theme

Set different wallpaper types when changing chat themes.

```js
// Solid color wallpaper (light + dark mode)
await sock.sendMessage(jid, {
  chatTheme: {
    solidColor: {
      colorLight: '#4285F4',
      colorDark: '#1A73E8',
      isDoodleEnabled: false
    }
  }
})

// Stock image wallpaper
await sock.sendMessage(jid, {
  chatTheme: {
    stockImage: {
      id: 'wallpaper_flowers_01',
      dimLevel: 0.3
    }
  }
})

// Default doodle wallpaper
await sock.sendMessage(jid, {
  chatTheme: {
    defaultWallpaper: { isDoodleEnabled: true }
  }
})

// Reset to no theme
await sock.sendMessage(jid, {
  chatTheme: { clear: true }
})
```

### Forward Origin & Status Audience

Add metadata about where a message was forwarded from, or restrict status visibility.

```js
const { ForwardOrigin } = require('baron-baileys-v2')

// Mark as forwarded from a channel
await sock.sendMessage(jid, {
  text: 'Check this out!',
  forwardOrigin: 'channels'      // or ForwardOrigin.CHANNELS
  // 'chat' | 'status' | 'channels' | 'meta_ai' | 'ugc'
})

// Send status visible only to Close Friends
await sock.sendMessage('status@broadcast', {
  text: 'Private update 🔒',
  statusAudienceMetadata: {
    closeFriends: true,
    listName: 'Close Friends',
    listEmoji: '⭐'
  }
})
```

### AI-Generated Video Flag

Mark a video as AI-generated (sets `VideoMessage.videoSourceType = AI_GENERATED`).

```js
const { VideoSourceType } = require('baron-baileys-v2')

await sock.sendMessage(jid, {
  video: { url: './ai-video.mp4' },
  caption: 'AI-generated content',
  aiGenerated: true    // sets videoSourceType = AI_GENERATED
})
```

### AI Media Collection

Start an AI media collection session (batches multiple media into one response).

```js
await sock.sendMessage(jid, {
  aiMediaCollection: {
    collectionId: 'my-collection-id',   // auto-generated if omitted
    expectedMediaCount: 4,              // how many media items to expect
    hasGlobalCaption: true
  }
})

// React to incoming AI media collections
sock.ev.on('bot.media-collection', ({ collection, chatId }) => {
  console.log('Collection:', collection.collectionId, '| Count:', collection.expectedMediaCount)
})
```

### Chat Notes / Drafts

Create, update, or delete a note/draft for any chat.

```js
// Set a note for a chat
await sock.setChatNote(jid, 'Follow up on Monday')

// Delete the note
await sock.deleteChatNote(jid)
```

### Contact Status Mute / Favorites

```js
// Mute status updates from a contact (they won't appear in your status feed)
await sock.muteContactStatus('alice@s.whatsapp.net', true)
await sock.muteContactStatus('alice@s.whatsapp.net', false)  // unmute

// Add to / remove from favorites
await sock.toggleFavorite('alice@s.whatsapp.net', true)
await sock.toggleFavorite('alice@s.whatsapp.net', false)
```

### Call Log Management & Label Reordering

```js
// Delete a specific call log entry
await sock.deleteCallLog('CALL_ID_FROM_LOG')

// Reorder labels (provide new sorted order)
await sock.reorderLabels(['label-id-3', 'label-id-1', 'label-id-2'])

// React to label reorder sync
sock.ev.on('labels.reorder', ({ labelIds }) => {
  console.log('New label order:', labelIds)
})
```

### New Incoming Events (Round 7)

```js
// Quarantined message (suspicious message flagged by WhatsApp)
sock.ev.on('message.quarantined', ({ key, quarantineInfo }) => {
  console.log('Quarantined:', key.id)
})

// Interactive galaxy flow completed
sock.ev.on('galaxy.flow.completed', ({ key, chatId }) => {
  console.log('Galaxy flow done for', chatId)
})

// AI PSI metadata received
sock.ev.on('bot.psi-metadata', ({ psiMetadata }) => {
  console.log('PSI metadata:', psiMetadata)
})

// AI query fanout (AI response being distributed)
sock.ev.on('bot.query-fanout', ({ queryFanout }) => {
  console.log('Query fanout for message:', queryFanout?.messageKey?.id)
})

// Status mentioned via ProtocolMessage
sock.ev.on('status.mention', ({ statusMention, chatId }) => {
  console.log('Status mentioned in', chatId)
})
```

### Quiz Polls

Create a poll with a correct answer (quiz mode), deadline, and extended options.

```js
// Quiz poll — only one correct answer
await sock.sendMessage(jid, {
  poll: {
    name: 'Capital of France?',
    values: ['Paris', 'London', 'Berlin'],
    selectableCount: 1,
    type: 'quiz',                 // enables quiz mode
    correctAnswer: 'Paris',       // the right answer
    hideParticipantName: true,    // anonymous votes
    allowAddOption: true,         // participants can add options
    endTime: new Date(Date.now() + 86400000)  // poll ends in 24h
  }
})

// Poll V6 (latest format)
await sock.sendMessage(jid, {
  poll: {
    name: 'Favourite color?',
    values: ['Red', 'Blue', 'Green'],
    selectableCount: 0,   // 0 = multiple choice
    v6: true
  }
})

// Get the correct answer from a quiz poll message
const { getPollCorrectAnswer } = require('baron-baileys-v2')
const answer = getPollCorrectAnswer(msg.message)   // 'Paris' or null
```

### Timed / After-Read Duration Messages

Send a message that auto-deletes after the recipient reads it (like Snapchat-style).

```js
await sock.sendMessage(jid, {
  text: 'This message vanishes 30 seconds after reading!',
  afterReadDuration: 30    // seconds; sets contextInfo.afterReadDuration
})

// Works with media too
await sock.sendMessage(jid, {
  image: { url: './secret.jpg' },
  caption: 'Gone in 10s',
  afterReadDuration: 10
})
```

### Message Utility Helpers

```js
const {
  isScheduledMessage,
  getScheduledMessageTime,
  getMessagePaymentInfo,
  getMessageCommentMetadata,
  getMessageAddOns,
  getPollCorrectAnswer
} = require('baron-baileys-v2')

// Check if a message is scheduled (conditional reveal)
if (isScheduledMessage(msg)) {
  const revealAt = getScheduledMessageTime(msg)
  console.log('Reveals at:', revealAt)
}

// Get payment info from WebMessageInfo
const payment = getMessagePaymentInfo(msg)
if (payment) {
  console.log('Payment status:', payment.status, '| Amount:', payment.amount1000)
}

// Get comment thread metadata
const comments = getMessageCommentMetadata(msg)

// Get all message add-ons (reactions, pins, etc.)
const addOns = getMessageAddOns(msg)  // MessageAddOn[]
```

### Community — New Methods

```js
// Update community profile picture
await sock.communityUpdatePicture('123@g.us', fs.readFileSync('./avatar.jpg'))
await sock.communityRemovePicture('123@g.us')

// Set community settings (announce-only, not_announcement, etc.)
await sock.communitySettingUpdate('123@g.us', 'announcement')

// Toggle ephemeral messages for the whole community
await sock.communityToggleEphemeral('123@g.us', 604800)  // 7 days
await sock.communityToggleEphemeral('123@g.us', 0)       // disable

// Get community info from an invite code
const info = await sock.communityGetInviteInfo('ABC123XYZ')
console.log(info.subject, info.size)

// Deactivate / delete a community
await sock.communityDeactivate('123@g.us')
```

### New Payment and Scheduling Events

```js
// Payment split received
sock.ev.on('payment.split', ({ splitPayment, chatId }) => {
  console.log('Split:', splitPayment.description, '| Total:', splitPayment.totalAmount)
  splitPayment.participants.forEach(p => console.log(p.jid, p.status))
})

// P2P payment reminder
sock.ev.on('payment.reminder', ({ reminder }) => {
  console.log('Reminder:', reminder.description, '| State:', reminder.state)
})

// Poll option added by participant
sock.ev.on('poll.add-option', ({ pollKey, addedOption, addedBy }) => {
  console.log(addedBy, 'added option:', addedOption.optionName)
})

// Scheduled call created
sock.ev.on('call.scheduled', ({ scheduledCall, chatId }) => {
  const ts = Number(scheduledCall.scheduledTimestampMs)
  console.log('Call at:', new Date(ts), '| Video:', scheduledCall.callType === 2)
})

// Scheduled call cancelled
sock.ev.on('call.schedule-cancelled', ({ editedCallKey }) => {
  console.log('Cancelled call:', editedCallKey.id)
})
```

### Keep in Chat Message

Keep or un-keep a message visible for all participants.

```js
const { KeepType } = require('baron-baileys-v2')

// Keep a message for all
await sock.sendMessage(jid, {
  keepInChat: {
    key: { id: 'MSG_ID', remoteJid: jid },
    keepType: 'keep'   // or KeepType.KEEP_FOR_ALL
  }
})

// Undo keep
await sock.sendMessage(jid, {
  keepInChat: {
    key: { id: 'MSG_ID', remoteJid: jid },
    keepType: 'undo'   // or KeepType.UNDO_KEEP_FOR_ALL
  }
})
```

### Bot Feedback Message

Send thumbs-up/down feedback on a bot response.

```js
const { BotFeedbackKind } = require('baron-baileys-v2')

// Thumbs up
await sock.sendMessage(jid, {
  botFeedback: {
    key: { id: 'BOT_MSG_ID', remoteJid: jid },
    kind: 'positive'                      // or BotFeedbackKind.BOT_FEEDBACK_POSITIVE
  }
})

// Thumbs down with reason
await sock.sendMessage(jid, {
  botFeedback: {
    key: { id: 'BOT_MSG_ID', remoteJid: jid },
    kind: 'negative_helpful'              // not helpful
  }
})

// Listen for incoming feedback
sock.ev.on('bot.feedback', ({ key, botFeedback, targetKey }) => {
  console.log('Feedback received:', botFeedback.kind)
})
```

### Poll Add Option

Add a new option to an existing poll (requires `allowAddOption: true` in poll creation).

```js
await sock.sendMessage(jid, {
  pollAddOption: {
    key: { id: 'POLL_MSG_ID', remoteJid: jid },
    optionName: 'Option D'
  }
})
```

### Chat Theme

Change the color scheme of a chat.

```js
// Set a color theme
await sock.sendMessage(jid, {
  chatTheme: { colorSchemeId: 'blue-500' }
})

// Clear the theme (reset to default)
await sock.sendMessage(jid, {
  chatTheme: { clear: true }
})

// React to theme changes
sock.ev.on('chats.update', ([chat]) => {
  if (chat.chatThemeSetting) {
    console.log('Theme changed:', chat.chatThemeSetting.colorSchemeId)
  }
})
```

### Stop AI Generation / Unschedule Message

```js
// Cancel an in-progress AI response
await sock.sendMessage(jid, {
  stopGeneration: { id: 'BOT_MSG_ID', remoteJid: jid }
})

// Cancel a scheduled message
await sock.sendMessage(jid, {
  unscheduleMessage: {
    key: { id: 'SCHEDULED_MSG_ID', remoteJid: jid }
  }
})
```

### Cloud API Thread Control (Receiving)

```js
sock.ev.on('cloud.thread.control', ({ notification, chatId }) => {
  const { CloudAPIThreadControl } = require('baron-baileys-v2')
  switch (notification.status) {
    case CloudAPIThreadControl.CONTROL_PASSED:
      console.log('Human agent handed off to bot at', chatId)
      break
    case CloudAPIThreadControl.CONTROL_TAKEN:
      console.log('Bot handed off to human agent at', chatId)
      break
  }
})
```

### Broadcast Call Message

Send a broadcast call (voice or video) to a group or broadcast list.

```js
const { BCallMediaType } = require('baron-baileys-v2')

await sock.sendMessage(jid, {
  bcall: {
    mediaType: 'video',        // or BCallMediaType.VIDEO
    caption: 'Join our call!',
    sessionId: 'my-session-123'   // optional, auto-generated
  }
})
```

### Live Location Update / Stop

Update coordinates of an existing live location share, or stop it.

```js
// Update live location (send to the same jid as the original message)
await sock.sendMessage(jid, {
  liveLocationUpdate: {
    latitude: 48.8566,
    longitude: 2.3522,
    accuracy: 10,     // meters
    speed: 1.4,       // m/s
    heading: 90,      // degrees clockwise from north
    sequence: 2       // increment per update
  }
})

// Stop sharing live location (revokes the message)
await sock.sendMessage(jid, {
  stopLiveLocation: { id: 'LIVE_LOCATION_MSG_ID', remoteJid: jid }
})
```

### Chat Management — New Methods

```js
// Mark a chat as explicitly unread (shows unread dot)
await sock.markChatAsUnread(jid)

// Set disappearing messages for a specific chat
await sock.setChatEphemeral(jid, 86400)    // 24 hours
await sock.setChatEphemeral(jid, 604800)   // 7 days
await sock.setChatEphemeral(jid, 0)        // disable

// Silence/mute a chat permanently
await sock.silenceChat(jid, true)
// Mute until a specific time
await sock.silenceChat(jid, true, Date.now() + 8 * 3600 * 1000)
// Unmute
await sock.silenceChat(jid, false)
```

### Bot Profile Lookup

Fetch detailed bot profiles for one or more bot JIDs.

```js
const { USyncBotProfileProtocol } = require('baron-baileys-v2')

// Batch fetch bot profiles
const profiles = await sock.fetchBotProfiles([
  '259786046210223@bot',
  '987654321@bot'
])
profiles.forEach(p => {
  console.log(p.jid, '|', p.name, '|', p.description)
  console.log('Commands:', p.commands)
  console.log('Prompts:', p.prompts)
})
```

### Newsletter — Extended Methods

```js
// Change the newsletter category/topic
await sock.newsletterUpdateCategory('120363xxx@newsletter', 'Technology')

// Update settings object
await sock.newsletterUpdateSettings('120363xxx@newsletter', { joinApproval: 'required' })

// Promote a follower to admin
await sock.newsletterPromoteAdmin('120363xxx@newsletter', 'alice@s.whatsapp.net')

// Pin a newsletter message (for 24h by default)
await sock.newsletterPinMessage('120363xxx@newsletter', SERVER_ID, 86400)
// Unpin
await sock.newsletterUnpinMessage('120363xxx@newsletter', SERVER_ID)

// Get message stats/reach
const stats = await sock.newsletterViewStats('120363xxx@newsletter', SERVER_ID)
```

### New Events (Round 5)

```js
// Incoming comment on a message
sock.ev.on('message.comment', ({ comment, commentKey, targetKey, encrypted }) => {
  console.log('Comment on', targetKey.id, ':', comment)
})

// Message kept/unkept for all
sock.ev.on('messages.update', ([{ key, update }]) => {
  if (update.keepInChat) {
    console.log('Keep status:', update.keepInChat.keepType, 'for', key.id)
  }
})

// Broadcast call received
sock.ev.on('call', ([call]) => {
  if (call.isBroadcast) {
    console.log('Broadcast', call.isVideo ? 'video' : 'audio', 'call from', call.from)
  }
})
```

### Community / Subgroup Management

```js
// Create a subgroup inside a community
const subgroup = await sock.groupCreateSubgroup(
  '1234@g.us',       // community JID
  'Dev Team',        // subgroup name
  ['alice@s.whatsapp.net', 'bob@s.whatsapp.net']  // initial members
)

// Link an existing group to a community
await sock.groupLinkToCommunity('1234@g.us', '5678@g.us')

// Unlink a subgroup from a community
await sock.groupUnlinkFromCommunity('1234@g.us', '5678@g.us')

// List all subgroups of a community
const subgroups = await sock.groupGetLinkedSubgroups('1234@g.us')
// [{ jid: '5678@g.us', linkType: 'sub_group' }, ...]

// Get past participants (left/removed)
const past = await sock.groupGetPastParticipants('1234@g.us')
// [{ jid: 'alice@s.whatsapp.net', leaveReason: 'left', leaveTimestamp: 1700000000 }, ...]

// Tag a group member with a label
await sock.groupSetMemberLabel('1234@g.us', 'alice@s.whatsapp.net', 'moderator')
// Remove label
await sock.groupSetMemberLabel('1234@g.us', 'alice@s.whatsapp.net', '')
```

### Tap Link / Action Link Message

Send a message with a tappable link button.

```js
await sock.sendMessage(jid, {
  tapLink: {
    title: 'Visit our website',
    url: 'https://example.com',
    buttonTitle: 'Open'        // optional
  }
})
```

### Citation Message

Send an AI-style citation / source reference.

```js
await sock.sendMessage(jid, {
  citation: {
    title: 'Wikipedia',
    subtitle: 'Free encyclopedia',
    text: 'Paris is the capital of France.'
  }
})
```

### Embedded Music Message

Send a music attribution text message.

```js
await sock.sendMessage(jid, {
  embeddedMusic: {
    title: 'Bohemian Rhapsody',
    author: 'Queen',
    text: 'Now playing:'    // optional prefix
  }
})
```

### Spoiler / Question Flags

Mark any message as a spoiler or a question via `contextInfo`.

```js
// Spoiler
await sock.sendMessage(jid, {
  text: 'Snape kills Dumbledore',
  isSpoiler: true
})

// Question
await sock.sendMessage(jid, {
  text: 'What is 2 + 2?',
  isQuestion: true
})

// Works on media too
await sock.sendMessage(jid, {
  image: { url: './surprise.jpg' },
  caption: 'Surprise ending!',
  isSpoiler: true
})
```

### Chat Lock / Wallpaper / Media Visibility

```js
// Lock a chat (requires device screen lock to open)
await sock.updateChatLock(jid, true)
await sock.updateChatLock(jid, false)    // unlock

// Set a custom wallpaper (directPath from upload)
await sock.updateChatWallpaper(jid, {
  directPath: '/v/t62.7118-24/...',
  mediaKey: Buffer,
  fileEncSha256: Buffer,
  fileSha256: Buffer,
  dimLevel: 0.3              // 0.0–1.0 dimming
})
// Remove wallpaper
await sock.updateChatWallpaper(jid, null)

// Auto-download media setting per chat
await sock.updateChatMediaVisibility(jid, 'on')      // always download
await sock.updateChatMediaVisibility(jid, 'off')     // never download
await sock.updateChatMediaVisibility(jid, 'default') // use global setting
```

### History Sync — New Fields

The `messaging-history.set` event now includes additional fields from `HistorySync`:

```js
sock.ev.on('messaging-history.set', ({
  chats, contacts, messages,   // existing
  globalSettings,              // GlobalSettings: font size, wallpaper, notification prefs
  callLogRecords,              // CallLogRecord[]: full call history
  recentStickers,              // StickerMetadata[]: recently used stickers
  pastParticipants,            // PastParticipants[]: who left groups and when
  statusV3Messages,            // WebMessageInfo[]: status messages from bootstrap
  isLatest
}) => {
  if (globalSettings) {
    console.log('Font size:', globalSettings.fontSize)
    console.log('Disappearing mode:', globalSettings.disappearingModeDuration)
  }
  if (callLogRecords?.length) {
    console.log('Call log entries:', callLogRecords.length)
    // { isVideo, callResult, durationSecs, participants: [...] }
  }
  if (recentStickers?.length) {
    console.log('Recent stickers:', recentStickers.map(s => s.url))
  }
})
```

### AI Thread Management

```js
// Rename an AI chat thread
await sock.renameAIThread(jid, 'My Research Thread')

// Pin a message in a thread
await sock.pinThreadMessage(jid, 'MESSAGE_ID', true)

// Unpin
await sock.pinThreadMessage(jid, 'MESSAGE_ID', false)

// Enable AI end-to-end private processing
await sock.updatePrivateProcessingSetting(true)
// Disable
await sock.updatePrivateProcessingSetting(false)
```

### Bio Privacy

```js
// Set who can see your About/bio text
await sock.updateBioPrivacy('all')       // everyone
await sock.updateBioPrivacy('contacts')  // contacts only
await sock.updateBioPrivacy('none')      // nobody
```

### Bot Management

```js
// Fetch available bots
const bots = await sock.getBotListV2()
// [{ jid: '...@bot', personaId: '...' }, ...]

// Get details for a specific bot
const profile = await sock.getBotProfile('259786046210223@bot')
// { jid, personaId, name, description }

// Block / unblock a bot
await sock.blockBot('259786046210223@bot')
await sock.unblockBot('259786046210223@bot')
```

### Sending Messages with Link Previews

1. By default, wa does not have link generation when sent from the web
2. Baileys has a function to generate the content for these link previews
3. To enable this function's usage, add `link-preview-js` as a dependency to your project with `yarn add link-preview-js`
4. Send a link:

```ts
await sock.sendMessage(jid, {
	text: 'Hi, this was sent using https://github.com/Barons-Team/baron-baileys-v2'
})
```

### Media Messages

Sending media (video, stickers, images) is easier & more efficient than ever.

> [!NOTE]
> In media messages, you can pass `{ stream: Stream }` or `{ url: Url }` or `Buffer` directly, you can see more [here](https://baileys.whiskeysockets.io/types/WAMediaUpload.html)

- When specifying a media url, Baileys never loads the entire buffer into memory it even encrypts the media as a readable stream.

> [!TIP]
> It's recommended to use Stream or Url to save memory

#### Gif Message

- Whatsapp doesn't support `.gif` files, that's why we send gifs as common `.mp4` video with `gifPlayback` flag

```ts
await sock.sendMessage(jid, {
	video: fs.readFileSync('Media/ma_gif.mp4'),
	caption: 'hello word',
	gifPlayback: true
})
```

#### Video Message

```ts
await sock.sendMessage(id, {
	video: {
		url: './Media/ma_gif.mp4'
	},
	caption: 'hello word'
})
```

#### Video Ptv Message

```ts
await sock.sendMessage(id, {
	video: {
		url: './Media/ma_gif.mp4'
	},
	ptv: true
})
```

#### Audio Message

- To audio message work in all devices you need to convert with some tool like `ffmpeg` with this flags:

  ```bash
      codec: libopus //ogg file
      ac: 1 //one channel
      avoid_negative_ts
      make_zero
  ```

  - Example:

  ```bash
  ffmpeg -i input.mp4 -avoid_negative_ts make_zero -ac 1 output.ogg
  ```

```ts
await sock.sendMessage(jid, {
	audio: {
		url: './Media/audio.mp3'
	},
	mimetype: 'audio/mp4'
})
```

#### Image Message

```ts
await sock.sendMessage(id, {
	image: {
		url: './Media/ma_img.png'
	},
	caption: 'hello word'
})
```

#### View Once Message

- You can send all messages above as `viewOnce`, you only need to pass `viewOnce: true` in content object

```ts
await sock.sendMessage(id, {
	image: {
		url: './Media/ma_img.png'
	},
	viewOnce: true, //works with video, audio too
	caption: 'hello word'
})
```

## Modify Messages

### Deleting Messages (for everyone)

```ts
const msg = await sock.sendMessage(jid, { text: 'hello word' })
await sock.sendMessage(jid, { delete: msg.key })
```

**Note:** deleting for oneself is supported via `chatModify`, see in [this section](#modifying-chats)

### Editing Messages

- You can pass all editable contents here

```ts
await sock.sendMessage(jid, {
	text: 'updated text goes here',
	edit: response.key
})
```

## Manipulating Media Messages

### Thumbnail in Media Messages

- For media messages, the thumbnail can be generated automatically for images & stickers provided you add `jimp` or `sharp` as a dependency in your project using `yarn add jimp` or `yarn add sharp`.
- Thumbnails for videos can also be generated automatically, though, you need to have `ffmpeg` installed on your system.

### Downloading Media Messages

If you want to save the media you received

```ts
import { createWriteStream } from 'fs'
import { downloadMediaMessage, getContentType } from 'baron-baileys-v2'

sock.ev.on('messages.upsert', async ({ [m] }) => {
    if (!m.message) return // if there is no text or media message
    const messageType = getContentType(m) // get what type of message it is (text, image, video...)

    // if the message is an image
    if (messageType === 'imageMessage') {
        // download the message
        const stream = await downloadMediaMessage(
            m,
            'stream', // can be 'buffer' too
            { },
            {
                logger,
                // pass this so that baileys can request a reupload of media
                // that has been deleted
                reuploadRequest: sock.updateMediaMessage
            }
        )
        // save to file
        const writeStream = createWriteStream('./my-download.jpeg')
        stream.pipe(writeStream)
    }
}
```

### Re-upload Media Message to Whatsapp

- WhatsApp automatically removes old media from their servers. For the device to access said media -- a re-upload is required by another device that has it. This can be accomplished using:

```ts
await sock.updateMediaMessage(msg)
```

## Reject Call

- You can obtain `callId` and `callFrom` from `call` event

```ts
await sock.rejectCall(callId, callFrom)
```

## Send States in Chat

### Reading Messages

- A set of message [keys](https://baileys.whiskeysockets.io/types/WAMessageKey.html) must be explicitly marked read now.
- You cannot mark an entire 'chat' read as it were with Baileys Web.
  This means you have to keep track of unread messages.

```ts
const key: WAMessageKey
// can pass multiple keys to read multiple messages as well
await sock.readMessages([key])
```

The message ID is the unique identifier of the message that you are marking as read.
On a `WAMessage`, the `messageID` can be accessed using `messageID = message.key.id`.

### Update Presence

- `presence` can be one of [these](https://baileys.whiskeysockets.io/types/WAPresence.html)
- The presence expires after about 10 seconds.
- This lets the person/group with `jid` know whether you're online, offline, typing etc.

```ts
await sock.sendPresenceUpdate('available', jid)
```

> [!NOTE]
> If a desktop client is active, WA doesn't send push notifications to the device. If you would like to receive said notifications -- mark your Baileys client offline using `sock.sendPresenceUpdate('unavailable')`

## Modifying Chats

WA uses an encrypted form of communication to send chat/app updates. This has been implemented mostly and you can send the following updates:

> [!IMPORTANT]
> If you mess up one of your updates, WA can log you out of all your devices and you'll have to log in again.

### Archive a Chat

```ts
const lastMsgInChat = await getLastMessageInChat(jid) // implement this on your end
await sock.chatModify({ archive: true, lastMessages: [lastMsgInChat] }, jid)
```

### Mute/Unmute a Chat

- Supported times:

| Time   | Miliseconds |
| ------ | ----------- |
| Remove | null        |
| 8h     | 86.400.000  |
| 7d     | 604.800.000 |

```ts
// mute for 8 hours
await sock.chatModify({ mute: 8 * 60 * 60 * 1000 }, jid)
// unmute
await sock.chatModify({ mute: null }, jid)
```

### Mark a Chat Read/Unread

```ts
const lastMsgInChat = await getLastMessageInChat(jid) // implement this on your end
// mark it unread
await sock.chatModify({ markRead: false, lastMessages: [lastMsgInChat] }, jid)
```

### Delete a Message for Me

```ts
await sock.chatModify(
	{
		clear: {
			messages: [
				{
					id: 'ATWYHDNNWU81732J',
					fromMe: true,
					timestamp: '1654823909'
				}
			]
		}
	},
	jid
)
```

### Delete a Chat

```ts
const lastMsgInChat = await getLastMessageInChat(jid) // implement this on your end
await sock.chatModify(
	{
		delete: true,
		lastMessages: [
			{
				key: lastMsgInChat.key,
				messageTimestamp: lastMsgInChat.messageTimestamp
			}
		]
	},
	jid
)
```

### Pin/Unpin a Chat

```ts
await sock.chatModify(
	{
		pin: true // or `false` to unpin
	},
	jid
)
```

### Star/Unstar a Message

```ts
await sock.chatModify(
	{
		star: {
			messages: [
				{
					id: 'messageID',
					fromMe: true // or `false`
				}
			],
			star: true // - true: Star Message false: Unstar Message
		}
	},
	jid
)
```

### Disappearing Messages

- Ephemeral can be:

| Time   | Seconds   |
| ------ | --------- |
| Remove | 0         |
| 24h    | 86.400    |
| 7d     | 604.800   |
| 90d    | 7.776.000 |

- You need to pass in **Seconds**, default is 7 days

```ts
// turn on disappearing messages
await sock.sendMessage(
	jid,
	// this is 1 week in seconds -- how long you want messages to appear for
	{ disappearingMessagesInChat: WA_DEFAULT_EPHEMERAL }
)

// will send as a disappearing message
await sock.sendMessage(jid, { text: 'hello' }, { ephemeralExpiration: WA_DEFAULT_EPHEMERAL })

// turn off disappearing messages
await sock.sendMessage(jid, { disappearingMessagesInChat: false })
```

### Clear Messages

```ts
await sock.clearMessage(jid, key, timestamps)
```

## User Querys

### Check If ID Exists in Whatsapp

```ts
const [result] = await sock.onWhatsApp(jid)
if (result.exists) console.log(`${jid} exists on WhatsApp, as jid: ${result.jid}`)
```

### Query Chat History (groups too)

- You need to have oldest message in chat

```ts
const msg = await getOldestMessageInChat(jid)
await sock.fetchMessageHistory(
	50, //quantity (max: 50 per query)
	msg.key,
	msg.messageTimestamp
)
```

- Messages will be received in `messaging.history-set` event

### Fetch Status

```ts
const status = await sock.fetchStatus(jid)
console.log('status: ' + status)
```

### Fetch Profile Picture

- To get the display picture of some person, group and channel

```ts
// for low res picture
const ppUrl = await sock.profilePictureUrl(jid)
console.log(ppUrl)
```

### Fetch Bussines Profile (such as description or category)

```ts
const profile = await sock.getBusinessProfile(jid)
console.log('business description: ' + profile.description + ', category: ' + profile.category)
```

### Fetch Someone's Presence (if they're typing or online)

```ts
// the presence update is fetched and called here
sock.ev.on('presence.update', console.log)

// request updates for a chat
await sock.presenceSubscribe(jid)
```

## Change Profile

### Change Profile Status

```ts
await sock.updateProfileStatus('Hello World!')
```

### Change Profile Name

```ts
await sock.updateProfileName('My name')
```

### Change Display Picture (groups too)

- To change your display picture or a group's

> [!NOTE]
> Like media messages, you can pass `{ stream: Stream }` or `{ url: Url }` or `Buffer` directly, you can see more [here](https://baileys.whiskeysockets.io/types/WAMediaUpload.html)

```ts
await sock.updateProfilePicture(jid, { url: './new-profile-picture.jpeg' })
```

### Remove display picture (groups too)

```ts
await sock.removeProfilePicture(jid)
```

## Groups

- To change group properties you need to be admin

### Create a Group

```ts
// title & participants
const group = await sock.groupCreate('My Fab Group', ['1234@s.whatsapp.net', '4564@s.whatsapp.net'])
console.log('created group with id: ' + group.gid)
await sock.sendMessage(group.id, { text: 'hello there' }) // say hello to everyone on the group
```

### Add/Remove or Demote/Promote

```ts
// id & people to add to the group (will throw error if it fails)
await sock.groupParticipantsUpdate(
	jid,
	['abcd@s.whatsapp.net', 'efgh@s.whatsapp.net'],
	'add' // replace this parameter with 'remove' or 'demote' or 'promote'
)
```

### Change Subject (name)

```ts
await sock.groupUpdateSubject(jid, 'New Subject!')
```

### Change Description

```ts
await sock.groupUpdateDescription(jid, 'New Description!')
```

### Change Settings

```ts
// only allow admins to send messages
await sock.groupSettingUpdate(jid, 'announcement')
// allow everyone to send messages
await sock.groupSettingUpdate(jid, 'not_announcement')
// allow everyone to modify the group's settings -- like display picture etc.
await sock.groupSettingUpdate(jid, 'unlocked')
// only allow admins to modify the group's settings
await sock.groupSettingUpdate(jid, 'locked')
```

### Leave a Group

```ts
// will throw error if it fails
await sock.groupLeave(jid)
```

### Get Invite Code

- To create link with code use `'https://chat.whatsapp.com/' + code`

```ts
const code = await sock.groupInviteCode(jid)
console.log('group code: ' + code)
```

### Revoke Invite Code

```ts
const code = await sock.groupRevokeInvite(jid)
console.log('New group code: ' + code)
```

### Join Using Invitation Code

- Code can't have `https://chat.whatsapp.com/`, only code

```ts
const response = await sock.groupAcceptInvite(code)
console.log('joined to: ' + response)
```

### Get Group Info by Invite Code

```ts
const response = await sock.groupGetInviteInfo(code)
console.log('group information: ' + response)
```

### Query Metadata (participants, name, description...)

```ts
const metadata = await sock.groupMetadata(jid)
console.log(metadata.id + ', title: ' + metadata.subject + ', description: ' + metadata.desc)
```

### Join using `groupInviteMessage`

```ts
const response = await sock.groupAcceptInviteV4(jid, groupInviteMessage)
console.log('joined to: ' + response)
```

### Get Request Join List

```ts
const response = await sock.groupRequestParticipantsList(jid)
console.log(response)
```

### Approve/Reject Request Join

```ts
const response = await sock.groupRequestParticipantsUpdate(
	jid, // group id
	['abcd@s.whatsapp.net', 'efgh@s.whatsapp.net'],
	'approve' // or 'reject'
)
console.log(response)
```

### Get All Participating Groups Metadata

```ts
const response = await sock.groupFetchAllParticipating()
console.log(response)
```

### Toggle Ephemeral

- Ephemeral can be:

| Time   | Seconds   |
| ------ | --------- |
| Remove | 0         |
| 24h    | 86.400    |
| 7d     | 604.800   |
| 90d    | 7.776.000 |

```ts
await sock.groupToggleEphemeral(jid, 86400)
```

### Change Add Mode

```ts
await sock.groupMemberAddMode(
	jid,
	'all_member_add' // or 'admin_add'
)
```

## Privacy

### Block/Unblock User

```ts
await sock.updateBlockStatus(jid, 'block') // Block user
await sock.updateBlockStatus(jid, 'unblock') // Unblock user
```

### Get Privacy Settings

```ts
const privacySettings = await sock.fetchPrivacySettings(true)
console.log('privacy settings: ' + privacySettings)
```

### Get BlockList

```ts
const response = await sock.fetchBlocklist()
console.log(response)
```

### Update LastSeen Privacy

```ts
const value = 'all' // 'contacts' | 'contact_blacklist' | 'none'
await sock.updateLastSeenPrivacy(value)
```

### Update Online Privacy

```ts
const value = 'all' // 'match_last_seen'
await sock.updateOnlinePrivacy(value)
```

### Update Profile Picture Privacy

```ts
const value = 'all' // 'contacts' | 'contact_blacklist' | 'none'
await sock.updateProfilePicturePrivacy(value)
```

### Update Status Privacy

```ts
const value = 'all' // 'contacts' | 'contact_blacklist' | 'none'
await sock.updateStatusPrivacy(value)
```

### Update Read Receipts Privacy

```ts
const value = 'all' // 'none'
await sock.updateReadReceiptsPrivacy(value)
```

### Update Groups Add Privacy

```ts
const value = 'all' // 'contacts' | 'contact_blacklist'
await sock.updateGroupsAddPrivacy(value)
```

### Update Default Disappearing Mode

- Like [this](#disappearing-messages), ephemeral can be:

| Time   | Seconds   |
| ------ | --------- |
| Remove | 0         |
| 24h    | 86.400    |
| 7d     | 604.800   |
| 90d    | 7.776.000 |

```ts
const ephemeral = 86400
await sock.updateDefaultDisappearingMode(ephemeral)
```

## Broadcast Lists & Stories

### Send Broadcast & Stories

- Messages can be sent to broadcasts & stories. You need to add the following message options in sendMessage, like this:

```ts
await sock.sendMessage(
	jid,
	{
		image: {
			url: url
		},
		caption: caption
	},
	{
		backgroundColor: backgroundColor,
		font: font,
		statusJidList: statusJidList,
		broadcast: true
	}
)
```

- Message body can be a `extendedTextMessage` or `imageMessage` or `videoMessage` or `voiceMessage`, see [here](https://baileys.whiskeysockets.io/types/AnyRegularMessageContent.html)
- You can add `backgroundColor` and other options in the message options, see [here](https://baileys.whiskeysockets.io/types/MiscMessageGenerationOptions.html)
- `broadcast: true` enables broadcast mode
- `statusJidList`: a list of people that you can get which you need to provide, which are the people who will get this status message.

- You can send messages to broadcast lists the same way you send messages to groups & individual chats.
- Right now, WA Web does not support creating broadcast lists, but you can still delete them.
- Broadcast IDs are in the format `12345678@broadcast`

### Query a Broadcast List's Recipients & Name

```ts
const bList = await sock.getBroadcastListInfo('1234@broadcast')
console.log(`list name: ${bList.name}, recps: ${bList.recipients}`)
```

## Writing Custom Functionality

Baileys is written with custom functionality in mind. Instead of forking the project & re-writing the internals, you can simply write your own extensions.

### Enabling Debug Level in Baileys Logs

First, enable the logging of unhandled messages from WhatsApp by setting:

```ts
const sock = makeWASocket({
	logger: P({ level: 'debug' })
})
```

This will enable you to see all sorts of messages WhatsApp sends in the console.

### How Whatsapp Communicate With Us

> [!TIP]
> If you want to learn whatsapp protocol, we recommend to study about Libsignal Protocol and Noise Protocol

- **Example:** Functionality to track the battery percentage of your phone. You enable logging and you'll see a message about your battery pop up in the console:
  ```
  {
      "level": 10,
      "fromMe": false,
      "frame": {
          "tag": "ib",
          "attrs": {
              "from": "@s.whatsapp.net"
          },
          "content": [
              {
                  "tag": "edge_routing",
                  "attrs": {},
                  "content": [
                      {
                          "tag": "routing_info",
                          "attrs": {},
                          "content": {
                              "type": "Buffer",
                              "data": [8,2,8,5]
                          }
                      }
                  ]
              }
          ]
      },
      "msg":"communication"
  }
  ```

The `'frame'` is what the message received is, it has three components:

- `tag` -- what this frame is about (eg. message will have 'message')
- `attrs` -- a string key-value pair with some metadata (contains ID of the message usually)
- `content` -- the actual data (eg. a message node will have the actual message content in it)
- read more about this format [here](/src/WABinary/readme.md)

### Register a Callback for Websocket Events

> [!TIP]
> Recommended to see `onMessageReceived` function in `socket.ts` file to understand how websockets events are fired

```ts
// for any message with tag 'edge_routing'
sock.ws.on('CB:edge_routing', (node: BinaryNode) => {})

// for any message with tag 'edge_routing' and id attribute = abcd
sock.ws.on('CB:edge_routing,id:abcd', (node: BinaryNode) => {})

// for any message with tag 'edge_routing', id attribute = abcd & first content node routing_info
sock.ws.on('CB:edge_routing,id:abcd,routing_info', (node: BinaryNode) => {})
```

> [!NOTE]
> Also, this repo is now licenced under GPL 3 since it uses [libsignal-node](https://git.questbook.io/backend/service-coderunner/-/merge_requests/1)
