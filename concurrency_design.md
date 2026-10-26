Sprint 2: Game Behavior & Server Concurrency Architecture Design

## 1. Overview

The server is the single source of truth for a two-player Tic-Tac-Toe game. Clients only send requests JOIN, MOVE, REMATCH, SCORES, QUIT. The server validates each request, changes state, and pushes the result to both players.

Design goals:

Service 2 simultaneous clients without ever blocking the accept loop.
Make illegal or OutOfTurn moves impossible to apply, even when both players send at the same instant.
Use fine-grained locks so unrelated work (for example, a score lookup) does not wait on game logic.
Guarantee both clients see every state change in the same order.
Keep round scores in server memory and broadcast them after every round.
## 2. Server Concurrency Model

## 2.1 Why threads with perresource locks

Each client gets its own worker thread, so a blocking recv() for one player never stalls the other player or the listener. Shared state is split into resources, each with its own lock, so threads only contend when they genuinely touch the same data. A fixed lock hierarchy (Section 4.3) prevents deadlock.

## 2.2 Thread layout
Thread	Count	Responsibility
Listener (main)	1	Owns the listening socket, loops on accept(), spawns a worker. Never parses game messages.
Client worker	1 per connection	Blocks on recv() for its own socket, parses lines, calls game logic, and sends replies.

Workers are created with threading.Thread(target=handle_client, args=(conn, addr), daemon=True).


## 2.3 Connection lifecycle
Listen: create socket, set SO_REUSEADDR, bind(), listen(5).
Accept: accept() returns (conn, addr). The listener spawns a worker and immediately returns to accept().
Register: the worker waits for JOIN <name>. Under clients_lock it takes a free slot (X or O). If both slots are taken, it replies ERR GAME_FULL and closes the socket.
Start: when the second player registers, the pairing starts: scores are reset, the board is cleared, and START messages are sent. X always moves first.
Service loop: each worker reads lines, dispatches MOVE, REMATCH, SCORES, or QUIT, and replies or broadcasts.
Teardown: on QUIT, empty recv(), or socket error, a finally block removes the client from the registry, resets the game to WAITING, tells the remaining player LEFT, and closes the socket.


## 2.4 Capacity planning
Steady state is 3 threads: the listener and 2 workers.
A third connection is accepted briefly just to be told ERR GAME_FULL, then closed, so it cannot hang in the backlog.
Every socket uses settimeout(30) for sends, so a stalled peer cannot freeze a broadcast forever.

## 3. Message Protocol 

Each message is one line ending in \n. Fields are separated by single spaces. Because name is a single token (1 to 16 letters/digits, no spaces), splitting on spaces is always safe.

Board encoding: a 9-character string in row-major order using X, O, and . for empty. Index = row * 3 + col. Example: X.O.X...O means X at (0,0), O at (0,2), X at (1,1), O at (2,2).

Client → Server
Message	| Meaning
JOIN <name> | Request a slot, for example JOIN Alice
MOVE <row> <col> | Place my mark, for example MOVE 0 2 (rows and columns are 0 to 2)
REMATCH | Vote to play another round
SCORES	| Ask for the current scoreboard (read-only)
QUIT | Leave the game


Server → Client
Message	| Meaning
WAITING <role>	| Joined as X or O, waiting for an opponent
START <role> <opponent> <turn> <board> | A round begins
STATE <seq> <turn> <board> | Authoritative board after every valid move
OVER <result> <board> <line> | Round finished. result is X_WINS, O_WINS, or DRAW. line is the winning cells like 0,1,2, or - for a draw
SCORES <x> <o> <draws> <rounds>	| Scoreboard
VOTES <n> | Number of rematch votes so far (0 to 2)
LEFT | The opponent disconnected
ERROR <code> <text> | Request rejected. Codes: NOT_YOUR_TURN, CELL_TAKEN, OUT_OF_RANGE, GAME_NOT_ACTIVE, GAME_FULL, BAD_MESSAGE

Framing: each worker keeps a receive buffer and splits on \n, so partial TCP reads never produce half-parsed commands. Lines longer than 64 bytes are rejected with ERROR BAD_MESSAGE.

## 4. Thread Safety, Lock Acquisition & State Isolation

