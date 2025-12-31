# Compiling TSWoW on Windows ARM

Explicitly tell CMake to use the **x64 (x86_64) platform architecture** for all builds. This allows Windows ARM to use x86_64 emulation, which works seamlessly with the pre-built x86_64 dependencies.

Set the `CMAKE_GENERATOR_PLATFORM` environment variable to `x64` in your Windows environment:

**PowerShell (Permanent - User Level):**
```powershell
[System.Environment]::SetEnvironmentVariable('CMAKE_GENERATOR_PLATFORM', 'x64', 'User')
```

**PowerShell (Permanent - System Level - Requires Admin):**
```powershell
[System.Environment]::SetEnvironmentVariable('CMAKE_GENERATOR_PLATFORM', 'x64', 'Machine')
```

**Note:** After setting a permanent environment variable, you may need to restart your terminal or PowerShell session for it to take effect.

The following components will use x64 architecture when the environment variable is set:

- **TrinityCore** - Server core (x64)
- **MPQBuilder** - Build tool (x64)
- **ADTCreator** - Build tool (x64)
- **BLPConverter** - Build tool (x64)
- **ClientExtensions** - Client DLL (remains Win32/32-bit, as required by WoW client)

### Verifying the Environment Variable

To verify the environment variable is set correctly:

**PowerShell:**
```powershell
$env:CMAKE_GENERATOR_PLATFORM
# Should output: x64
```

## Build Process

1. **Set the environment variable** (see above)

2. **Clean any existing build artifacts** that may have been created with the wrong architecture:
   ```powershell
   Remove-Item -Recurse -Force C:\dev\tswow-build\TrinityCore -ErrorAction SilentlyContinue
   Remove-Item -Recurse -Force C:\dev\tswow-build\mpqbuilder -ErrorAction SilentlyContinue
   Remove-Item -Recurse -Force C:\dev\tswow-build\adtcreator -ErrorAction SilentlyContinue
   Remove-Item -Recurse -Force C:\dev\tswow-build\blpconverter -ErrorAction SilentlyContinue
   ```

3. **Build TSWoW:**
   ```powershell
   npm run build
   ```

## How It Works

When `CMAKE_GENERATOR_PLATFORM` is set to `x64`, CMake will:

1. Use the Visual Studio x64 toolchain (`Hostarm64/x64/cl.exe`) instead of ARM64
2. Look for x64 Boost libraries (which match the pre-built dependencies)
3. Generate x64 build configurations consistently across all components

Windows ARM's x86_64 emulation handles the rest transparently, allowing the x64 binaries to run on ARM hardware.