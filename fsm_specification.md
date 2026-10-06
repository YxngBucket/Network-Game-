# Tic-Tac-Toe FSM Specification 

## The server will us a FSM to control the Tic-Tac-Toe game. The server is also managing the board, determining draws and wins, handing errors and disconnections, and determining who wins.

## INIT
Server prepares listening socket and starts game and will transtion to WAITING_FOR_PLAYERS

## WAITING_FOR_PLAYERS
Server will begin waiting for 2 players to connect 
Server sends Lobby_wait when player 1 connects, after player 2 connects ther server will contuine to Game_Start 

## GAME_START
Server will Assign Player 1 = O and Player 2 = X, Player 1 is the first player 

## PLAYER_TURN
Waits for the players Move for whoevers turn it is
When the player sends a move, the server will evaluate the move 
If the wrong player sends a move, the server sends an Error message 
If a player does disconnect, the game ends, the player forfeits, and the game ends 

## EVALUATE_MOVES 
Server manages move requests 
-Cell is occupied 
-Row or Column is outside 0-2
-Message is malformed 

Incorrect moves will result in error message and will go back to player_turn

## Check_If_Win_Or_Draw 
Server will check if the move resulted in a win, a draw, or neither 

If there is a winner or draw the server will result in a Game_Over

If the game continues, the server will switch the active player and will return to Player_Turn  

## ERORR 
Server will send an error to the client, the board is not changed, the server will return to player_turn 

## GAME_OVER
Server will result Game_Over
-Win, Draw, Player forfeit 

Transitions to Clean_Up

## CLEANUP
Server handles resetting the board and game, connections, and creates new game 
Will go back to waiting for players 

## 3. Mermaid State Diagram

```mermaid stateDiagram-v2
    [*] --> INIT
    INIT --> WAITING_FOR_PLAYERS : Server initializes

    WAITING_FOR_PLAYERS --> WAITING_FOR_PLAYERS : Player 1 connects / send LOBBY_WAIT
    WAITING_FOR_PLAYERS --> GAME_START : Player 2 connects
    WAITING_FOR_PLAYERS --> Clean_Up : Waiting client disconnects

    GAME_START --> PLAYER_TURN : Assign P1=O, P2=X; P1 starts

    PLAYER_TURN --> EVALUATE_MOVE : MOVE from active player
    PLAYER_TURN --> Error_State : MOVE from wrong player
    PLAYER_TURN --> GAME_OVER : Graceful Disconnect / forfeit
    PLAYER_TURN --> GAME_OVER : Abrupt disconnect / forfeit

    Error_State --> PLAYER_TURN : Send ERROR; board unchanged

    EVALUATE_MOVE --> Error_State : Invalid coordinates
    EVALUATE_MOVE --> Error_State : Cell occupied
    EVALUATE_MOVE --> Error_State : Malformed MOVE
    EVALUATE_MOVE --> Check_If_Win_Or_Draw: Valid move
    EVALUATE_MOVE --> GAME_OVER : Abrupt disconnect / forfeit

    Check_If_Win_Or_Draw --> GAME_OVER : Three in a row
    Check_If_Win_Or_Draw --> GAME_OVER : Board full / draw
    Check_If_Win_Or_Draw --> PLAYER_TURN : No winner and board not full / switch turn

    GAME_OVER --> CLEANUP : Broadcast GAME_OVER

    CLEANUP -->  WAITING_FOR_PLAYERS : Reset game state

