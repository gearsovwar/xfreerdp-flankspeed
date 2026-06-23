# xfreerdp-flankspeed

`xfreerdp-flankspeed` is a Linux launcher for connecting to Azure Virtual Desktop / Flank Speed-style remote desktops with CAC smartcard redirection using a patched FreeRDP build.

It uses Chromium for CAC/Microsoft authentication, captures the current AVD gateway bearer token from the browser session, launches patched `xfreerdp`, completes the AAD redirect flow automatically, closes Chromium to release the CAC, and then connects with smartcard redirection enabled.

## Status

Working locally with:

- Linux host
- CAC authentication through Chromium
- Azure Virtual Desktop gateway token capture
- Patched FreeRDP AAD login
- CAC/smartcard redirection into the remote Windows desktop
- One-command launcher flow

## What this project does

The launcher automates this flow:

1. Opens the AVD web client in Chromium.
2. User signs in normally with CAC.
3. Watches for the AVD feed discovery request.
4. Extracts the current `Authorization: Bearer ...` gateway token from that browser request.
5. Starts patched `xfreerdp` with `/gateway:type:arm,bearer:<token>`.
6. Waits for FreeRDP to print the AAD login URL.
7. Opens that URL in Chromium.
8. Captures the final `aadj.html?code=...` redirect.
9. Closes Chromium so the CAC is released.
10. Feeds the final redirect URL back into FreeRDP.
11. FreeRDP connects and redirects the CAC/smartcard into the remote desktop.

## What this project does not do

This does not bypass authentication.

You still need:

- A valid CAC
- A working smartcard reader
- Access to the assigned Azure Virtual Desktop resource
- A valid `.rdpw` file from the environment
- A patched FreeRDP build

This repository does not include `.rdpw` files, bearer tokens, refresh tokens, browser profiles, certificates, or user-specific government data.

## Why FreeRDP had to be patched

The target environment required a Gov AVD/AAD flow plus a browser-style SPA token redemption flow. Stock FreeRDP did not produce the exact flow needed.

The patch in this repository modifies FreeRDP in several areas.

### 1. Government cloud endpoints

FreeRDP was using commercial Microsoft AVD/AAD defaults in some places. The patch changes the relevant defaults to Gov cloud values, including:

    login.microsoftonline.us
    https://www.wvd.azure.us/.default

The RDS device service scope remains based on:

    ms-device-service://termsrv.wvd.microsoft.com/...

because that is the resource principal accepted for the second-hop RDS AAD token flow.

### 2. Web client redirect URI

The working redirect URI is:

    https://rdweb.wvd.azure.us/arm/webclient/aadj.html

The launcher passes it to FreeRDP with:

    /azure:avd-access:'https%%3A%%2F%%2Frdweb.wvd.azure.us%%2Farm%%2Fwebclient%%2Faadj.html'

The doubled percent signs are intentional because FreeRDP treats this setting as a format string internally.

### 3. PKCE support

The web client redirect flow requires PKCE. The patch adds support for:

    code_challenge
    code_challenge_method=S256
    code_verifier

These are added to the AAD authorization URL and token redemption POST.

### 4. SPA-style token redemption headers

After PKCE was added, Azure AD rejected the token request with a SPA/cross-origin error.

The patch adds browser-style headers to Microsoft authorization-code token POSTs:

    Origin: https://rdweb.wvd.azure.us
    Referer: https://rdweb.wvd.azure.us/arm/webclient/aadj.html

These headers are only added for Microsoft token POSTs using:

    grant_type=authorization_code

## Repository contents

    xfreerdp-flankspeed/
    ├── README.md
    ├── xfreerdp-flankspeed
    ├── patches/
    │   └── freerdp-avd-gov-spa-pkce.patch
    ├── examples/
    │   └── config.example.env
    └── .gitignore

## Requirements

- Linux
- Python 3
- Playwright
- Chromium installed by Playwright
- PC/SC smartcard stack
- OpenSC / CCID support
- Patched FreeRDP with AAD and smartcard support

On Fedora/Nobara-style systems:

    sudo dnf install pcsc-lite pcsc-lite-ccid opensc
    sudo systemctl enable --now pcscd

Install Playwright:

    python3 -m pip install --user playwright
    python3 -m playwright install chromium

## Configuration

Copy the example config:

    mkdir -p ~/.config/xfreerdp-flankspeed
    cp examples/config.example.env ~/.config/xfreerdp-flankspeed/config.env

Edit it:

    nano ~/.config/xfreerdp-flankspeed/config.env

Example:

    XFREERDP_PATH="$HOME/freerdp-webview/build/client/X11/xfreerdp"
    RDPW_PATH="$HOME/Desktop/East NVDesktop-10.rdpw"
    BROWSER_PROFILE="$HOME/.cache/avd-token-browser"
    USER_UPN="first.last.civ@example.mil"

Do not commit your real config file.

## Usage

Install the launcher:

    mkdir -p ~/.local/bin
    cp xfreerdp-flankspeed ~/.local/bin/
    chmod +x ~/.local/bin/xfreerdp-flankspeed

Run:

    xfreerdp-flankspeed

## Applying the FreeRDP patch

The FreeRDP patch is stored at:

    patches/freerdp-avd-gov-spa-pkce.patch

Apply it to a matching FreeRDP source tree:

    cd ~/freerdp-webview
    git apply /path/to/xfreerdp-flankspeed/patches/freerdp-avd-gov-spa-pkce.patch
    cmake --build build -j"$(nproc)"

Then point `XFREERDP_PATH` in the config file to the patched `xfreerdp` binary.

## Troubleshooting

### SCARD_E_SHARING_VIOLATION

This usually means Chromium is still holding the CAC while FreeRDP tries to use it.

The launcher closes Chromium before feeding the final redirect URL into FreeRDP to avoid this.

### tcgetattr() failed

FreeRDP expects a real terminal for the AAD redirect prompt.

The launcher runs FreeRDP inside a pseudo-terminal to avoid this.

### Ctrl+C leaves FreeRDP open

The launcher starts FreeRDP in its own process group and kills it during cleanup.

If cleanup ever fails, manually run:

    pkill -f "xfreerdp"
    pkill -f "avd-token-browser"
    pkill -f "playwright"

### Browser times out while loading the web client

The launcher treats slow browser navigation as non-fatal and continues waiting for the token request.

If login gets stuck, close leftover browser processes and retry:

    pkill -f "avd-token-browser"
    pkill -f "playwright"
    xfreerdp-flankspeed

## Security notes

Never commit:

- `.rdpw` files
- bearer tokens
- refresh tokens
- ID tokens
- browser profiles
- HAR files
- final redirected `code=` URLs
- `session_state` URLs
- personal UPNs
- tenant-specific private details

After testing, clear temporary token artifacts:

    unset GW_TOKEN AAD_TOKEN TOKEN
    rm -rf "$HOME/avd-token-candidates"

If a token or full redirect URL is accidentally posted publicly, sign out/revoke sessions before continuing.

## Disclaimer

This is an unofficial local launcher and FreeRDP patch. It is intended to make a valid, already-authorized AVD/CAC workflow work from Linux. It is not affiliated with Microsoft, FreeRDP, or any government organization.
