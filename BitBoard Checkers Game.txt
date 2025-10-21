#include <stdio.h>
#include <string.h>
#include <stdbool.h>
#include <time.h> // Needed for timer

// ================== Bit Helpers ==================
unsigned long long SetBit(unsigned long long value, int position) {
    if (position < 0 || position > 63) return value;
    return value | (1ULL << position);
}
unsigned long long ClearBit(unsigned long long value, int position) {
    if (position < 0 || position > 63) return value;
    return value & ~(1ULL << position);
}
int GetBit(unsigned long long value, int position) {
    if (position < 0 || position > 63) return 0;
    return (value >> position) & 1ULL;
}

// ================== Board Printing & Capture Bars ==================
void print_capture_bar(const char letter, int count) {
    // print up to 8 slots: first `count` letters, then zeroes
    int cap = count;
    if (cap > 12) cap = 12;
    for (int i = 0; i < cap; i++) putchar(letter);
    for (int i = cap; i < 12; i++) putchar('0');
    putchar('\n');
}

void create_board(unsigned long long player1, unsigned long long player1_kings,
                  unsigned long long player2, unsigned long long player2_kings,
                  int red_captures_count, int black_captures_count) {
    printf("\nCurrent board:\n\n");

    // Red captures shown first (player2 captures black pieces)
    printf("Red Captures:\n");
    print_capture_bar('B', red_captures_count);
    printf("\n");
    // Board grid
    for (int r = 0; r < 8; r++) {
        for (int c = 0; c < 8; c++) {
            int bitIndex = 63 - (r * 8 + c); // Top-left is 63
            unsigned long long mask = 1ULL << bitIndex;

            if (player1_kings & mask)         printf("%3s", "KB");
            else if (player1 & mask)          printf("%3s", ".B");
            else if (player2_kings & mask)    printf("%3s", "KR");
            else if (player2 & mask)          printf("%3s", "*R");
            else                              printf("%3d", bitIndex);
        }
        printf("\n");
    }

    // Black captures shown after board (player1 captures red pieces)
    printf("\nBlack Captures:\n");
    print_capture_bar('R', black_captures_count);

    printf("\n");
}

// ================== Utility ==================
// Left edge (column c==0) for bitIndex map top-left=63
int LeftEdge[]  = {63, 55, 47, 39, 31, 23, 15, 7};
int RightEdge[] = {56, 48, 40, 32, 24, 16, 8, 0};

bool IsLeftEdge(int pos) {
    for (int i = 0; i < 8; i++) if (pos == LeftEdge[i]) return true;
    return false;
}
bool IsRightEdge(int pos) {
    for (int i = 0; i < 8; i++) if (pos == RightEdge[i]) return true;
    return false;
}
bool InBounds(int pos) { return pos >= 0 && pos <= 63; }

