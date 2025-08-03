# Fixing .Net Build Issues in Windows / WSL Hybrid Environments

This guide fixes two related but distinct issues when developing .NET applications in a hybrid Windows/WSL environment. **Follow this sequence**: first fix the basic build permissions, then address the build cache state pollution.

> **Quick Check**: If you can already build .NET projects in WSL but encounter errors after building in Visual Studio, skip to the [Build Cache State Management](#build-cache-state-management) section.

## Problem 1: Cannot Build in WSL (Permission Error)

When building .NET projects in WSL on Windows filesystem mounts, you encounter:

```
System.ComponentModel.Win32Exception (1): Could not set file permission 755 for /mnt/c/Users/YourName/your-project/obj/Debug/net9.0/file1
```

### Root Cause
- WSL's .NET build process tries to set Linux executable permissions (chmod 755) on files
- Default WSL mount configuration doesn't support Linux metadata on Windows drives
- The Windows filesystem cannot handle Linux-style permission operations

First, determine which WSL configuration approach you need:

#### Step 1: Check Your Current Mount Type
```bash
mount | grep /mnt/c
```

- If you see `type drvfs`, try the **Legacy Configuration** first
- If you see `type 9p`, use the **Modern Configuration**

#### Legacy Configuration (Older WSL Versions)
**Note**: This may not be needed for current WSL2, but try it first if you see `type drvfs`.

1. **Create or edit `/etc/wsl.conf`**:
   ```bash
   sudo nano /etc/wsl.conf
   ```

2. **Add the following content**:
   ```ini
   [automount]
   options = "metadata"
   ```

3. **Restart WSL completely**:
   ```powershell
   wsl --shutdown
   wsl
   ```

4. **Test the build**:
   ```bash
   cd /mnt/c/Users/YourName/your-project
   dotnet build
   ```

#### Modern Configuration (Current WSL2)
If the legacy approach doesn't work or you see `type 9p`, use this approach:

1. **Configure WSL to use fstab**:
   ```bash
   sudo nano /etc/wsl.conf
   ```
   
   Contents should be:
   ```ini
   [boot]
   systemd=true
   
   [automount]
   mountFsTab = true
   ```

2. **Configure fstab for metadata support**:
   ```bash
   sudo nano /etc/fstab
   ```
   
   Add or modify the C: drive line:
   ```
   C:    /mnt/c    drvfs    defaults,metadata    0   0
   ```

3. **Restart WSL completely**:
   ```powershell
   wsl --shutdown
   wsl
   ```

4. **Verify the configuration**:
   ```bash
   mount | grep /mnt/c
   ```
   
   Should show `type drvfs` with `metadata` in the options.

5. **Test the build**:
   ```bash
   cd /mnt/c/Users/YourName/your-project
   dotnet build
   ```

#### Troubleshooting the Permission Fix
If the build still fails:

1. **Verify WSL processed your config**:
   ```bash
   mount | grep /mnt/c
   ```
   Look for `metadata` in the mount options.

2. **Check file ownership**:
   ```bash
   ls -l /etc/wsl.conf
   ```
   Should show `root root` ownership.

3. **Manual test** (temporary fix to verify the solution works):
   ```bash
   sudo umount /mnt/c
   sudo mount -t drvfs C: /mnt/c -o metadata
   dotnet build  # Should work now
   ```

---

## Problem 2: Build Cache State Management

**This issue only appears AFTER you've fixed Problem 1 above.** Once you can build in WSL, you may encounter this error when switching between Visual Studio and WSL:

```
NuGet.Packaging.Core.PackagingException: Unable to find fallback package folder 'C:\Program Files (x86)\Microsoft Visual Studio\Shared\NuGetPackages'
```

### Root Cause: State Pollution
- Visual Studio (Windows) creates `obj/project.assets.json` with Windows paths
- WSL's .NET tools cannot interpret Windows drive letter paths like `C:\Program Files...`
- This creates a "tug-of-war" where Windows and WSL write incompatible environment-specific state

### Solution: The "Good Hygiene" Workflow

#### The Rule
**Always run `dotnet restore` when switching from Visual Studio to WSL.**

#### Recommended Workflow

1. **Develop in Visual Studio** as normal
2. **Switch to WSL terminal** and navigate to your project
3. **Before any dotnet commands**, run:
   ```bash
   dotnet restore
   ```
4. **Now run your commands** without issues:
   ```bash
   dotnet clean
   dotnet build
   dotnet run
   ```

#### Why This Works
- `dotnet restore` regenerates `obj/project.assets.json` with Linux-compatible paths
- It's lightweight and only updates package resolution state (no compilation)
- This "resets" the project state for the Linux environment
- Fixes path pollution caused by Windows builds

### Alternative Advanced Solution (Not Recommended)

You could configure both Windows and WSL to use the same NuGet package directory using the `NUGET_PACKAGES` environment variable. However, this approach:

- Adds unnecessary complexity
- Can introduce performance penalties in WSL
- Requires careful environment variable management across both systems

The "Good Hygiene" workflow is preferred by most developers.

---

## Summary

For smooth hybrid Windows/WSL .NET development, follow this sequence:

1. **First**: Fix the basic permission issue so you can build in WSL at all
   - Check your mount type with `mount | grep /mnt/c`
   - Use legacy config for `drvfs`, modern config for `9p`
   - This is a one-time configuration

2. **Second**: Adopt the hygiene workflow for build cache management
   - Run `dotnet restore` when switching from Visual Studio to WSL
   - This prevents state pollution between environments
   - This is an ongoing workflow habit

**Remember**: These are two separate problems requiring different solutions. Permission issues are about filesystem metadata, while state issues are about build cache pollution between Windows and Linux environments.
