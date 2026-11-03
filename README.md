# Herald

A webhook delivery service. Apps send events to Herald; Herald delivers each one to every subscribed endpoint, with retries, signatures and a dead-letter queue, so no event is lost and one slow endpoint doesn't hold up the rest.

**Status:** work in progress. See [REQUIREMENTS.md](REQUIREMENTS.md) for the spec and milestones.

Stack: Python 3.12, FastAPI, Postgres 16, Redis 7, Docker Compose.
