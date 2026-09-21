# Binding with Buddy

Before machines can gossip with one another or accept jobs, a single machine needs to be able to do five things: run our own async Rust code alongside the firmware, talk UDP, talk TCP, read and write files, and drive the printer itself. This part of the book builds each of those five primitives in turn, and each one ends with something you can flash and see for yourself:

- [Embassy Async Runtime in FreeRTOS](./embassy.md) — get a task running and see it boot.
- [LWIP UDP](./lwip_udp.md) — broadcast a heartbeat and see it printed on the wire.
- [LWIP TCP](./lwip_tcp.md) — serve a real HTML page over the network.
- [FS](./fs.md) — write a file to the USB stick and read it back.
- [Marlin](./marlin.md) — drive the printer itself.

By the end of this section, every primitive [Submitting Jobs](./job_submission.md) and [Gossiping](./gossip.md) depend on will already be working — those later chapters are mostly about wiring these five together.
