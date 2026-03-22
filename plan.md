# Aerochat modernization plan

This document converts the repository review into an actionable modernization roadmap for Aerochat.

## Product goals

Aerochat should evolve in this order:

1. **Safe to log in**.
2. **Safe to chat every day**.
3. **Safe to hang out in voice**.
4. **Pleasant enough to daily-drive**.
5. **Distinctive in its retro identity without sacrificing parity**.

---

## Phase 0 — Stabilization audit and feature inventory

**Goal:** create a truthful baseline before building new features.

### Tasks
- Audit all user-visible features into a matrix:
  - works,
  - partial,
  - UI stub,
  - broken,
  - blocked by Discord API/library,
  - blocked by architecture.
- Catalog all Discord object types currently handled in UI:
  - normal messages,
  - replies,
  - attachments,
  - embeds,
  - stickers,
  - threads,
  - forum posts,
  - polls,
  - call/voice state,
  - system messages.
- Inventory all TODO / placeholder / `NotImplementedException` paths in `Aerochat`, `Aerovoice`, and the bundled `DSharpPlus` fork.
- Add telemetry/logging around:
  - login failures,
  - message send failures,
  - voice handshake failures,
  - deserialization failures,
  - unsupported message/channel types.

### Deliverables
- `docs/feature-matrix.md`
- `docs/runtime-risk-register.md`
- issue labels for `p0`, `p1`, `compat`, `voice`, `ui-stub`

### Example code
```csharp
public enum FeatureStatus
{
    Working,
    Partial,
    Stub,
    Broken,
    BlockedByApi,
    BlockedByArchitecture,
}

public sealed record FeatureAuditItem(
    string Area,
    string Feature,
    FeatureStatus Status,
    string Notes,
    string Owner
);
```

```csharp
_logger.LogWarning(
    "Unsupported Discord payload. Type={Type} ChannelId={ChannelId} MessageId={MessageId}",
    payloadType,
    channelId,
    messageId
);
```

---

## Phase 1 — Security and compatibility blockers

**Goal:** make the client viable before polishing UX.

### Priority 1A: Replace or harden the login model
- Move away from brittle token extraction from injected JS where possible.
- If full replacement is not yet feasible, isolate the current token path behind an advanced/manual login mode.
- Encrypt locally stored session material.
- Add explicit error reporting when login extraction fails.

### Priority 1B: Voice crypto and protocol modernization
- Implement or remove unsupported crypto paths in `Aerovoice`.
- Review gateway and voice compatibility together.
- Add compatibility tests for current Discord voice handshake assumptions.

### Priority 1C: Update Discord library strategy
- Decide whether to continue maintaining the vendored `DSharpPlus` fork,
- rebase to a newer upstream,
- or wrap Discord operations behind a smaller internal API boundary.

### Exit criteria
- Login succeeds reliably.
- No user-facing `NotImplementedException` in critical networking code.
- Voice connect/join works with current Discord expectations.

### Example code
```csharp
public interface ITokenStore
{
    Task SaveAsync(string accountId, string token, CancellationToken cancellationToken = default);
    Task<string?> LoadAsync(string accountId, CancellationToken cancellationToken = default);
    Task DeleteAsync(string accountId, CancellationToken cancellationToken = default);
}
```

```csharp
public async Task<Result<string>> TryAcquireTokenAsync(CancellationToken cancellationToken)
{
    try
    {
        var token = await _loginProvider.AcquireAsync(cancellationToken);
        return Result<string>.Success(token);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to acquire Discord login token.");
        return Result<string>.Failure("login_failed", ex.Message);
    }
}
```

```csharp
public override byte[] Encrypt(byte[] data, byte[] key)
{
    if (data.Length == 0)
        throw new ArgumentException("Audio payload cannot be empty.", nameof(data));

    return _cryptoImplementation.Encrypt(data, key);
}
```

---

## Phase 2 — Core messaging parity

**Goal:** reach daily-driver text chat quality.

### Must-have features
1. **Reactions**
   - add/remove reactions,
   - render reaction bars,
   - live update counts,
   - support permission-aware UI.
2. **Message actions parity**
   - edit,
   - reply,
   - delete,
   - copy text/link/ID,
   - pin/unpin,
   - jump to reply/source.
