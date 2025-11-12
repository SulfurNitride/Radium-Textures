# Radium Textures Development Log

## Session Overview
This document covers the complete development session where we fixed critical path handling bugs, rebranded the project, added multi-game support infrastructure, and set up GitHub CI/CD.

---

## Major Accomplishments

### 1. Fixed Critical Wine Path Handling Bug
**Problem:** GUI mode was failing with "Unknown option" errors because texconv.exe was receiving paths without leading `/`

**Error Example:**
```
ERROR: Unknown option: 'home/luke/Games/Tuxborn/mods/VRAMr Output/textures/...'
```

**Root Cause:**
- Wine was stripping the leading `/` from Linux absolute paths when they contained spaces
- The space in "VRAMr Output" triggered this behavior
- Debug logging showed Rust was sending correct paths, but Wine mangled them before passing to texconv.exe

**Failed Attempts:**
1. Removed `linux_to_wine_path()` function and passed Linux absolute paths directly
2. Added extensive debug logging to trace the issue
3. Discovered Wine was the culprit, not our code

**Final Solution:**
Convert all paths to Wine Z: drive format, which properly handles spaces:
```rust
// Convert Linux paths to Wine Z: drive format to handle spaces correctly
// Z: maps to the Linux root /
let wine_texconv = format!("Z:{}", texconv_path.display());
let wine_output_dir = format!("Z:{}", output_dir.display());
let wine_input_path = format!("Z:{}", full_path.display());

// Build texconv command
let mut cmd = Command::new("wine");
cmd.arg(&wine_texconv);
cmd.arg("-gpu").arg("0");
cmd.arg("-sepalpha");
cmd.arg("-nologo");
cmd.arg("-y");
cmd.arg("-o").arg(&wine_output_dir);
cmd.arg("--single-proc");
cmd.arg(&wine_input_path);
```

**Files Modified:**
- `src/optimization.rs` (lines 221-259)

**Result:** User confirmed: "ok its working now."

---

### 2. Rebranded from VRAMr to Radium Textures

**Changes Made:**

#### Package Configuration (`Cargo.toml`)
```toml
[package]
name = "radium-textures"  # Was: vramr-linux
version = "1.0.0"         # Was: 15.0.0
edition = "2021"
authors = ["Radium Textures"]
description = "Skyrim texture optimization tool for Linux with MO2 support"
```

#### Binary Output
- Old: `target/release/vramr-linux`
- New: `target/release/radium-textures`

#### Config Directory
- Old: `~/.config/vramr-linux/`
- New: `~/.config/radium-textures/`

#### GUI Updates
- Window title: "Radium Textures"
- All UI strings updated
- File: `src/gui/mod.rs`

#### Documentation
- Updated `README.md` with new branding
- Updated all code comments

---

### 3. Added Multi-Game Support Infrastructure

**Created New File:** `src/game.rs`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Game {
    SkyrimSE,
    #[allow(dead_code)]  // Coming later
    Fallout4,
}

