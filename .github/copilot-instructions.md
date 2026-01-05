# SVRaidBot Copilot Instructions

## Project Overview
SVRaidBot is a .NET 9.0 automation framework for hosting Pokémon Scarlet/Violet Tera Raid battles. The bot connects to Nintendo Switch consoles via sys-botbase to inject raid seeds, manage multiplayer lobbies, post Discord embeds, and track raid statistics.

## Architecture

### Core Layer Structure
- **SysBot.Base**: Foundation layer providing Switch connection abstractions (`ISwitchConnectionAsync`), bot lifecycle management (`BotRunner<T>`, `RoutineExecutor<T>`), and synchronization primitives
- **SysBot.Pokemon**: Game-specific logic with PKHeX.Core integration for Pokémon data manipulation and raid mechanics
- **SysBot.Pokemon.WinForms**: Primary UI application with embedded HTTP server (port 9090) for web-based remote control
- **SysBot.Pokemon.Discord**: Discord.Net command modules for user interaction (raid requests via `ra`, queue management via `rqc`, raid info via `rv`)
- **SysBot.Pokemon.ConsoleApp**: Headless bot runner for server deployments

### Execution Flow
1. `PokeRaidHub<PK9>` serves as the central coordinator holding `RotatingRaidSettingsSV` configuration
2. `BotFactory9SV` instantiates `RotatingRaidBotSV` executors based on `PokeRoutineType.RotatingRaidBot`
3. Each bot's `MainLoop` cycles through raid rotation, handling seed injection → teleportation → lobby management → battle execution
4. `RaidMemoryManager` abstracts region-specific memory operations (Paldea/Kitakami/Blueberry have different pointer offsets)

### Memory Architecture
Raid data exists in three separate blocks:
- **Paldea**: Indices 0-68, base pointer + 0x20 offset per raid
- **Kitakami**: Indices 69-93, separate pointer + 0x10 offset
- **Blueberry**: Indices 94+, separate pointer + 0x10 offset

`RaidMemoryManager.InjectSeed()` and `ReadSeedAtIndex()` handle region-specific pointer arithmetic automatically.

## Key Patterns

### Bot State Management
Bots inherit from `PokeRoutineExecutorBase` → `PokeRoutineExecutor<T>` → `RotatingRaidBotSV`. Each bot maintains:
- `_hub.Config.RotatingRaidSV` for user settings
- `_raidMemoryManager` for memory I/O
- State flags like `_isPaused`, `_shouldRefreshMap`, `HasErrored`

Always use `async Task` methods with `CancellationToken token` parameters. Example from [RotatingRaidBotSV.cs](SysBot.Pokemon/SV/BotRaid/RotatingRaidBotSV.cs):
```csharp
private async Task TeleportToInjectedDenLocation(int index, TeraCrystalType crystalType, 
    string speciesName, int? groupID, string? denIdentifier, CancellationToken token)
```

### Configuration System
Settings use `[Category]`, `[Description]`, and `[TypeConverter(typeof(ExpandableObjectConverter))]` attributes for WinForms PropertyGrid binding. See [PokeTradeHubConfig.cs](SysBot.Pokemon/RaidHub/PokeTradeHubConfig.cs).

JSON serialization uses `System.Text.Json` with source-generated contexts for AOT compatibility:
```csharp
Config = JsonSerializer.Deserialize(lines, ProgramConfigContext.Default.ProgramConfig)
```

### Discord Command Modules
Commands inherit `ModuleBase<SocketCommandContext>`. Generic type parameter `T : PKM` enables multi-generation support:
```csharp
public class RaidModule<T> : ModuleBase<SocketCommandContext> where T : PKM, new()
```

Access hub via `SysCord<T>.Runner.Hub`. Common pattern:
```csharp
var settings = Hub.Config.RotatingRaidSV;
```

