# P2P Distributed File Sharing System

A BitTorrent-style peer-to-peer file sharing system written in C++ using raw POSIX sockets and threads, with no external networking or hashing libraries.

Users register with a tracker, form groups, and share files within them. Files are split into 512 KB pieces, downloaded in parallel from multiple peers, and verified with SHA-1 at both the piece level and the whole-file level.

## Features
- **Two-tracker setup with client failover.** If the connected tracker goes down, the client switches to the other tracker and keeps its session.
- **User and group management.** Register and log in with session tokens. Create, join, leave and list groups. The group owner approves join requests.
- **Piece-based transfer.** Files are split into 512 KB pieces. Each piece is fetched directly from a peer (client to client).
- **Parallel multi-peer download.** A pool of 8 worker threads fetches pieces from different seeders at the same time, with retry and fallback to another peer.
- **Integrity checks.** SHA-1 is implemented from scratch. Every piece is checked as it arrives. Corrupted pieces are discarded and fetched again. The whole file is checked at the end.
- **Automatic seeding.** A completed download is announced to the tracker, so the downloader becomes a seeder.
- **Re-announce on login.** Files you shared before are re-registered automatically when you log in again.
- **Thread-safe tracker.** Each client gets its own thread. Shared state is protected by mutexes.

## Architecture
```
        +-----------+      sync_log.txt         +-----------+
        | Tracker 1 | <--- (append/replay) ---> | Tracker 2 |
        +-----------+                           +-----------+
              ^   metadata, auth, peer lists         ^
              |                                      |
        +-----------+   GET_PIECE <file> <idx>  +-----------+
        |  Client A | <-----------------------> |  Client B |
        +-----------+      512 KB raw piece     +-----------+
```
- **Tracker**: stores users, groups, join requests and file metadata (size, piece hashes, full hash, list of seeders). Every state change is written to an append-only operation log. When a tracker starts, it replays the log. A watcher thread applies any new entries, which keeps the two trackers in sync.
- **Client**: an interactive shell that talks to the tracker, plus a peer server running on a background thread that serves file pieces to other peers.

## Build and run (Linux)
```bash
cd tracker && make
./tracker tracker_info.txt 1      # in another terminal: ./tracker tracker_info.txt 2

cd client && make
./client 127.0.0.1:40000 tracker_info.txt
```
`tracker_info.txt` contains one `IP PORT` per line.

## Commands
| Command | Description |
|---|---|
| `create_user <uid> <pwd>` / `login <uid> <pwd>` / `logout` | Account and session |
| `create_group <gid>` / `join_group <gid>` / `leave_group <gid>` | Groups |
| `list_groups` / `list_requests <gid>` / `accept_request <gid> <uid>` | Group administration |
| `upload_file <gid> <path>` / `list_files <gid>` / `stop_share <gid> <file>` | Sharing |
| `download_file <gid> <file> <dest_dir>` / `show_downloads` | Downloading |

## Protocol
- **Client to tracker:** newline-terminated text commands. When logged in, each command is prefixed with `session <uid> <token>`. Replies end with an `END_OF_REPLY` marker line.
- **Peer to peer:** the downloader sends `GET_PIECE <filename> <index>\n`. The seeder replies with the raw bytes of that piece. The downloader checks the bytes against the piece's SHA-1 hash from the tracker.

## Tech stack
C++11, POSIX sockets (TCP), `std::thread` / `std::mutex` / `std::atomic`, a hand-written SHA-1 implementation, Make.

## Known limitations
- The trackers sync through a shared log file, so both must run on the same machine. TCP-based sync between trackers is planned.
- A client's downloads run one at a time. Pieces within a single file are downloaded in parallel.
- Seeders are not removed from the tracker when a client disconnects without logging out.
- File and user names cannot contain spaces.

## Credits
The SHA-1 implementation is adapted from Steve Reid's public-domain SHA-1.
