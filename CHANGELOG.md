# Changelog

All notable changes. Versions follow [Semantic Versioning](https://semver.org/).

## 1.0.0 (2026-11-05)

First public release.

- Incoming, missed, answered and outgoing calls with phone, line, caller name and number; restores the `+`
  Yealink sends unencoded; keeps the 50 most recent calls; `clear` and `drop` commands; works with several
  phones on one sensor.
- A call number reused after a phone reboot keeps the older call in the log; unsupported Action URL variables and
  unknown events are ignored.
- Tested live on Home Assistant 2024.11.0, 2026.3.3 and 2026.11.0.
