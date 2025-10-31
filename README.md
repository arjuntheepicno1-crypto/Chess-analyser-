# Chess-analyser-
It is a complete chess analysis app using pygame
#overview
This is a powerful, interactive Chess application developed in Python using the Pygame library for the graphical interface and the python-chess library for all core chess logic.

The game is designed to support:

Player vs. Player (PvP) local matches.

Player vs. Computer (PvC) matches via integration with an external UCI Chess Engine (like Stockfish).
To run this project, you need the following:

Python Environment
You must have Python 3.6 or newer installed.

Python Libraries
The following dependencies are required. Install them using pip:

pip install pygame python-chess
If you wish to play against the computer or use the analysis features, you will need a Universal Chess Interface (UCI) compatible engine.

Recommended Engine: Stockfish

Setup: Download the engine executable (e.g., stockfish.exe or stockfish) and either place it in the project's root directory or update the STOCKFISH_PATH variable within the chess.py file to point to its location
Install Dependencies: Run the installation command from the section above.

Run the Game: Execute the main script from your terminal:

python chess.py
Customizable Board: Adjustable board size, colors, and piece styles.

Intuitive Controls: Supports both drag-and-drop and click-to-click move input.

Visual Feedback: Highlights for legal moves and the previous move made.

Engine Integration: Communicates with UCI engines for strong AI opponents.

Move History: Tracks all moves, supporting PGN (Portable Game Notation) for game saving.
