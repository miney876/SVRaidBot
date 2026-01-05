# SVRaidBot Development Guide

## Project Overview
SVRaidBot is a .NET-based automation system for hosting Pokémon Scarlet/Violet (SV) raids via Nintendo Switch. It connects to the Switch through sys-botbase/usb-botbase and provides multiple user interfaces (WinForms, Discord, Twitch, Console). The bot automates raid hosting, user requests, seed injection, and complex raid mechanics.

## Architecture

### Core Projects
- **SysBot.Base**: Low-level Switch communication via sys-botbase protocol, bot lifecycle management (`BotRunner`, `RoutineExecutor`), connection handling (WiFi/USB)
- **SysBot.Pokemon**: Core Pokémon logic, raid mechanics, PKHeX integration, `PokeRoutineExecutor` base classes
- **SysBot.Pokemon.WinForms**: Primary GUI with built-in web API (port 9090) for remote control
- **SysBot.Pokemon.Discord**: Discord bot integration with reaction-based code distribution
- **SysBot.Pokemon.Twitch**: Twitch chat bot integration
- **SysBot.Pokemon.ConsoleApp**: Headless console runner

### Key Components
- **RotatingRaidBotSV** (`SysBot.Pokemon/SV/BotRaid/`): Main raid automation logic (~5000+ lines), handles raid rotation, seed injection, teleportation, story progress manipulation
- **PokeRaidHub**: Centralized configuration and bot coordination with `ConcurrentPool<PokeRoutineExecutorBase>` for multi-bot orchestration
- **RaidMemoryManager**: Direct Switch memory manipulation for raid seeds, event detection, game state
- **UserRequestManager**: User raid request queue and priority system

## Critical Switch Communication

### Botbase Protocol
- Requires **sys-botbase** (WiFi) or **usb-botbase** (USB) on the Switch with minimum version checks
- Commands sent via `SwitchCommand` in `SysBot.Base/Util/SwitchCommand.cs`
- Memory reading/writing uses pointer dereferencing with `ValidatePointerAll()` pattern
- Connection configured via `IConsoleBotConfig` with `SwitchProtocol` enum (WiFi/USB)

### Memory Offsets
- Raid block pointers stored per-map: `_raidBlockPointerP` (Paldea), `_raidBlockPointerK` (Kitakami), `_raidBlockPointerB` (Blueberry)
- Offsets defined in `SysBot.Pokemon/SV/BotRaid/Blocks.cs` and den location JSONs in `DenLocations/`
- Direct memory writes for seed injection, story progress flags, spawn toggles

## Development Workflows

### Building
- **Visual Studio 2022** or **MSBuild CLI**: `msbuild SysBot.NET.sln /p:Configuration=Release /p:Platform=x64`
- Target framework: **.NET 7.0** (legacy) upgrading to **.NET 9.0** (see `Directory.Build.props`)
- Primary build output: `SysBot.Pokemon.WinForms` single-file executable
- Azure Pipelines CI/CD configured in `azure-pipelines.yml` for automated releases

### Running & Debugging
- **WinForms**: Run as admin for web API network access (port 9090)
- **Config**: `config.json` auto-created in working directory, uses Source Generation serialization (`ProgramConfigContext`)
- **Logging**: NLog configured in `SysBot.Base/Util/NLog.config`, controlled by `LoggingEnabled` and `MaxArchiveFiles` settings
- **Testing**: xUnit tests in `SysBot.Tests/` using FluentAssertions for queue, command, and pool logic

### Configuration Management
- All settings in `PokeRaidHubConfig` → `RotatingRaidSettingsSV`
- Settings use `[Category]` and `[Description]` attributes for PropertyGrid binding in WinForms
- Config auto-saves every 5 minutes with backup rotation (`*.backup_*`)
- Raid list loaded from `raidfilessv/raidsv.txt` format: `<seed>-<species>-<stars>-<storyprogress>`

## Project-Specific Conventions

### Routine Executors
- All bot logic inherits from `PokeRoutineExecutor<T>` where `T : PKM` (PK9 for SV)
- Generation-specific: `PokeRoutineExecutor9SV` for Scarlet/Violet
- Main execution in `ExecuteAsync()` with `CancellationToken` support
- Use `Connection.SendAsync()` for all Switch I/O, never blocking calls

### PKHeX Integration
- **PKHeX.Core.dll** and **PKHeX.Core.AutoMod.dll** in `SysBot.Pokemon/deps/` (not NuGet packages)
- Types: `PK9` for Gen 9 Pokémon, `SAV9SV` for save data
- Showdown format parsing via `ShowdownUtil.cs` for `PartyPK` settings
- **RaidCrawler.Core** for raid seed calculation and validation

### Discord Bot Patterns
- Commands in `SysBot.Pokemon.Discord/Commands/` using Discord.Net command framework
- `[RequireRoleAccess]` and `[RequireSudo]` attributes for permission control
- Reaction-based code distribution: users react to raid embeds to receive DM with codes (anti-scraping measure)
- Embed customization via `RaidEmbedInfoHelpers.cs` and localization in `Language/EmbedLanguageMappings.json`

### Raid Request System
- Format: `ra <seed> <difficulty> <storyprogress>` command adds to queue
- Seed finder integration: https://genpkm.com/raids/seeds/
- Request limiting per-user configurable, queue managed by `UserRequestManager`
- Priority: User requests > Random rotation > Mystery raids

## Anti-Theft Measures
- GIF-based raid code display to prevent OCR scraping (see `raidthieves.md` for backstory)
- Reaction-gated code distribution for public raids
- Watermarking and obfuscation techniques in raid embeds

## Common Pitfalls
- **Tesla/dmnt interference**: `CheckForRAMShiftingApps()` detects RAM-shifting overlays that break memory reads
- **Story progress sync**: `AutoStoryProgress` must match raid difficulty (4★ unlocked vs 6★ unlocked) or game state won't match
- **Time rollback**: `EnableTimeRollback` prevents date change breaking `TodaySeed` tracking
- **Teleportation**: `Auto Teleport` feature handles lost raid dens by finding nearest matching den using embedded JSON coordinates

## Key Files for Understanding
- [SysBot.Pokemon/SV/BotRaid/RotatingRaidBotSV.cs](SysBot.Pokemon/SV/BotRaid/RotatingRaidBotSV.cs) - Main bot loop and raid logic
- [SysBot.Base/Control/BotRunner.cs](SysBot.Base/Control/BotRunner.cs) - Bot lifecycle management
- [SysBot.Pokemon/RaidHub/PokeRaidHub.cs](SysBot.Pokemon/RaidHub/PokeRaidHub.cs) - Configuration and coordination
- [README.md](README.md) - User-facing feature documentation
- [Directory.Build.props](Directory.Build.props) - Shared build properties

## External Dependencies
- **Switch Firmware**: sys-botbase (https://github.com/olliz0r/sys-botbase) or usb-botbase
- **Discord API**: Discord.Net 3.15.3
- **PKHeX Libraries**: Local DLL references (not package managed)
- **Web API**: Embedded HTTP server on port 9090 for mobile device access

## Versioning
Current version tracked in `SysBot.Pokemon/SV/BotRaid/Helpers/SVRaidBot.cs` as `Version` constant. Update on releases.
