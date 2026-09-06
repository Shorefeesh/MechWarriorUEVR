# Installation

## Build

Use the VS2022 Build Tools installation directly. The VS2022 Community IDE does not contain the C++ workload, but MSVC 14.34 is installed at `C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools`.

Run the following commands from the workspace root in PowerShell. Initialize dependencies once:

```powershell
git -C UEVR submodule update --init --recursive
git -C MW5-UEVR-Plugins submodule update --init --recursive

if (-not (Test-Path MW5-UEVR-Plugins/ext/minhook)) {
    git clone --depth 1 https://github.com/TsudaKageyu/minhook.git MW5-UEVR-Plugins/ext/minhook
}
```

If the UESDK submodule's SSH URL fails, retry it over HTTPS without modifying `.gitmodules`:

```powershell
git -c submodule.dependencies/submodules/UESDK.url=https://github.com/praydog/UESDK.git -C UEVR submodule update --init --recursive
```

Build UEVR with Ninja and the VS2022 developer environment:

```powershell
cmd.exe /d /s /c 'call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\Tools\VsDevCmd.bat" -no_logo -arch=x64 && cmake -S UEVR -B UEVR/build -G Ninja -DCMAKE_BUILD_TYPE=Release'

# Ninja needs delayimp.lib because UEVR delay-loads openvr_api.dll.
cmd.exe /d /s /c 'call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\Tools\VsDevCmd.bat" -no_logo -arch=x64 && set LINK=delayimp.lib&& cmake --build UEVR/build --target uevr -j 4'
```

The UEVR output is `UEVR/build/bin/uevr/UEVRBackend.dll`.

Build HeadAim with its Release output in the location expected by the repository's post-build deployment command:

```powershell
cmd.exe /d /s /c 'call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\Tools\VsDevCmd.bat" -no_logo -arch=x64 && cmake -S MW5-UEVR-Plugins -B MW5-UEVR-Plugins/build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE="%CD%/MW5-UEVR-Plugins/build/Release"'
cmd.exe /d /s /c 'call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\Tools\VsDevCmd.bat" -no_logo -arch=x64 && cmake --build MW5-UEVR-Plugins/build --target HeadAim -j 4'
```

The HeadAim build's existing post-build command copies `MW5-UEVR-Plugins/build/Release/HeadAim.dll` into the live UEVR plugin directory. Treat that build as a deployment and obtain approval before running it when operating in a sandbox. To compile without deployment, disable the generated `POST_BUILD` command in the scratch build tree or build only the relevant object target.

When Git rejects repository ownership under a sandbox account, pass `-c safe.directory=<absolute repository path>` to that Git invocation. Generate UEVR's commit header with the same exception before the first build:

```powershell
$env:GIT_CONFIG_COUNT = '1'
$env:GIT_CONFIG_KEY_0 = 'safe.directory'
$env:GIT_CONFIG_VALUE_0 = (Resolve-Path UEVR).Path.Replace('\', '/')
Push-Location UEVR
.\MakeCommitHash.bat
Pop-Location
```

## Installation

### Create a separate patched UEVR copy

Run from E:\Creative\Coding\MechWarriorUEVR:

$testUEVR = 'E:\Creative\Coding\MechWarriorUEVR\artifacts\eye-look\UEVR-test'

Expand-Archive `
  -LiteralPath 'C:\Users\samon\Downloads\uevr.zip' `
  -DestinationPath $testUEVR `
  -Force

Copy-Item `
  -LiteralPath 'E:\Creative\Coding\MechWarriorUEVR\artifacts\eye-look\UEVRBackend.dll' `
  -Destination "$testUEVR\UEVRBackend.dll" `
  -Force

Launch this injector for the test:

Start-Process 'E:\Creative\Coding\MechWarriorUEVR\artifacts\eye-look\UEVR-test\UEVRInjector.exe'

### Back up and install HeadAim

Close MW5 and UEVR first, then run:

$plugins = 'C:\Users\samon\AppData\Roaming\UnrealVRMod\MechWarrior-Win64-Shipping\plugins'
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'

Copy-Item `
-LiteralPath "$plugins\HeadAim.dll" `
-Destination "$plugins\HeadAim.dll.$stamp.bak"

Copy-Item `
-LiteralPath 'E:\Creative\Coding\MechWarriorUEVR\artifacts\eye-look\HeadAim.dll' `
-Destination "$plugins\HeadAim.dll" `
-Force

Do not copy anything into temp_extraction.

### Launch and inject

1. Connect the Quest Pro through Quest Link.
2. Start MW5.
3. Start the patched UEVRInjector.exe.
4. Select MechWarrior-Win64-Shipping.exe.
5. Select OpenXR.
6. Inject.
7. Enter a mech and enable the existing head-aim mode.

For the clearest initial test, keep your head still and move only your eyes between cockpit features. The arm/weapon reticle should follow your gaze; ordinary cockpit viewing remains head-driven.

### Check the log

Run:

$log = 'C:\Users\samon\AppData\Roaming\UnrealVRMod\MechWarrior-Win64-Shipping\log.txt'

Select-String -LiteralPath $log -Pattern `
'XR_EXT_eye_gaze_interaction',
'Eye gaze interaction supported',
'Eye gaze action initialized',
'HeadAim'

Successful initialization should include:

Enabling XR_EXT_eye_gaze_interaction extension
Eye gaze interaction supported: true
Eye gaze action initialized
HeadAim 2.13.2 initialized

If gaze becomes unavailable, HeadAim automatically falls back to head aiming.

### Rollback

Close MW5 and UEVR, restore the backup:

$plugins = 'C:\Users\samon\AppData\Roaming\UnrealVRMod\MechWarrior-Win64-Shipping\plugins'

Get-ChildItem "$plugins\HeadAim.dll.*.bak" |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1 |
    Copy-Item -Destination "$plugins\HeadAim.dll" -Force

Then launch your original UEVR injector instead of the UEVR-test copy.
