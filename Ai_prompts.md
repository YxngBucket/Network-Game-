# AI Prompts 

## 1.
The implemention should follow the Tic-Tac-Toe application protocol in the protocol blueprint 

AI cant invent new fields, state translations, message types, or framing rules 

## Protocol Constrained Prompt 
-Uses TCP for all communction 
-UTF-8 encoding JSON which contains a type and end with a newline \n
-Can only use the following messages 
-Connect, Lobby_Wait, Game_Start, Move, State_Update, Error, Disconnect, Game_Over

-Don't creat any other message types 
-Player 1 is O and Player 2 is X which Player 1 goes first
-Check row and column before changing the board 
-Handle disconnect messages, Recv() returning b""
-Use server state machine in 'fsm_specification.md'


## FSM-Constrined prompt
Do not skip or invent state transitions.

Valid moves proceed through:
-Player_Turn → Evaluate_Move → Check_If_Win_or_Draw
Invalid moves must not change the board.
Wins, draws, and forfeits transition to Game_Over followed by Clean_Up.
