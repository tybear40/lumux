# Files

- [Video and Reading Modes](video-reading-modes.md) - ModeManager state machine (OFF/VIDEO/READING) with mutually exclusive transitions: video drives DTLS entertainment streaming, reading sends one-time REST puts; covers the 1-second deactivation delay, the _reading_activation_pending race guard, auto-activation when video stops, and target-light resolution that never falls back to all bridge lights.
