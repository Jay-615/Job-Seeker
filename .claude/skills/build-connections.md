---
name: build-connections
description: Populate config/connections.csv from the user's 1st-degree LinkedIn connections. Uses Claude in Chrome to paginate through linkedin.com/mynetwork/invite-connect/connections/ at human pace, writes each connection to the CSV with rating=Unrated. Re-runnable — preserves existing ratings, only adds new connections. Optionally seeds rating=High from a user-provided meetings.csv file.
---

<!-- Implementation pending — module M8. -->
