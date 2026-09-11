# Decentralisation

The decentralised side of Derusting is built from three small ideas working together:

1. **Bind Buddy firmware to Rust services** so the printer can speak TCP, UDP, files, and Marlin from one runtime.
2. **Gossip machine state and jobs** so printers can discover one another and share work.
3. **Accept jobs through a simple HTTP upload flow** and turn them into network messages.

The rest of this part of the book breaks that system into short chapters. Each chapter focuses on one piece of the stack so the implementation stays easy to follow.
