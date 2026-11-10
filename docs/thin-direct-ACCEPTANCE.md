# Home acceptance: macOS straight on the panel (no GNOME, no Wayland)

The tablet is left **in direct mode**: the macOS VM full screen with no desktop underneath. If it ever shows GNOME instead, open
the app grid and start **"macOS (direct, no desktop)"**. It asks for your password, GNOME closes, and macOS comes up in about a
minute.

Check, with your fingers and the keyboard cover (no other tools needed):
1. The macOS login window fills the panel, sharp, right way up, colours normal.
2. Tap the password field, type your password on the cover keyboard, press Return → the desktop appears.
3. Tap the Apple menu (top-left) → it opens where you tapped. Tap each corner area → the pointer lands under your finger
   (not mirrored).
4. Close the keyboard cover / smart cover → the screen goes dark. Open it → macOS is back where it was.
5. Drag a Finder window by its title bar → it follows your finger. Two-finger scroll in Safari → the page scrolls.
6. Play a video or sound in Safari → sound from the tablet speakers.
7. Detach and re-attach the keyboard cover → typing works again within a few seconds, and no key stays stuck.
   Bluetooth keyboards: only ones already paired in GNOME work; they reconnect by themselves. Pairing a new one needs GNOME.
8. Plug in the charger → it charges (fast charging as before).
9. Apple menu → Restart → macOS restarts on the panel (it does not go to GNOME).
10. **The way back, A:** hold **Volume Up + Volume Down together for 2 seconds** → macOS shuts down cleanly and GNOME comes
    back by itself (about 30 s). The chord also nudges macOS's volume a little, and it works once per VM run.
11. **The way back, B:** start direct mode again, then in macOS choose Apple menu → Shut Down → GNOME comes back by itself.
12. Short press on the power button = the whole tablet shuts down cleanly (macOS first). If the tablet ever stops responding
    entirely, hold the power button as you would on a phone.

Known gaps (not failures): the cover's touchpad does nothing in direct mode; the screen only switches off with the cover
(there is no idle timeout); booting the tablet straight into this (with no Ubuntu desktop at all) needs the USB-C cable to
the Mac for the boot-image step.

If something fails, note the step number. Everything is logged on the tablet: `~/vm/logs/direct.log`, and
`journalctl -u macos-vm` (lines starting with "kms:").