## 4.1 Shared resources and their locks
Resource	Contents	Lock	Why it is its own resource
Client registry	slots = {"X": Player or None, "O": Player or None}	clients_lock	Changes only on join and leave, and is read by every broadcast
Game state	board, current_turn, status, seq, rematch_votes	game_lock	These fields share invariants (see 4.2), so they must change together
Scoreboard	scores = {X, O, draws}, rounds_played	score_lock	Written once per round but readable at any time via SCORES
Each socket's write side	Player.conn.sendall()	Player.send_lock	Two threads may send to the same client, and sendall calls must not interleave bytes


## 4.2 Why board and current_turn share one lock

Separate locks for the board and the turn would look more "fine-grained" but would be wrong. A move must check the turn, write a cell, and flip the turn as one atomic step. If the turn and board had separate locks, two threads could each pass the turn check before either flips it. Related fields with a shared invariant belong under one lock.

## 4.3 Lock hierarchy (deadlock prevention)

Locks may only be acquired in this order, never backwards:

game_lock  →  score_lock  →  Player.send_lock

clients_lock is a leaf lock: it is held only for a very short registry read or write, and no other lock is ever acquired while holding it.

Rules that follow from this:

Snapshot-then-release for the registry. Code that needs both players' sockets calls snapshot_clients() before taking game_lock. The function takes clients_lock, copies the two Player references, releases the lock, and returns the copy.
Never wait for another lock while holding send_lock. It is the innermost lock and is held only for one sendall().
recv() is always called with no locks held.
Always use with lock:, never bare acquire()/release(), so locks are released even on exceptions.
Helpers never take the lock their caller already holds. For example, broadcast() takes only send_locks, and the caller holds game_lock.

Because every thread follows the same total order, no two threads can each hold a lock the other needs, so deadlock is impossible.

## 4.4 Critical sections
# | Operation |	Locks held (in order) |	Work done
CS1 | Register (JOIN)	| clients_lock (released), then game_lock, then score_lock | Take  a free slot. If two players are present and status is WAITING, reset scores and board, set IN_PROGRESS, send START

CS2 | MOVE | game_lock → score_lock (only on round end) → send_locks | Validate, write cell, check win/draw, flip turn, bump seq, update scores, broadcast

CS3 | REMATCH | game_lock → send_locks | Record vote. When both have voted, clear board, set turn to X, set IN_PROGRESS, broadcast START

CS4 | SCORES query | score_lock → send_lock | Copy the four numbers and reply. Does not touch game_lock, so it never waits for a move in progress

CS5 | Disconnect cleanup | clients_lock (released), then game_lock, then send_locks | Remove from registry, set status WAITING, clear votes, notify opponent

CS1 is safe even if two JOINs happen at the same moment: both register under clients_lock, but the game only starts for whichever thread sees status == WAITING first inside game_lock. The second sees IN_PROGRESS and does nothing.


## 4.5 How each race is prevented
Race | Prevention
Both players send MOVE at the same instant | Whoever gets game_lock first runs to completion. The other blocks, then sees the flipped turn or filled cell and gets ERROR

Same cell chosen twice | Empty-cell check and write happen under one game_lock hold

Lost score update | Increment and rounds_played bump happen together under score_lock

Half-updated scoreboard seen by SCORES | SCORES copies all four numbers under score_lock, and writers update all four under the same lock

Two JOINs both get slot X | Slot assignment happens under clients_lock

Two JOINs both start the game | Start requires status == WAITING inside game_lock, so only one wins

Interleaved bytes on one socket | Each sendall is wrapped in that player's send_lock

Broadcast order differs between clients	| Both broadcast() calls happen inside one game_lock hold, in fixed X-then-O order, and STATE_UPDATE carries seq

Broadcast to a socket that just closed | Send failure is caught (OSError), the player is marked for cleanup by its own worker's finally block

## 4.7 Known trade-offs
1. game_lock is held during sendall() so that broadcast order is guaranteed. A slow client could briefly delay the other player, bounded by the 30-second send timeout. With 2 players and 20-byte messages this is acceptable.

2. Splitting locks adds hierarchy rules the programmer must follow. The benefit is that SCORES queries and registry reads do not wait on move processing.