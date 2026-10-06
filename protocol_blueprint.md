# Tic-Tac-Toe Protocol Blueprint 

# 1.

Application-Layer Protocol using a client and server for Tic-Tac-Toe 

The Game uses TCP to be able to commmuncicate between a server and two clients. The server will be used for the game board, the game state, and switching between player turns. 

Player 1 will be assigned 'O' and Player 2 will be assigned 'X'. Player 1 will start first.

# 2.

Messages are encoded using UTF-8 and serialized using JSON, each message will contain a type that will identify the message type

Example:
```json
{
    "type":"CONNECT", "player_id":"Player1"
}


# TCP Framing 
Protocol will use newline-delimited JSON, each JSON message will end with a newline (\n). The reciver will then store incoming TCP data in a buffer and will complete messages whenever a new line is recived  

Example: 
{
  "msg_type": "MOVE", "player_id": "Player1", "row": 0,"col": 1
}\n
{
    "msg_type": "MOVE", "player_id": "Player2", "row": 1,"col": 2
}\n


# 4.1) `CONNECT` (Client -> Server): Request to join the game room.

{
    "msg_type": "CONNECT",
    "player_id": "Player1",
}

4.2)`LOBBY_WAIT` (Server -> Client): Notification that server is waiting for Player 2.

{
    "msg_type": "LOBBY_WAIT",
    "player_id": "Player1",
    "message": "Waiting for Player 2"
}

4.3) `GAME_START` (Server -> Clients): Game initiated, assigns roles (e.g. Player X vs Player O).

{
    "msg_type": "GAME_START",
    "player_id": "Player1",
    "opponent": "Player2",
    "symbol": "O",
    "player_active": "Player1"
}

4.4 `MOVE` (Client -> Server): Player action (e.g., cell coordinates or answer choice).
{
  "msg_type": "MOVE", "player_id": "Player1", "row": 0,"col": 1
}

4.5 `STATE_UPDATE` (Server -> Clients): Broadcast current game board / state and active player turn.

{
    "msg_type": "STATE_UPDATE",
    "player_id": "Server",
        "board": [
            ["O", "-", "-"],
            ["X", "-", "-"],
            ["-", "-", "-"]
        ],
        "active_player": "Player1"
    },

4.6 `ERROR` (Server -> Client): Invalid move or malformed packet error.
{
    "msg_type": "ERROR",
    "player_id": "Player2",
    "code": "Not_Your_Turn",
    "message": "It is not your turn"
}

4.7 `DISCONNECT` (Client -> Server): Tells the server that a client is leaving 
{
    "msg_type": "DISCONNECT",
    "player_id": "Player1",
}
    
4.8 `GAME_OVER` (Server -> Clients): Victory / Draw notification with final scores.
{
    "msg_type": "GAME_OVER",
    "winner": "Player1",
    "result": "Win",
    "reason": "Three_In_A_Row",
}
{
    "msg_type": "GAME_OVER",
    "result": "Draw",
    "winner": null,
    "reason": "Board_Is_Full",
}

#5
Server is for validation for every move 

Move Validation:
1.) The players turn to move 
2.) The cell is empty 
3.) Row is between 0 and 2
4.) Column is between 0 and 2 

If a move is not possible, the server will send an Error and doesn't change the board 

6.) Win or Draw 
A player will win if they have 3 of there symbol in a vertical, horizontal, or diagonal. The game is a draw when all cells are filled and no one has 3 symbols in a row 

7.) Connection 
Server will handle graceful and unexpected termination. So the client will send a Disconnect message before ending the game and the server will end the game.

TCP FIN / Clean Closure
Recv() will return 0 bytes b"" then the server knows the peer closed its connection which will be know as a player disconnecting 

TCP Reset 
A TCP reset will happen if the connection is interutted 
Handled through socket excpetions: ConnectionResetError

Other Socket Errors:
Timeout Errors 
BrokenPipeError 

8.) Wire Example 
Client:
{
  "msg_type": "MOVE", "player_id": "Player1", "row": 0,"col": 2
}

Server Response:
{
    "msg_type": "STATE_UPDATE",
        "board": [
            ["O", "-", "-"],
            ["-", "-", "-"],
            ["-", "-", "-"]
        ],
        "active_player": "Player1"
}