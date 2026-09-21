# Gossiping

[Submitting Jobs](./job_submission.md) gets a file safely onto one machine's USB stick, but that's only half the story. A decentralised system doesn't have a single machine anyone else depends on, so that job needs to end up shared across every machine on the network, and something has to decide who actually gets to print it — without a central broker making that call. Gossiping is how: each machine periodically shouts a small message onto the LAN, and every other machine builds its own local picture of the world purely from what it happens to overhear.

- [Address Book](./address_book.md) — the shared message envelope, a receive loop that deduplicates and dispatches whatever arrives, and the list of peers each machine keeps of who else is out there.
- [File Share](./file_share.md) — using that address book to push a newly uploaded job's file out to every other machine, so any of them can print it.
- [Job Ledger](./ledger.md) — a token-ring ledger of pending jobs, passed from machine to machine, so exactly one machine at a time gets to pick a job to print.

Unlike the single-machine chapters so far, the real payoff here only shows itself with **two machines** on the same network — one machine gossiping to itself doesn't prove much. [Address Book](./address_book.md) still gives you a single-machine sanity check, but plan to have (or borrow) a second Buddy board if you want to see the full picture.