// ================== Movement (simple & jumps) ==================
// TryMove: attempts move for *player; on capture increments the capture counter via pointer.
bool TryMove(unsigned long long *player, unsigned long long *opponent,
             unsigned long long *player_kings, unsigned long long *opponent_kings,
             int from, int to, int playerId, int *capture_counter) {
    // playerId: 1 = player1 (black), 2 = player2 (red)
    if (!InBounds(from) || !InBounds(to)) return false;
    if (!GetBit(*player, from)) return false;   // no piece at source
    if (GetBit(*player, to)) return false;      // destination occupied by our piece

    int delta = to - from;
    int absd = delta >= 0 ? delta : -delta;

    // allowed deltas: 7,9,14,18 (and negative)
    if (!(absd == 7 || absd == 9 || absd == 14 || absd == 18)) return false;

    // --- new robust row/col validation (add this) ---
    int fromRow = from / 8, toRow = to / 8;
    int rowDiff = fromRow > toRow ? fromRow - toRow : toRow - fromRow;
    int fromCol = from % 8, toCol = to % 8;
    int colDiff = fromCol > toCol ? fromCol - toCol : toCol - fromCol;

    if (absd == 7 || absd == 9) {
        if (rowDiff != 1 || colDiff != 1) return false;
    } else { // absd == 14 || absd == 18
        if (rowDiff != 2 || colDiff != 2) return false;
    }
    // --- end insertion ---

    // Edge/wrap checks (explicit by delta sign)
    if (delta == 7 || delta == 14) { // down-right
        if (IsRightEdge(from)) return false;
    } else if (delta == 9 || delta == 18) { // down-left
        if (IsLeftEdge(from)) return false;
    } else if (delta == -7 || delta == -14) { // up-left
        if (IsLeftEdge(from)) return false;
    } else if (delta == -9 || delta == -18) { // up-right
        if (IsRightEdge(from)) return false;
    }

    // mid for jumps
    int mid = (from + to) / 2;
    if ((absd == 14 || absd == 18) && !InBounds(mid)) return false;

    bool isKing = GetBit(*player_kings, from);

    // Non-king forward-only restriction
    if (!isKing) {
        if (playerId == 1 && delta <= 0) return false; // black must move to larger index (down)
        if (playerId == 2 && delta >= 0) return false; // red must move to smaller index (up)
    }

    // Simple move
    if (absd == 7 || absd == 9) {
        if (GetBit(*opponent, to)) return false; // can't move into opponent
        // perform move
        *player = ClearBit(*player, from);
        *player = SetBit(*player, to);
        if (isKing) {
            *player_kings = ClearBit(*player_kings, from);
            *player_kings = SetBit(*player_kings, to);
        }
        // Promotion
        if (playerId == 1 && to >= 56 && to <= 63) *player_kings = SetBit(*player_kings, to);
        if (playerId == 2 && to >= 0  && to <= 7)  *player_kings = SetBit(*player_kings, to);
        return true;
    }

    // Jump move
    if (absd == 14 || absd == 18) {
        if (!GetBit(*opponent, mid)) return false; // must jump over opponent piece
        if (GetBit(*player, to) || GetBit(*opponent, to)) return false; // destination must be empty

        // remove opponent piece
        *opponent = ClearBit(*opponent, mid);
        // remove opponent king if present
        if (GetBit(*opponent_kings, mid)) *opponent_kings = ClearBit(*opponent_kings, mid);

        // increment capture counter
        if (capture_counter) {
            (*capture_counter)++;
            if (playerId == 1)
                printf("Player 1 captured a Red piece!\n");
            else
                printf("Player 2 captured a Black piece!\n");
        }

        // move our piece
        *player = ClearBit(*player, from);
        *player = SetBit(*player, to);
        if (isKing) {
            *player_kings = ClearBit(*player_kings, from);
            *player_kings = SetBit(*player_kings, to);
        }
        // Promotion after jump
        if (playerId == 1 && to >= 56 && to <= 63) *player_kings = SetBit(*player_kings, to);
        if (playerId == 2 && to >= 0  && to <= 7)  *player_kings = SetBit(*player_kings, to);
        return true;
    }
    return false;
}

// Check whether a player has any legal move
bool HasAnyMove(unsigned long long player, unsigned long long opponent,
                unsigned long long player_kings, unsigned long long opponent_kings,
                int playerId) {
    for (int pos = 0; pos < 64; pos++) {
        if (!GetBit(player, pos)) continue;
        bool isKing = GetBit(player_kings, pos);
        int deltas_simple[4] = {7, 9, -7, -9};
        int deltas_jump[4]   = {14, 18, -14, -18};
        // simple
        for (int i = 0; i < 4; i++) {
            int d = deltas_simple[i];
            int to = pos + d;
            if (!InBounds(to)) continue;
            if (!isKing) {
                if (playerId == 1 && d <= 0) continue;
                if (playerId == 2 && d >= 0) continue;
            }
            // edge checks
            if ((d == 7  || d == -7)  && IsRightEdge(pos)) continue;
            if ((d == 9  || d == -9)  && IsLeftEdge(pos)) continue;
            if (!GetBit(player, to) && !GetBit(opponent, to)) return true;
        }
        // jumps
        for (int i = 0; i < 4; i++) {
            int d = deltas_jump[i];
            int to = pos + d;
            int mid = (pos + to) / 2;
            if (!InBounds(to) || !InBounds(mid)) continue;
            if (!isKing) {
                if (playerId == 1 && d <= 0) continue;
                if (playerId == 2 && d >= 0) continue;
            }
            if ((d == 14 || d == -14) && IsRightEdge(pos)) continue;
            if ((d == 18 || d == -18) && IsLeftEdge(pos)) continue;
            if (!GetBit(player, to) && GetBit(opponent, mid)) return true;
        }
    }
    return false;
}