### Den Location System
Den coordinates loaded from embedded JSON resources using lazy initialization:
```csharp
private static readonly Lazy<Dictionary<string, float[]>> CachedPaldeaDenLocations = new(() =>
    LoadDenLocations("SysBot.Pokemon.SV.BotRaid.DenLocations.den_locations_base.json"));
```

Files: [den_locations_base.json](SysBot.Pokemon/SV/BotRaid/DenLocations/den_locations_base.json), [den_locations_kitakami.json](SysBot.Pokemon/SV/BotRaid/DenLocations/den_locations_kitakami.json), [den_locations_blueberry.json](SysBot.Pokemon/SV/BotRaid/DenLocations/den_locations_blueberry.json)

### Event Raid Mapping
`SpeciesToGroupIDMap` populated from RaidCrawler data structures maps species names to distribution group IDs:
```csharp
Dictionary<string, List<(int GroupID, int Index, string DenIdentifier)>>
```

## Development Workflows

### Building
- **Solution**: [SysBot.NET.sln](SysBot.NET.sln) contains all projects
- **Primary output**: SysBot.Pokemon.WinForms.exe (net9.0-windows)
- **Build configurations**: Debug, Release, sysbottest (x86/x64/AnyCPU)
- Azure Pipelines CI configured in [azure-pipelines.yml](azure-pipelines.yml) for automated builds

Build in Visual Studio 2022 or via CLI:
```powershell
dotnet build SysBot.NET.sln -c Release -p:Platform=x64
```

### Running/Debugging
Launch profiles in [Properties/launchSettings.json](SysBot.Pokemon.WinForms/Properties/launchSettings.json). Default configuration loads from `config.json` in the working directory (`Application.StartupPath`).

**Configuration Management**:
- Config path: `Program.ConfigPath` (defaults to `config.json` in exe directory)
- Automatic backup system creates `config.backup_*` files on corruption
- Serialization uses `System.Text.Json` with `ProgramConfigContext` for AOT compatibility

To debug raid bot specifically:
1. Set breakpoint in `RotatingRaidBotSV.MainLoop` or `InnerLoop`
2. Launch via F5 or attach to running WinForms process
3. Monitor logs via `LogUtil.LogInfo()` and `LogUtil.LogError()` output
4. Check `HasErrored` flag and `PerformRebootAndReset()` for recovery states

### Testing
Unit tests in [SysBot.Tests](SysBot.Tests/) using xUnit. Run via:
```powershell
dotnet test SysBot.Tests/SysBot.Tests.csproj
```

Test coverage includes:
- Command parsing (`CommandTests.cs`)
- PKM generation (`GenerateTests.cs`)
- Hub configuration (`PokeHubTests.cs`)
- Queue management (`QueueTests.cs`)
- String utilities (`StringTests.cs`)

### Dependency Versions
- **Framework**: .NET 9.0
- **Discord.Net**: 3.15.3+ (check [SysBot.Pokemon.Discord.csproj](SysBot.Pokemon.Discord/SysBot.Pokemon.Discord.csproj))
- **PKHeX.Core**: 23.9.25+
- Switch connection requires sys-botbase 2.4+ (`BotbaseVersion` constant in [PokeRoutineExecutorBase.cs](SysBot.Pokemon/Actions/PokeRoutineExecutorBase.cs))

## Code Conventions

### Naming
- Private fields: `_camelCase` (e.g., `_raidMemoryManager`, `_settings`)
- Constants: `PascalCase` or `ALL_CAPS` for clarity (e.g., `MaxTeleportRetries`, `PULSE_UPDATE_INTERVAL_MS`)
- Static properties: `PascalCase` (e.g., `GameProgress`, `IsKitakami`)

### Error Handling
Log errors via `LogUtil.LogError()`. Critical failures set `HasErrored = true` flag. Recovery logic in `PerformRebootAndReset()`.

Teleportation failures tracked via `_consecutiveDenFailures` with automatic map refresh after `DenFailuresBeforeMapRefresh` threshold.

