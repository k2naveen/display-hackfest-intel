# Judder with low-fps video playback under VRR

## Overview

- On a 4K60 panel with a 40 to 60 Hz VRR window, refresh can vary only between
16.67 ms and 25 ms.
- But 30 fps video delivers a new frame every 33.33 ms, which falls below the
panel’s minimum VRR rate.
- Because of this, frames cannot be displayed at a uniform cadence and instead
alternate between longer and shorter display times, about 41.67 ms and 25 ms.
- The average frame rate remains correct, but the uneven frame pacing appears
as visible judder.


![](/images/judder_with_media_playback_vrr.png)

## How can this be fixed in userspace?
- The client passes the content frame rate to the compositor.

- The compositor then chooses the smallest stable refresh multiple that fits within the VRR range.

- For 30 fps content on a 40 to 60 Hz panel, that target becomes 60 Hz, so each video frame is shown for exactly two scanouts.

- This restores a stable presentation cadence and removes judder.

- Mutter implements this by deriving a virtual display mode from the current KMS mode and retiming scanout to the computed target refresh rate.

## More details
For the full PoC and implementation details, see:
[Fixing judder issue with low fps video playback](https://k2naveen.github.io/wayland/vrr/2026/05/23/fixing-judder-issue-with-low-fps-video-playback.html)
