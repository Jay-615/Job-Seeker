---
name: network-check
description: Tier 1 network-proximity analysis for a single company. Uses Claude in Chrome on the user's LinkedIn People search to count 1st- and 2nd-degree connections, identifies which 1st-degrees appear as mutuals, cross-references against config/connections.csv ratings to compute bonus points, returns base/bonus/final scores plus warmest intro paths. Caches results in jobs/network-cache.json per the settings.md TTL.
---

<!-- Implementation pending — module M4. -->