impl Game {
    pub fn display_name(&self) -> &'static str {
        match self {
            Game::SkyrimSE => "Skyrim SE",
            Game::Fallout4 => "Fallout 4",
        }
    }

    pub fn exclusions_file(&self) -> &'static str {
        match self {
            Game::SkyrimSE => "SkyrimSE_Exclusions.txt",
            Game::Fallout4 => "Fallout4_Exclusions.txt",
        }
    }
}
```

**Exclusions System:**
- Moved: `Exclusions.mod` → `exclusions/SkyrimSE_Exclusions.txt`
- Created: `exclusions/Fallout4_Exclusions.txt` (placeholder for future)

**GUI Integration:**
- Added `game: Game` field to `AppSettings` struct
- Simple display: "Game: Skyrim SE" (toggle UI removed per user feedback)
- Ready for future Fallout 4 support

**User Feedback:**
> "these icons kinda suck tbh :skull: also we dont need fo4 right now will be added later"

Decision: Simplified to text label, kept infrastructure ready.

---

### 4. GitHub Repository Setup

**Repository:** https://github.com/SulfurNitride/Radium-Textures

**Git Configuration:**
```bash
git config user.name "SulfurNitride"
git config user.email "lukew19@proton.me"
```

**Initial Commit Structure:**
```
d2c3cb8 Initial release of Radium Textures v1.0.0
b9ce01f Update README.md
dcabd47 Add GitHub Actions workflows for CI/CD
fe90020 Fix build workflow to include texconv.exe and exclusions
b20a889 Launch GUI by default when double-clicked
```

---

### 5. GitHub Actions CI/CD

**Created Two Workflows:**

#### Build Workflow (`.github/workflows/build.yml`)
- Triggers: Push/PR to master or main branches
- Runs on: ubuntu-latest
- Steps:
  1. Checkout code
  2. Install Rust toolchain (stable, with rustfmt and clippy)
  3. Cache cargo registry, index, and build artifacts
  4. Check formatting (`cargo fmt`)
  5. Run clippy (`cargo clippy`)
  6. Build debug (`cargo build`)
  7. Run tests (`cargo test`)
  8. Build release (`cargo build --release`)
  9. Bundle artifacts (binary + texconv.exe + exclusions + README)
  10. Upload artifacts (7-day retention)

**Artifact Bundle Contents:**
```bash
mkdir -p artifact
cp target/release/radium-textures artifact/
cp texconv.exe artifact/
cp -r exclusions artifact/
cp README.md artifact/
```

**Bug Fix:** Initially only uploaded binary, user noticed texconv.exe was missing. Fixed to bundle all dependencies.

#### Release Workflow (`.github/workflows/release.yml`)
- Triggers: Git tags matching `v*` pattern
- Runs on: ubuntu-latest
- Creates: GitHub releases with tar.gz archives
- Filename: `radium-textures-{tag}-linux-x64.tar.gz`
- Auto-generates release notes

---

### 6. Made GUI Default Behavior

**Problem:** Double-clicking binary showed CLI help instead of launching GUI

**User Feedback:**
> "i tried opening it and its just the cli? You should make it so if they double click it, it will open up the gui"

**Solution:** Made CLI command optional in `src/main.rs`

**Before:**
```rust
struct Cli {
    #[command(subcommand)]
    command: Commands,  // Required
}
```

**After:**
```rust
struct Cli {
    #[command(subcommand)]
    command: Option<Commands>,  // Optional
}

match cli.command {
    // Default to GUI when no command is provided
    None | Some(Commands::Gui) => {
        return gui::run().map_err(|e| anyhow::anyhow!("GUI error: {}", e));
    }
    Some(Commands::Analyze { profile, mods, data }) => {
        analyze_profile(profile, mods, data)?;
    }
    // ... other commands ...
}
```

**Result:** Double-clicking now launches GUI automatically, CLI still available via `./radium-textures <command>`

---

## Additional Bug Fixes

### Logger Initialization Error
**Problem:** "Builder::init should not be called after logger initialized: SetLoggerError(())"

**Cause:** Logger initialized in both `main.rs` and `gui/mod.rs`

**Fix:** Removed duplicate initialization in `gui/mod.rs`, kept only in `main.rs`

**Enhancement:** Made logger respect `RUST_LOG` environment variable:
```rust
let log_level = std::env::var("RUST_LOG")
    .ok()
    .and_then(|s| s.parse().ok())
    .unwrap_or(LevelFilter::Info);

env_logger::Builder::new()
    .filter_level(log_level)
    .init();
