# Fixing Audio Dropouts in Linux (PipeWire / WirePlumber)

## The Issue

When starting playback after a period of silence, you may hear the first
split second of audio missing or muted. Short sounds (like
notifications) can even be swallowed completely.\
**Reason:** PipeWire (with WirePlumber) suspends the audio device after
a few seconds of inactivity to save power.

## How to Confirm the Problem

Run:

``` bash
watch -cd -n .1 pactl list short sinks
```

-   `RUNNING` → Audio actively playing
-   `IDLE` → Device open but silent
-   `SUSPENDED` → Device fully closed (power saving mode)

By default, after 5 seconds of silence, the state will go to
**SUSPENDED**.

## The Fix: Disable Suspend in WirePlumber

1.  **Create a WirePlumber config override**

``` bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d/
```

2.  **Create `~/.config/wireplumber/wireplumber.conf.d/disable-suspend.conf` and paste this:**

```
monitor.alsa.rules = [
  {
    matches = [
      { node.name = "~alsa_input.*" },
      { node.name = "~alsa_output.*" }
    ]
    actions = {
      update-props = { session.suspend-timeout-seconds = 0 }
    }
  }
]
```

3.  **Restart PipeWire and WirePlumber**

``` bash
systemctl --user restart wireplumber pipewire
```

## How to Verify the Fix

Run again:

``` bash
watch -cd -n .1 pactl list short sinks
```

-   After playback stops, state should go to **IDLE** and stay there
    (never suspends).

## TL;DR

-   **Problem:** Audio device suspends after silence → first-note
    dropout.
-   **Solution:** Set `session.suspend-timeout-seconds = 0` in
    WirePlumber.
-   **Verification:** Sink state remains **IDLE** instead of
    **SUSPENDED** when silent.

## Note
Sink state will remain suspended before first sound after logging in the system, so to wake it up, you can add a login sound of your choice in autostart, like this command:
```
canberra-gtk-play -f /usr/share/sounds/LinuxMint/stereo/desktop-login.ogg
```

------------------------------------------------------------------------

**Extra Tip:** If configs fail, you can run a silent audio loop to keep
the device awake --- but fixing WirePlumber config is cleaner. Also if your system doesn't use PipeWire/WirePlumber then take help from AI.
