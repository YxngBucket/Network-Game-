# AI Prompts 

## 1.
The implemention should follow the Tic-Tac-Toe application protocol in the protocol blueprint 

AI cant invent new fields, state translations, message types, or framing rules 

## Protocol Constrained Prompt 
-Uses TCP for all communction 
-Use the protocol_blueprint.md and fsm_specification.md as the implementation contract 
-UTF-8 encoding JSON which contains a type and end with a newline \n

-Don't create any other message types 
-Player 1 is O and Player 2 is X which Player 1 goes first

-The server must correctly validate player corrdinates, turns, occuiped cells, and identity before changing the board  

-Handle disconnect messages, Recv() returning b"", BrokenPipeError, and ConnectionResetError 
-Use server state machine in 'fsm_specification.md'
-Follow the FSM exactly and don't create undocumented transitions 
-Use these message types exactly:
CONNECT, LOBBY_WAIT, GAME_START, MOVE, STATE_UPDATE, ERROR, DISCONNECT, GAME_OVER 

-Maintain a recived buffer because TCP may combine or split messages 


## FSM-Constrined prompt
Do not skip or invent state transitions.

Valid moves proceed through:
-PLAYER_TURN → EVALUTE_MOVE → CHECK_WIN_DRAW
Invalid moves must not change the board.
Wins, draws, and forfeits transition to GAME_OVER followed by CLEANUP.
