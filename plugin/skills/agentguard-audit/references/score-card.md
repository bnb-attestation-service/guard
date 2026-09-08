# The score card

Print this the first time a user sees an AgentGuard score in a conversation — right after the
first scan of `/aguard-setup`, and whenever they ask what the score means or why it is what it
is. On routine scans it is optional, and never re-print it for a user who has already seen it
in this session. Like the usage card, it lives in one place — edit it here, nowhere else.

Print it as written — it explains the number the report already shows, so nothing is filled in.

---

How to read an AgentGuard score:

- **The bands** — 85–100 Low · 70–84 Watch · 50–69 Elevated · below 50 High. One critical
  finding caps the score at 49; one high caps it at 69.
- **It is an average** across everything installed, so one bad skill hides behind many clean
  ones. The number that matters is the worst artifact's score — the report names it.
- **Higher than last time is not automatically better** — installing more clean skills raises
  the average without fixing anything.
- **"No findings" means none from the static rules.** The score is a risk signal, not a safety
  certificate: a static scan cannot see runtime behaviour or prove intent.
