# Galaxy S25 (SM-S931B) — diagnosis notes (17 Nov 2026)

Hardware: Android 17, kernel 6.6.127-android15-8, Adreno 830, 10 GB RAM. Andromarchy v1.0.1.

## The desktop "crash" is a SIGKILL by Android

When the app's SurfaceView is destroyed (app backgrounded/occluded, screen off, rotation), the app enters
display fallback and the Linux session's processes (registered as Android "phantom processes") are killed
with SIGKILL. Evidence captured by wrapping /usr/bin/Hyprland and logging the exit status:

```
22:04:21.305  daemon: consumer re-deposited 5 fds
22:04:21.305  Andromarchy: fallback triggered
22:04:21.305  ActivityManager: Process PhantomProcessRecord {…}.died  (cascade)
22:04:21.325  Hyprland.real exit rc=137 (SIGKILL)
```

- rc=137 => external SIGKILL (a real Hyprland fault gives rc=134/139; manual restart outside the app aborts with CBackend::create() failed, rc=134).
- No lmkd kill, earlyoom thresholds not met (MemAvailable ~3.5 GB at kill; Hyprland in --avoid), no ANR/crash records.
- The session does not auto-restart after the kill; the app must be force-stopped and reopened.
- opencode was a red herring: any activity that hides the app's surface triggers the same kill.

## Workarounds
- Keep the app foreground while using the desktop; if you must switch apps, use split-screen (both visible) or PiP.
- Recovery: `am force-stop andro.marchy && am start -n andro.marchy/.MainActivity` (script: ~/recover-andromarchy.sh on the host).
- Keep the screen on.

## Fix direction
- App-side: park the session on fallback instead of letting Android reap the process tree, and/or restart the
  session on exit_fallback; optionally add a debug option to keep the compositor alive in background.
- OS-side: max_phantom_processes is already at INT_MAX and settings_enable_monitor_phantom_procs=false on this device.

## Secondary observations
- Hyprland logs are disabled (stdout redirected by the app), so enable a wrapper or read the archived session journal.
- `linux_dmabuf_feedback: Failed to fetch DRM device` and quickshell falling back to SHM (no /dev/dri, KGSL path).
- Manual Hyprland restart outside the app fails: CBackend::create() failed.