```

### Debug Logs Not Appearing
**Problem:** `RUST_LOG=debug` wasn't showing debug logs in GUI mode

**Fix:** Updated main.rs to properly check for RUST_LOG environment variable before defaulting to Info level

---

## Technical Architecture

### Core Components

1. **VFS Builder** (`src/vfs.rs`)
   - Parses MO2 profiles and modlist.txt
   - Resolves file conflicts based on mod priority
   - Builds virtual file system matching game load order

2. **BSA Parser** (`src/bsa.rs`)
   - Reads Bethesda archive files (BSA/BA2)
   - Extracts textures for processing
   - Handles compression and file records

3. **Texture Analyzer** (`src/analyzer.rs`)
   - Scans textures and determines types (Diffuse, Normal, Specular, etc.)
   - Reads DDS headers for format and dimensions
   - Applies exclusion patterns

4. **Optimizer** (`src/optimization.rs`)
   - Groups textures by processing type
   - Parallel processing with Rayon (configurable thread count)
   - Executes texconv.exe via Wine for format conversion and resizing
   - Smart batching: BC7, BC4, RGBA, PBR, Specular, Emissive, Gloss

5. **Database** (`src/database.rs`)
   - SQLite for texture metadata
   - Tracks processing history
   - Supports incremental updates

6. **GUI** (`src/gui/mod.rs`, `src/gui/worker.rs`)
   - egui-based interface
   - Background worker thread for long-running operations
   - Preset system (Ultra, High, Medium, Low)
   - Progress tracking and logging

7. **Game System** (`src/game.rs`)
   - Multi-game support framework
   - Game-specific exclusions
   - Extensible for future games

### Processing Pipeline

```
1. Parse MO2 Profile → Build VFS
2. Scan BSA/BA2 archives + loose files
3. Analyze textures → Determine types and dimensions
4. Apply exclusions (game-specific patterns)
5. Group by processing type (BC7, BC4, RGBA, etc.)
6. Parallel optimization with texconv
7. Pack into optimized BSA
```

---

## File Structure

```
radium-textures/
├── .github/
│   └── workflows/
│       ├── build.yml          # CI builds on every push/PR
│       └── release.yml        # Release automation for tags
├── exclusions/
│   ├── SkyrimSE_Exclusions.txt    # Skyrim texture exclusions
│   └── Fallout4_Exclusions.txt   # Fallout 4 (placeholder)
├── src/
│   ├── main.rs                # Entry point, CLI parsing
│   ├── vfs.rs                 # MO2 VFS builder
│   ├── bsa.rs                 # BSA/BA2 parser
│   ├── analyzer.rs            # Texture analysis
│   ├── optimization.rs        # texconv processing
│   ├── database.rs            # SQLite metadata
│   ├── game.rs                # Multi-game support
│   └── gui/
│       ├── mod.rs             # GUI interface
│       └── worker.rs          # Background worker
├── texconv.exe                # DirectXTex conversion tool
├── Cargo.toml                 # Package configuration
├── README.md                  # User documentation
└── .gitignore                 # Excludes build artifacts, session notes
```

---

## Usage Examples

### GUI Mode (Default)
```bash
./radium-textures
# or explicitly:
./radium-textures gui
```

### CLI Mode
```bash
# Analyze and optimize
./radium-textures analyze \
  --profile /path/to/MO2/profiles/YourProfile \
  --mods /path/to/MO2/mods \
  --data /path/to/Skyrim/Data

