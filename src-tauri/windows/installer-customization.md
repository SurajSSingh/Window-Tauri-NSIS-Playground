# NSIS Installer Customization Guide

This folder contains `installer.nsi`, the custom NSIS template used by Tauri when building the Windows installer. Most visual changes do not require learning NSIS. Start by changing values in `src-tauri/tauri.conf.json`, then only edit `installer.nsi` when the Tauri options do not cover the change you want.

## Minimal Customization Path

1. Put your installer assets in `src-tauri/windows/` or `src-tauri/icons/`.
2. Reference those assets from `src-tauri/tauri.conf.json` under `bundle.windows.nsis`.
3. Rebuild the Windows installer.

Example:

```json
{
  "bundle": {
    "windows": {
      "nsis": {
        "template": "windows/installer.nsi",
        "installerIcon": "icons/icon.ico",
        "sidebarImage": "windows/sidebar.bmp",
        "headerImage": "windows/header.bmp"
      }
    }
  }
}
```

Paths are relative to `src-tauri/`. Keep `template: "windows/installer.nsi"` so Tauri continues to use this script.

## Cross-Compile NSIS From macOS

Tauri can cross-compile Windows NSIS installers from macOS, but MSI installers cannot be created off Windows. Keep `bundle.targets` set to `["nsis"]` in `src-tauri/tauri.conf.json`; using `"all"` includes MSI and will break this workflow.

One-time setup on macOS:

```sh
brew install nsis llvm
rustup target add x86_64-pc-windows-msvc
cargo install --locked cargo-xwin
```

Ensure Homebrew LLVM is on `PATH`; on Apple Silicon this is usually `/opt/homebrew/opt/llvm/bin`.

Build the Windows NSIS installer:

```sh
cargo tauri build --runner cargo-xwin --target x86_64-pc-windows-msvc
```

The installer is written to `src-tauri/target/x86_64-pc-windows-msvc/release/bundle/nsis/` when running from `src-tauri/`, or `target/x86_64-pc-windows-msvc/release/bundle/nsis/` relative to the Tauri project root.

Use this for fast unsigned test builds. Signing cross-compiled Windows installers requires an external signing tool.

## Change The Background Image

The easiest background-style change is the left-side image shown on the welcome and finish pages.

1. Create a Windows bitmap file, for example `src-tauri/windows/sidebar.bmp`.
2. Add this to `bundle.windows.nsis` in `tauri.conf.json`:

```json
"sidebarImage": "windows/sidebar.bmp"
```

The script already connects that setting here:

```nsis
!define SIDEBARIMAGE "{{sidebar_image}}"

!if "${SIDEBARIMAGE}" != ""
  !define MUI_WELCOMEFINISHPAGE_BITMAP "${SIDEBARIMAGE}"
!endif
```

Recommended image format:

- Use `.bmp` for the most reliable NSIS compatibility.
- A common Modern UI sidebar size is `164x314` pixels.
- If the image appears stretched or cropped, resize the bitmap instead of changing NSIS layout code.

## Change The Header Image

The header image appears at the top of installer pages after the welcome page.

1. Create `src-tauri/windows/header.bmp`.
2. Add this to `bundle.windows.nsis`:

```json
"headerImage": "windows/header.bmp"
```

The script already enables header images when `headerImage` is provided:

```nsis
!if "${HEADERIMAGE}" != ""
  !define MUI_HEADERIMAGE
  !define MUI_HEADERIMAGE_BITMAP "${HEADERIMAGE}"
!endif
```

Recommended image format:

- Use `.bmp`.
- A common Modern UI header size is `150x57` pixels.
- Keep important text or logos away from the edges.

## Change The Installer Icon

Use an `.ico` file that contains multiple sizes, such as `16x16`, `32x32`, `48x48`, and `256x256`.

Add this to `bundle.windows.nsis`:

```json
"installerIcon": "icons/icon.ico"
```

The script maps that value to NSIS with:

```nsis
!if "${INSTALLERICON}" != ""
  !define MUI_ICON "${INSTALLERICON}"
!endif
```

## Change The Installer Progress Bar

The install progress page is created by this line in `installer.nsi`:

```nsis
!insertmacro MUI_PAGE_INSTFILES
```

For simple visual changes, add Modern UI settings before that line. For example:

```nsis
; 7. Installation page
!define MUI_INSTFILESPAGE_PROGRESSBAR smooth
!insertmacro MUI_PAGE_INSTFILES
```

Some NSIS themes also support custom page colors:

```nsis
; 7. Installation page
!define MUI_INSTFILESPAGE_COLORS "FFFFFF 000000"
!define MUI_INSTFILESPAGE_PROGRESSBAR smooth
!insertmacro MUI_PAGE_INSTFILES
```

Notes:

- Put these `!define` lines before `!insertmacro MUI_PAGE_INSTFILES`.
- Windows visual styles can override some progress bar colors.
- If the color does not change, keep the default Windows progress bar and customize the sidebar/header images instead.

## Change Button And Page Text

Prefer changing Tauri metadata first:

- `productName` changes the app name shown by the installer.
- `version` changes the displayed installer version.
- `bundle.publisher` or related bundle metadata can change publisher/manufacturer text when configured.

For text controlled by NSIS language strings, search `installer.nsi` for the text or string key you want to change. Keep the page order the same unless you are comfortable with NSIS page macros.

## Safe Areas To Edit

These parts are designed for simple customization:

- The Tauri `bundle.windows.nsis` settings in `tauri.conf.json`.
- The image and icon definitions near the top of `installer.nsi`.
- Modern UI `!define` lines placed immediately before a page macro, such as `MUI_PAGE_INSTFILES`.

Avoid changing these unless you are debugging installer behavior:

- Registry keys.
- Reinstall and uninstall detection logic.
- WebView2 installation logic.
- File copy and shortcut creation sections.

## Quick Checklist

Before rebuilding, confirm:

- Asset paths are relative to `src-tauri/`.
- Bitmap files are `.bmp`.
- Icon files are `.ico`.
- `template` still points to `windows/installer.nsi`.
- Any NSIS `!define` customization appears before the page macro that uses it.
