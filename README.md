# xfreerdp-flankspeed

A Linux launcher for connecting to Azure Virtual Desktop / Flank Speed-style remote desktops with CAC smartcard redirection using a patched FreeRDP build.

This project exists because the standard Linux FreeRDP AVD/AAD flow does not fully handle this environment out of the box. The launcher uses a real Chromium browser for CAC/Microsoft authentication, captures the required AVD gateway bearer token, launches patched `xfreerdp`, completes the AAD redirect flow automatically, then closes the browser so the CAC is released for smartcard redirection inside the remote desktop.

See the patch in `patches/freerdp-avd-gov-spa-pkce.patch` for the FreeRDP changes.