# With verbose logging
RUST_LOG=debug ./radium-textures analyze --profile ... --mods ... --data ...
```

### Build from Source
```bash
cargo build --release
# Binary: target/release/radium-textures
```

---

## Key Technical Decisions

### Wine Z: Drive Format
**Why:** Wine's Z: drive mapping to Linux root `/` properly handles paths with spaces, preventing path mangling when passing arguments to Windows executables.

**Implementation:** All paths converted via `format!("Z:{}", path.display())`

### Rayon Parallel Processing
**Why:** Maximize throughput by running multiple texconv instances in parallel (default: CPU core count)

**Configuration:** Texconv runs with `--single-proc`, Rayon manages parallelization at batch level

### Game-Specific Exclusions
**Why:** Different games have different texture requirements and problem assets

**Design:** Pattern-based exclusion files in `exclusions/` directory, loaded based on selected game

### Optional CLI Command
**Why:** User-friendly default behavior (GUI) while maintaining full CLI functionality

**Trade-off:** Slightly unconventional for CLI apps, but better UX for desktop use

---

## Dependencies

### Runtime Requirements
- Wine (for texconv.exe)
- texconv.exe (included in repository)

### Build Requirements
- Rust 1.70+
- Cargo

### Key Rust Crates
- `clap` - CLI argument parsing
- `egui` / `eframe` - GUI framework
- `rayon` - Parallel processing
- `rusqlite` - SQLite database
- `anyhow` - Error handling
- `log` / `env_logger` - Logging
- `serde` / `serde_json` - Serialization

---

## Future Enhancements

### Planned Features
1. **Fallout 4 Support** - Infrastructure ready, needs testing and exclusions
2. **Proton Detection** - Auto-detect and use Proton's Wine when available
3. **GUI Game Selector** - Toggle between Skyrim SE and Fallout 4
4. **Progress Persistence** - Resume interrupted optimizations
5. **Diff Mode** - Compare before/after texture sizes

### Potential Improvements
- GPU acceleration for format conversion (may require native Linux texconv alternative)
- Multi-game batch processing
- Texture quality presets per-texture-type
- Integration with Vortex mod manager

---

## Known Issues / Limitations

1. **Wine Dependency** - Requires Wine to run texconv.exe (Windows-only tool)
2. **Linux Only** - Designed specifically for Linux with MO2 via Wine
3. **Skyrim SE Focus** - Fallout 4 support planned but not yet implemented
4. **Manual Output Path** - User must specify where optimized BSA should go

---

## Testing Notes

### Verified Configurations
- **OS:** Linux (tested on CachyOS 6.17.7-3)
- **MO2:** Mod Organizer 2 via Wine
- **Wine:** Standard Wine installation
- **Paths:** Handles spaces correctly via Z: drive format

### Test Cases Covered
1. ✅ Paths with spaces (e.g., "VRAMr Output")
2. ✅ Large texture sets (parallel processing)
3. ✅ MO2 profile parsing and conflict resolution
4. ✅ Multiple texture types (BC7, BC4, RGBA, etc.)
5. ✅ GUI preset system
6. ✅ CLI mode with verbose logging
7. ✅ GitHub Actions build artifacts

---

## Git Commit History

```
b20a889 - Launch GUI by default when double-clicked
fe90020 - Fix build workflow to include texconv.exe and exclusions
dcabd47 - Add GitHub Actions workflows for CI/CD
b9ce01f - Update README.md
d2c3cb8 - Initial release of Radium Textures v1.0.0
```

---

## Credits

- **Original Project:** VRAMr by gavwhittaker
- **Port/Rewrite:** Radium Textures (Rust implementation)
- **texconv:** Microsoft DirectXTex library
- **Developer:** SulfurNitride (lukew19@proton.me)

---

## License

MIT License

---

## Session Timeline

1. **Initial Problem:** texconv path errors in GUI mode
2. **Debug Phase:** Added logging, traced Wine path mangling
3. **Fix Applied:** Wine Z: drive format for all paths
4. **Rebranding:** VRAMr → Radium Textures
5. **Multi-Game:** Added game selection infrastructure
6. **GitHub Setup:** Repository creation, git configuration
7. **CI/CD:** GitHub Actions workflows
8. **Final Touch:** GUI as default behavior

**Total Session Time:** ~2-3 hours
**Files Changed:** 10+ files across codebase
**Commits:** 5 major commits
**Status:** Production ready, v1.0.0 released

---

## Quick Reference Commands

```bash
# Build release binary
cargo build --release

# Run with debug logging
RUST_LOG=debug ./radium-textures gui

# CLI mode
./radium-textures analyze --profile PATH --mods PATH --data PATH

# Check git status
git status

# Push changes
git add . && git commit -m "message" && git push

# Create release tag
git tag v1.0.1
git push origin v1.0.1
```

---

## Contact & Support

- **GitHub:** https://github.com/SulfurNitride/Radium-Textures
- **Email:** lukew19@proton.me
- **Issues:** https://github.com/SulfurNitride/Radium-Textures/issues

---

*Last Updated: 2025-11-12*
*Version: 1.0.0*
