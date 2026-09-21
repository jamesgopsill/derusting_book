# Submitting Jobs

With [Binding with Buddy](./binding.md)'s five primitives in place, a machine can already talk TCP and read/write files — enough to accept a job from a human, even before it knows anything about the rest of the network. This section covers that entry point: a deliberately tiny HTTP server, the single-page form it serves, and the upload flow that turns a PUT request into a file safely on disk.

- [HTTP Service](./http.md) — how much HTTP we actually implement (not much, on purpose) and what happens on a malformed or unsupported request.
- [Submission Portal](./portal.md) — the upload page itself, embedded straight into the firmware binary and served identically by every machine.
- [File Upload](./upload.md) — the PUT handler that streams an uploaded file onto the USB stick.

Everything here builds directly on [LWIP TCP](./lwip_tcp.md) and [FS](./fs.md) — if you haven't been through those yet, start there first. Getting a copy of that uploaded file onto every other machine, and deciding who actually prints it, is covered next in [Gossiping](./gossip.md).
