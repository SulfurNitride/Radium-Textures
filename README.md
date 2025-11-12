# Radium Textures

<img width="1825" height="1645" alt="image" src="https://github.com/user-attachments/assets/2f2d23cb-ec47-4cfb-a367-d58f49218267" />


**Skyrim texture optimization for Linux with MO2 support. Fallout 4 support coming soon!**

## Requirements

### Build Requirements
- Rust 1.70+
- Cargo

### Runtime Requirements
- Wine (for texconv.exe)
  - *Note: Proton detection coming in a future update!*
- texconv.exe (included in repository)

Port of VRAMr by gavwhittaker, rewritten in Rust.

## Building

```bash
cargo build --release
```

The release binary will be in `target/release/radium-textures`.

## Usage

### GUI Mode

```bash
chmod +x ./radium-textures
```
Either by double clicking it or:
```bash
./radium-textures gui
```

### CLI Mode
```bash
# Analyze MO2 profile and build VFS
./radium-textures analyze \
  --profile /path/to/MO2/profiles/YourProfile \
  --mods /path/to/MO2/mods \
  --data /path/to/Skyrim/Data

# With verbose logging
./radium-textures -v analyze --profile ... --mods ... --data ...
```

## Credits

- **Port Of:** VRAMr by gavwhittaker
- **texconv:** Microsoft DirectXTex library

## License

MIT License