// ================== Initial Setup ==================
unsigned long long create_player1(unsigned long long player) {
    // bottom rows for black (your original mapping)
    for (int n = 0; n <= 6; n += 2)         player = SetBit(player, n);
    for (int n = 9; n <= 15; n += 2)       player = SetBit(player, n);
    for (int n = 16; n <= 22; n += 2)      player = SetBit(player, n);
    return player;
}
unsigned long long create_player2(unsigned long long player) {
    for (int n = 57; n <= 63; n += 2)      player = SetBit(player, n);
    for (int n = 48; n <= 54; n += 2)      player = SetBit(player, n);
    for (int n = 41; n <= 47; n += 2)      player = SetBit(player, n);
    return player;
}
//===================  AI  ==================
// TryMove wrapper for AI: tries all moves, returns true if move made
bool AIMove(unsigned long long *ai, unsigned long long *human,
            unsigned long long *ai_kings, unsigned long long *human_kings,
            int aiId, int *ai_captures_count,
            int *fromPos, int *toPos) {
    /*
        Argument            Meaning
        &ai	                Address of AI’s bitboard (normal pieces)
        &human	            Address of the player’s bitboard
        &ai_kings	        Address of AI’s king pieces
        &human_kings	    Address of the player’s king pieces
        aiId	            Which side the AI represents (e.g., 1 or 2)
        &ai_captures_count	Address of AI’s capture counter
        &fromPos, &toPos	Output variables to record what move was made
    */
    int deltas_simple[4] = {7, 9, -7, -9};
    int deltas_jump[4]   = {14, 18, -14, -18};

    // (1) Try all capture moves first
    for (int pos = 0; pos < 64; pos++) {
        // Skip this square if AI doesn't have a piece here
        if (!GetBit(*ai, pos)) continue;
        // Check if this piece is a king (kings can move backward)
        bool isKing = GetBit(*ai_kings, pos);
        // Try every possible jump direction (4 diagonals)
        for (int i = 0; i < 4; i++) {
            int d = deltas_jump[i];
            int to = pos + d;
            // Skip if move goes out of board bounds
            if (!InBounds(to)) continue;
            // Make temporary copies of the board state for simulation
            unsigned long long tmp_ai = *ai;
            unsigned long long tmp_op = *human;
            unsigned long long tmp_ai_k = *ai_kings;
            unsigned long long tmp_op_k = *human_kings;
            int tmpCap = *ai_captures_count;
            // Attempt the move — TryMove() returns true if it's valid
            if (TryMove(&tmp_ai, &tmp_op, &tmp_ai_k, &tmp_op_k, pos, to, aiId, &tmpCap)) {
                // Move was valid — commit simulated state to actual game
                *ai = tmp_ai;
                *human = tmp_op;
                *ai_kings = tmp_ai_k;
                *human_kings = tmp_op_k;
                *ai_captures_count = tmpCap;
                // Record from/to positions for display or debugging
                *fromPos = pos;
                *toPos = to;
                // Return success — AI made a move
                return true;
            }
        }
    }

    // (2) Try all simple moves next
    for (int pos = 0; pos < 64; pos++) {
        // Skip this square if AI doesn't have a piece here
        if (!GetBit(*ai, pos)) continue;
        // Check if this piece is a king
        bool isKing = GetBit(*ai_kings, pos);
        // Try every simple diagonal move direction
        for (int i = 0; i < 4; i++) {
            int d = deltas_simple[i];
            int to = pos + d;
            // Skip invalid board positions
            if (!InBounds(to)) continue;
            // Copy state for simulation
            unsigned long long tmp_ai = *ai;
            unsigned long long tmp_op = *human;
            unsigned long long tmp_ai_k = *ai_kings;
            unsigned long long tmp_op_k = *human_kings;
            int tmpCap = *ai_captures_count;
            // Try to make a simple move
            if (TryMove(&tmp_ai, &tmp_op, &tmp_ai_k, &tmp_op_k, pos, to, aiId, &tmpCap)) {
                // Move was valid — commit to actual game state
                *ai = tmp_ai;//Update AI’s normal pieces after a move
                *human = tmp_op;//Update opponent’s normal pieces (possibly captured)
                *ai_kings = tmp_ai_k;//Update AI’s kings after a move or promotion
                *human_kings = tmp_op_k;//Updates the human’s king piece board, possibly removing a captured king.
                *ai_captures_count = tmpCap;//Update the AI’s total capture count
                // Record move info
                *fromPos = pos;//output — tells you which squares the AI moved from and to
                *toPos = to;
                return true;
            }
        }
    }
    // (3) No legal move found
    return false;
}
// ================== Main ==================
int main(void) {
    unsigned long long player1 = 0ULL;   // Black pieces
    unsigned long long player2 = 0ULL;   // Red pieces
    unsigned long long player1_kings = 0ULL;
    unsigned long long player2_kings = 0ULL;
    int red_captures_count = 0;   // how many blacks red has captured (show as R..)
    int black_captures_count = 0; // how many reds black has captured (show as B..)

    player1 = create_player1(player1);
    player2 = create_player2(player2);

    int turn = 1; // 1 = black (player1), 2 = red (player2)
    int num = 0;
    
    //Single Player or Multiplayer
    while (num != 1 && num != 2) {
        printf("Select Mode:\n");
        printf("1. Single Player\n");
        printf("2. Multiplayer\n");
        printf("Enter choice: ");
        scanf("%d", &num);
    
        if (num != 1 && num != 2) {
            printf("Invalid choice. Please enter 1 or 2.\n\n");
        }
    }
    
    if(num == 1)
    {
        while (1) {
        create_board(player1, player1_kings, player2, player2_kings,
                     red_captures_count, black_captures_count);

        // Check for win conditions
        if (player1 == 0ULL || !HasAnyMove(player1, player2, player1_kings, player2_kings, 1)) {
            printf("AI (Red) Wins!\n");
            break;
        }
        if (player2 == 0ULL || !HasAnyMove(player2, player1, player2_kings, player1_kings, 2)) {
            printf("You (Black) Win!\n");
            break;
        }

        int from, to;
        bool moved = false;

        if (turn == 1) {
            printf("Player 1's turn (Black). Enter move as: from, to  (Ex. from:22 -> to:31) (Only for Capture Ex. from:22 -> to:36)\n");
            if (scanf("%d %d", &from, &to) != 2) {
                printf("Invalid input. Exiting.\n");
                break;
            }
            moved = TryMove(&player1, &player2, &player1_kings, &player2_kings, from, to, 1, &black_captures_count);
            if (!moved) {
                printf("Illegal move. Try again.\n");
                continue;
            }
        } else {
            // Simple AI (Red)
            printf("AI (Red) is thinking...\n");
            #ifdef _WIN32
                Sleep(3000);  // 3 seconds (Sleep uses milliseconds on Windows)
            #endif

            int aiFrom = -1;
            int aiTo = -1;
        
            moved = AIMove(&player2, &player1, &player2_kings, &player1_kings, 2, &red_captures_count, &aiFrom, &aiTo);
            if (!moved || (player2 == 0ULL || !HasAnyMove(player2, player1, player2_kings, player1_kings, 2))) {
                printf("Player 1 Wins!\n");
                break;
            } 
            else if (!moved || (player1 == 0ULL || !HasAnyMove(player1, player2, player1_kings, player2_kings, 1))) {
                printf("Player 2 Wins!\n");
                break;
            }
            else {
                printf("AI moves from %d to %d\n", aiFrom, aiTo);
            }
        }
        // next turn
        turn = (turn == 1) ? 2 : 1;
        }
    }

    if(num == 2)
    {
        while (1) {
        create_board(player1, player1_kings, player2, player2_kings, red_captures_count, black_captures_count);

        // game over conditions
        if (player1 == 0ULL || !HasAnyMove(player1, player2, player1_kings, player2_kings, 1)) {
            printf("Player 2 (Red) Wins!\n");
            break;
        }
        if (player2 == 0ULL || !HasAnyMove(player2, player1, player2_kings, player1_kings, 2)) {
            printf("Player 1 (Black) Wins!\n");
            break;
        }

        printf("Player %d's turn (%s). Enter move as: from, to  (Ex. from:22 -> to:31) (Only for Capture Ex. from:22 -> to:36)\n",
               turn, (turn == 1) ? "Black (B)" : "Red (R)");
        int from, to;
        if (scanf("%d %d", &from, &to) != 2) {
            printf("Invalid input or EOF. Exiting.\n");
            break;
        }
        
        /*
         //or -1 -1 to quit
        if (from == -1 && to == -1) {
            printf("Quitting.\n");
            break;
        }
        */
        
        bool moved = false;
        if (turn == 1) {
            // black moves; if black captures increment black_captures_count
            
            moved = TryMove(&player1, &player2, &player1_kings, &player2_kings, from, to, 1, &black_captures_count);
        } else {
            // red moves; if red captures increment red_captures_count
            
            moved = TryMove(&player2, &player1, &player2_kings, &player1_kings, from, to, 2, &red_captures_count);
        }

        if (!moved) {
            printf("Illegal move. Try again.\n");
            continue; // same player's turn
        }

        // swap turns (we're not enforcing multi-jump continuation here)
        turn = (turn == 1) ? 2 : 1;
        }
    }
    printf("Game Over");
    #ifdef _WIN32
        Sleep(10000);  // 10 seconds (Sleep uses milliseconds on Windows)
    #endif
    return 0;
}