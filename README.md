cd "$HOME/xfreerdp-flankspeed"

cat > README.md <<'EOF'
# xfreerdp-flankspeed

`xfreerdp-flankspeed` is a Linux launcher for connecting to Azure Virtual Desktop / Flank Speed-style remote desktops with CAC smartcard redirection using a patched FreeRDP build.

The launcher uses Chromium for CAC/Microsoft authentication, captures the current AVD gateway bearer token from the browser session, launches patched `xfreerdp`, completes the AAD redirect flow automatically, closes Chromium to release the CAC, and connects with smartcard redirection enabled.

## Important

This repository does not include a full patched FreeRDP source tree or compiled FreeRDP binary.

It includes:

    xfreerdp-flankspeed launcher
    FreeRDP patch file
    example config
    documentation

You must build FreeRDP yourself and apply the patch from this repo.

## What works

Tested locally with:

    Linux host
    CAC authentication through Chromium
    Azure Virtual Desktop gateway token capture
    Patched FreeRDP AAD login
    CAC/smartcard redirection into the remote Windows desktop
    One-command launcher flow

## What this does not do

This does not bypass authentication.

You still need:

    Valid CAC
    Working smartcard reader
    Assigned Azure Virtual Desktop resource
    Valid .rdpw file from your environment
    Patched FreeRDP build

This repository does not include:

    .rdpw files
    bearer tokens
    refresh tokens
    ID tokens
    browser profiles
    certificates
    user-specific government data

## How it works

The launcher automates this flow:

1. Opens the AVD web client in Chromium.
2. User signs in normally with CAC.
3. Watches for the AVD feed discovery request.
4. Extracts the current Authorization bearer token from that browser request.
5. Starts patched xfreerdp with the gateway bearer token.
6. Waits for FreeRDP to print the AAD login URL.
7. Opens that AAD login URL in Chromium.
8. Captures the final aadj.html code redirect.
9. Closes Chromium so the CAC is released.
10. Feeds the final redirect URL back into FreeRDP.
11. FreeRDP connects and redirects the CAC/smartcard into the remote desktop.

## Repository contents

    xfreerdp-flankspeed/
    ├── README.md
    ├── xfreerdp-flankspeed
    ├── patches/
    │   └── freerdp-avd-gov-spa-pkce.patch
    ├── examples/
    │   └── config.example.env
    └── .gitignore

## Step 1: Install system dependencies

Fedora / Nobara:

    sudo dnf install git cmake ninja-build gcc gcc-c++ openssl-devel \
      pcsc-lite pcsc-lite-ccid opensc fuse3-devel \
      libX11-devel libXext-devel libXcursor-devel libXi-devel \
      libXinerama-devel libxkbfile-devel libXrandr-devel \
      libXrender-devel libXv-devel libxkbcommon-devel \
      cups-devel alsa-lib-devel pulseaudio-libs-devel \
      wayland-devel wayland-protocols-devel

Enable the smartcard service:

    sudo systemctl enable --now pcscd

Verify your smartcard reader is visible:

    opensc-tool -l

Optional deeper test:

    pcsc_scan

## Step 2: Install Playwright

The launcher uses Playwright Chromium for CAC/browser authentication.

Install it:

    python3 -m pip install --user playwright
    python3 -m playwright install chromium

## Step 3: Clone this repo

    git clone https://github.com/gearsovwar/xfreerdp-flankspeed.git
    cd xfreerdp-flankspeed

## Step 4: Clone FreeRDP

Clone FreeRDP somewhere outside this repo:

    cd "$HOME"
    git clone https://github.com/FreeRDP/FreeRDP.git freerdp-webview
    cd freerdp-webview

## Step 5: Apply the FreeRDP patch

From the FreeRDP source folder:

    git apply "$HOME/xfreerdp-flankspeed/patches/freerdp-avd-gov-spa-pkce.patch"

If the patch does not apply cleanly, make sure you are using a compatible FreeRDP version.

## Step 6: Configure FreeRDP