3. **Composer parity**
   - emoji/autocomplete,
   - drag/drop files,
   - paste image/file,
   - mention/channel/emoji suggestions,
   - poll creation entry point.
4. **Message rendering parity**
   - polls,
   - stickers,
   - richer embeds,
   - voice messages if supported,
   - broader system message handling.
5. **History/navigation**
   - pagination,
   - unread separators,
   - jump-to-present,
   - channel search,
   - message permalink navigation.

### Exit criteria
- DMs and text channels are comfortable to use without falling back to the official client for basic message flows.

### Example code
```csharp
public async Task ToggleReactionAsync(DiscordMessage message, DiscordEmoji emoji, bool hasReaction)
{
    if (hasReaction)
        await message.DeleteOwnReactionAsync(emoji).ConfigureAwait(false);
    else
        await message.CreateReactionAsync(emoji).ConfigureAwait(false);
}
```

```csharp
public async Task<IReadOnlyList<DiscordMessage>> GetMessagesPageAsync(
    DiscordChannel channel,
    ulong? beforeMessageId,
    int pageSize = 50)
{
    return beforeMessageId is ulong before
        ? await channel.GetMessagesBeforeAsync(before, pageSize).ConfigureAwait(false)
        : await channel.GetMessagesAsync(pageSize).ConfigureAwait(false);
}
```

```csharp
public sealed record ComposerSuggestion(string Label, string InsertText, string Kind);
```

---

## Phase 3 — Channel model modernization

**Goal:** support current Discord information architecture.

### Must-have
- Thread discovery, open/join/leave/create/edit/delete.
- Forum channel browsing.
- Forum post creation and tag filtering.
- Poll display and voting.
- Better handling of announcement/media-style channel differences.

### Strategy
- Keep the service layer channel-type aware.
- Build reusable view models for thread/forum/post primitives.
- Avoid baking assumptions that every message lives in a flat text channel.

### Example code
```csharp
public interface IChannelNavigator
{
    Task OpenChannelAsync(ulong channelId);
    Task OpenThreadAsync(ulong threadId);
    Task OpenForumPostAsync(ulong postId);
}
```

```csharp
public bool IsModernDiscussionChannel(DiscordChannel channel)
{
    return channel.Type is ChannelType.PublicThread
        or ChannelType.PrivateThread
        or ChannelType.AnnouncementThread
        or ChannelType.GuildForum;
}
```

```csharp
public sealed class ForumTagViewModel : ViewModelBase
{
    public ulong Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public bool IsSelected { get; set; }
}
```

---

## Phase 4 — Voice/video/calls experience

**Goal:** make the client credible for modern social use.

### Must-have
- Reliable voice join/leave/device switching.
- Mute/deafen/PTT states.
- Voice member list and speaking indicators.
- Call controls in DMs/GDMs.
- Screen share / Go Live strategy.
- Video call strategy.
- Encryption/privacy indicators where possible.

### Recommendation
Split delivery into two tracks:
- **Track A:** voice reliability.
- **Track B:** video and screenshare.

### Example code
```csharp
public sealed class VoiceStateViewModel : ViewModelBase
{
    public bool IsMuted { get; set; }
    public bool IsDeafened { get; set; }
    public bool IsPushToTalkEnabled { get; set; }
    public ObservableCollection<UserViewModel> ConnectedUsers { get; } = new();
}
```

```csharp
public async Task SetInputDeviceAsync(int deviceIndex)
{
    _logger.LogInformation("Switching input device to index {DeviceIndex}", deviceIndex);
    await _voiceSession.SetInputDeviceAsync(deviceIndex).ConfigureAwait(false);
}
```

```csharp
public void OnUserSpeaking(ulong userId, bool speaking)
{
    if (_usersById.TryGetValue(userId, out var user))
        user.IsSpeaking = speaking;
}
```

---

## Phase 5 — Performance and daily-driver quality

**Goal:** make Aerochat pleasant enough to stay open all day.

### Focus areas
- Faster cold start.
- Faster channel switching.
- Lower memory churn.
- Message list virtualization.
- Attachment/avatar caching.
- Resilient reconnect behavior.
- Reduced UI-thread blocking.

