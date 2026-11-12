# Open Source Contributions

A record of my contributions to open source projects: what the problem was, how I fixed it, and how I verified the fix.

## Contributions

| Project | Contribution | Type | Status |
|---|---|---|---|
| [sqlmap](https://github.com/sqlmapproject/sqlmap) | [#6181: Close leaked sqlite3 connections](https://github.com/sqlmapproject/sqlmap/pull/6181) | Bug fix (Python) | ✅ Merged (2026-11-12) |

---

### sqlmap: Close leaked sqlite3 connections

**Project:** [sqlmapproject/sqlmap](https://github.com/sqlmapproject/sqlmap), the open source automatic SQL injection and database takeover tool
**PR:** [#6181](https://github.com/sqlmapproject/sqlmap/pull/6181) · **Language:** Python · **Status:** ✅ Merged by the maintainer on 2026-11-12 (upstream commit [`cb52165`](https://github.com/sqlmapproject/sqlmap/commit/cb5216564))

#### The problem
Python 3.13+ warns when a SQLite connection is garbage collected without being closed. Running sqlmap's test suite on Linux with CPython 3.14 produced more than 20 `ResourceWarning: unclosed database` warnings. sqlmap's CI only tests Python 3.14 on Windows, so the warnings never showed up there.

The root causes:
- **A real leak in the product:** the SQLAlchemy direct-connection backend (`-d sqlite://...`) closed its connection by returning it to SQLAlchemy's connection pool. The underlying SQLite connection stayed open, which can also leave the database file locked.
- **A common Python gotcha:** `with sqlite3.connect(...) as conn:` commits on exit but does **not** close the connection. `--vuln-test` used this form.
- Two test modules never closed the databases they opened.

#### The fix
- `SQLAlchemy.close()` now also disposes of the engine, which releases the pooled connection. It does this even if the normal close fails, and only logs any error.
- `vulnTest()` uses `contextlib.closing()` with an explicit commit.
- The tests close their connections with `unittest`'s `addCleanup`.

#### Verification
- All 2751 unit tests pass, and the "unclosed database" warnings went from 20+ to 0.
- With only the SQLAlchemy change reverted, the leak still appears even though `close()` is called, which shows that change is the real fix.
- sqlmap's end-to-end `--vuln-test` passes with no sqlite warnings.

**Skills:** Python, debugging resource leaks, SQLite and SQLAlchemy internals, writing unit tests, reading an unfamiliar large codebase.
