# Decentralisation

The decentralised side of Derusting is built from four small ideas working together:

1. **Bind Buddy firmware to Rust services** so the printer can speak TCP, UDP, files, and Marlin from one runtime.
2. **Accept jobs through a simple HTTP upload flow** and land them safely on disk.
3. **Gossip machine state and jobs** so printers can discover one another and share work.
4. **Surface machine state back on the printer's own screen** so a technician can see and control what it's doing without needing a laptop.

The rest of this part of the book breaks that system into short chapters. Each chapter focuses on one piece of the stack so the implementation stays easy to follow.