Example build configuration:

    cmake -S . -B build -G Ninja \
      -DWITH_AAD=ON \
      -DWITH_WEBVIEW=ON \
      -DWITH_PCSC=ON \
      -DWITH_SMARTCARD=ON \
      -DWITH_SMARTCARD_PCSC=ON \
      -DWITH_PKCS11=ON \
      -DWITH_FFMPEG=OFF \
      -DWITH_DSP_FFMPEG=OFF \
      -DWITH_VIDEO_FFMPEG=OFF \
      -DWITH_SWSCALE=OFF

Build it:

    cmake --build build -j"$(nproc)"

Your patched binary should be here:

    $HOME/freerdp-webview/build/client/X11/xfreerdp

Verify build options:

    "$HOME/freerdp-webview/build/client/X11/xfreerdp" /buildconfig | grep -E "WITH_AAD|WITH_WEBVIEW|WITH_PCSC|WITH_SMARTCARD|WITH_PKCS11"

## Step 7: Configure xfreerdp-flankspeed

Create the config folder:

    mkdir -p ~/.config/xfreerdp-flankspeed

Copy the example config:

    cp "$HOME/xfreerdp-flankspeed/examples/config.example.env" \
      ~/.config/xfreerdp-flankspeed/config.env

Edit it:

    nano ~/.config/xfreerdp-flankspeed/config.env

Example config:

    XFREERDP_PATH="$HOME/freerdp-webview/build/client/X11/xfreerdp"
    RDPW_PATH="$HOME/Desktop/East NVDesktop-10.rdpw"
    BROWSER_PROFILE="$HOME/.cache/avd-token-browser"
    USER_UPN="first.last.civ@example.mil"

Do not commit your real config file if you decide to contribute.

## Step 8: Install the launcher

From this repo folder:

    cd "$HOME/xfreerdp-flankspeed"
    mkdir -p ~/.local/bin
    cp xfreerdp-flankspeed ~/.local/bin/
    chmod +x ~/.local/bin/xfreerdp-flankspeed

Make sure ~/.local/bin is in your PATH:

    echo "$PATH" | grep "$HOME/.local/bin"

If it is not, add this to your shell profile:

    export PATH="$HOME/.local/bin:$PATH"

## Step 9: Run it

    xfreerdp-flankspeed

Expected flow:

    Browser opens
    You sign in with CAC if needed
    Launcher captures the AVD gateway token
    Patched FreeRDP starts
    Browser opens the FreeRDP AAD login URL
    Launcher captures the final redirect
    Browser closes to release the CAC
    FreeRDP connects
    CAC is available inside the remote desktop

## Why FreeRDP had to be patched

The target environment required a Gov AVD/AAD flow plus browser-style SPA token redemption. Stock FreeRDP did not produce the exact flow needed.

The patch modifies FreeRDP in these areas.

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

### FreeRDP cannot find the .rdpw file

Check your config:

    cat ~/.config/xfreerdp-flankspeed/config.env

Make sure RDPW_PATH points to your actual .rdpw file.

### FreeRDP binary not found

Check your config:

    cat ~/.config/xfreerdp-flankspeed/config.env

Make sure XFREERDP_PATH points to the patched FreeRDP binary.

## Security notes

i originially removed this as i didnt think anyone would do it but i should mention if you do contribute NEVER commit the following files.

    .rdpw files
    bearer tokens
    refresh tokens
    ID tokens
    browser profiles
    HAR files
    final redirected code URLs
    session_state URLs
    personal UPNs
    tenant-specific private details

to remove temporary short lived tokens from local storage.

    unset GW_TOKEN AAD_TOKEN TOKEN
    rm -rf "$HOME/avd-token-candidates"

If a token or full redirect URL is accidentally posted publicly, sign out and revoke sessions before continuing.

## Development notes

Check tracked files:

    git ls-files

Check for obvious secrets before pushing:

    grep -RInE "eyJ|refresh_token|access_token|id_token|Bearer [A-Za-z0-9]|session_state=|token_01|avd-token-candidates/token" .

## Disclaimer

This is an unofficial local launcher and FreeRDP patch. It is intended to make a valid, already-authorized AVD/CAC workflow work from Linux. It is not affiliated with Microsoft, FreeRDP, or any government organization.
EOF