### Async Patterns
Always propagate `CancellationToken`. Use `ConfigureAwait(false)` for non-UI contexts:
```csharp
await _connection.ReadBytesAbsoluteAsync(offset, size, token).ConfigureAwait(false);
```

UI updates in WinForms use `this.Invoke()` for thread safety.

### Constants over Magic Numbers
Define thresholds at class level:
```csharp
private const int MaxTeleportRetries = 3;
private const float TeleportDistanceThreshold = 2.5f;
```

## Common Tasks

### Adding New Raid Types
1. Update `PokeRoutineType` enum in [PokeRoutineType.cs](SysBot.Pokemon/Actions/PokeRoutineType.cs)
2. Implement executor in `SysBot.Pokemon/SV/` inheriting `PokeRoutineExecutor9SV`
3. Register in [BotFactory9SV.cs](SysBot.Pokemon/SV/BotFactory9SV.cs) `CreateBot` switch

### Modifying Raid Settings
Settings hierarchy: `PokeTradeHubConfig` → `RotatingRaidSettingsSV` → nested categories. Add properties with attributes:
```csharp
[Category("RaidSettings"), Description("Enable mystery shiny raids")]
public bool MysteryRaids { get; set; }
```

### Discord Command Development
Add methods to [RaidModule.cs](SysBot.Pokemon.Discord/Commands/Bots/RaidModule.cs):
```csharp
[Command("mycommand")]
[Alias("mc")]
[Summary("Description for help text")]
public async Task MyCommandAsync(string param)
{
    var settings = Hub.Config.RotatingRaidSV;
    await ReplyAsync("Response").ConfigureAwait(false);
}
```

### Web API Extensions
HTTP server logic in [WebApi/BotServer.cs](SysBot.Pokemon.WinForms/WebApi/BotServer.cs). Uses simple HTTP listener on port 9090. Extend endpoints via route handlers.

## Integration Points

### PKHeX.Core
`PK9` objects represent Gen 9 Pokémon data. Use `AutoLegalityWrapper` for Showdown parsing:
```csharp
ShowdownSet set = new ShowdownSet(showdownText);
var pk = TrainerSettings.GetTrainerInfo<PK9>().GetLegal(set, out _);
```

### RaidCrawler Integration
Raid mechanics data structures from RaidCrawler.Core.Structures (e.g., `RaidContainer`, `TeraCrystalType`). Event data fetched via `HttpClient` calls to external raid databases.

### sys-botbase Protocol
Commands sent via `Connection.SendAsync()`. Common patterns:
- `Click(SwitchButton.A, delay, token)` - button presses
- `PointerPoke(bytes, pointerPath, token)` - memory writes via pointer chains
- `ReadBytesAbsoluteAsync(offset, size, token)` - direct memory reads

## Troubleshooting

### "Seed Mismatch" Errors
Increment `_seedMismatchCount` triggers re-injection. Check `OverrideSeedIndex()` method and verify memory pointer calculations in `RaidMemoryManager`.

### Teleportation Failures
Retry logic in `TeleportToInjectedDenLocation()` uses distance threshold validation. Ensure den locations JSON files have correct coordinates. Tracked via `_consecutiveDenFailures` with automatic map refresh after `DenFailuresBeforeMapRefresh` threshold.

### Discord Disconnects
Discord token in `DiscordSettings`. Connection managed by `SysCord<T>` startup. Check token permissions (Read Messages, Send Messages, Manage Reactions).

### Web UI Not Accessible
Requires admin rights or firewall rule. See README.md network setup instructions. Server starts in [Main.cs](SysBot.Pokemon.WinForms/Main.cs) `InitializeAsync()`.

**Firewall Setup**:
```cmd
netsh advfirewall firewall add rule name="SVRaidBot Web" dir=in action=allow protocol=TCP localport=9090
```

### Config Corruption Recovery
Program automatically attempts to restore from most recent `config.backup_*` file. Manual recovery: find latest backup in exe directory and rename to `config.json`.