### Example code
```csharp
public sealed class AvatarCache
{
    private readonly MemoryCache _cache = new(new MemoryCacheOptions
    {
        SizeLimit = 256
    });

    public void Store(string key, byte[] imageBytes)
        => _cache.Set(key, imageBytes, new MemoryCacheEntryOptions { Size = 1 });

    public bool TryGet(string key, out byte[]? imageBytes)
        => _cache.TryGetValue(key, out imageBytes);
}
```

```csharp
await Application.Current.Dispatcher.InvokeAsync(() =>
{
    VisibleMessages.Clear();
    foreach (var item in nextPage)
        VisibleMessages.Add(item);
});
```

```csharp
private CancellationTokenSource? _channelLoadCts;

public async Task ReloadChannelAsync(ulong channelId)
{
    _channelLoadCts?.Cancel();
    _channelLoadCts = new CancellationTokenSource();

    var messages = await _messageLoader
        .LoadInitialPageAsync(channelId, _channelLoadCts.Token)
        .ConfigureAwait(false);

    ApplyMessages(messages);
}
```

---

## Phase 6 — Settings, privacy, and trust

**Goal:** make users comfortable relying on the client.

### Must-have
- Clear privacy model around tokens and session storage.
- Import/export/reset settings.
- Richer notification settings.
- Accessibility review.
- Crash diagnostics that are easy to share.
- Update channel management.
- User-facing compatibility page: what works / what does not.

### Example code
```csharp
public sealed record PrivacyDisclosure(
    bool StoresDiscordToken,
    bool EncryptsAtRest,
    bool SendsTelemetry,
    string DiagnosticsLocation
);
```

```csharp
public async Task ExportSettingsAsync(Stream outputStream)
{
    await JsonSerializer.SerializeAsync(outputStream, SettingsManager.Instance, _jsonOptions)
        .ConfigureAwait(false);
}
```

```csharp
public sealed record CrashReportModel(
    DateTimeOffset Timestamp,
    string Version,
    string ExceptionType,
    string Message,
    string StackTrace
);
```

---

## Phase 7 — Aerochat identity features

**Goal:** invest in the retro identity only after parity is stable.

### Ideas
- Optional WLM-inspired nudges and sounds.
- Retro contact-list layouts.
- Scene packs and skins.
- Messenger-like status styling.
- Theme/plugin APIs after core stability.

### Guardrail
Do not let eyecandy outpace compatibility again.

### Example code
```csharp
public interface IScenePack
{
    string Id { get; }
    string DisplayName { get; }
    Uri PreviewImage { get; }
    Task ApplyAsync(CancellationToken cancellationToken = default);
}
```

```csharp
public sealed class NudgeService
{
    public event EventHandler? Nudged;

    public void Trigger()
    {
        SystemSounds.Exclamation.Play();
        Nudged?.Invoke(this, EventArgs.Empty);
    }
}
```

```csharp
public sealed record ThemeManifest(string Id, string Name, string Author, string EntryFile);
```

---

## Suggested execution order

### Month 1
1. Feature inventory + issue triage.
2. Login hardening.
3. Voice crypto audit.
4. Library fork strategy decision.

### Month 2
5. Reactions.
6. Composer/autocomplete/attachments polish.
7. Message pagination + navigation.
8. Better logging and error handling.

### Month 3
9. Threads.
10. Forum channels.
11. Polls.
12. Improved system message coverage.

### Month 4+
13. Voice UX polish.
14. DM/GDM call flows.
15. Screenshare/video investigation.
16. Performance pass.
17. Aerochat identity features.

---

## Immediate backlog

### P0
- Replace or redesign token-extraction login flow.
- Finish or remove unimplemented voice crypto modes.
- Verify current Discord compatibility for auth and voice.
- Add structured failure logging.

### P1
- Implement reactions.
- Implement poll rendering and voting.
- Add thread browse/join/open flows.
- Increase message history loading beyond a single fixed page.

### P2
- Forum channels.
- Composer suggestions and emoji UX.
- Proper DM/group call UX.
- Performance/virtualization.

### P3
- Video/screenshare.
- Activities/games/invites.
- Retro-only flourish features.
