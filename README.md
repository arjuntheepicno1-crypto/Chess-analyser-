import pygame
import chess
import chess.engine
import chess.pgn
import math
import os
from datetime import datetime
import time
import traceback
import random

# Initialize Pygame
pygame.init()

# Constants
BOARD_SIZE = 480
SQUARE_SIZE = BOARD_SIZE // 8
SIDEBAR_WIDTH = 200
WINDOW_WIDTH = BOARD_SIZE + SIDEBAR_WIDTH
WINDOW_HEIGHT = BOARD_SIZE + 100

# Colors
LIGHT_SQUARE = (240, 217, 181)
DARK_SQUARE = (181, 136, 99)
HIGHLIGHT_COLOR = (255, 255, 0, 128)
ARROW_COLOR = (255, 0, 0, 200)
GREEN_HIGHLIGHT = (0, 255, 0, 128)

class ChessGame:
    def __init__(self):
        self.screen = pygame.display.set_mode((WINDOW_WIDTH, WINDOW_HEIGHT))
        pygame.display.set_caption("Advanced Chess Game")
        
        # Game state
        self.board = chess.Board()
        self.move_history = []
        self.flipped = False
        self.game_mode = "pvp"
        self.move_mode = "drag"
        self.selected_square = None
        self.dragging_piece = None
        self.drag_pos = None
        self.arrows = []
        self.arrow_start = None
        self.analysis_mode = False
        self.show_best_move = False
        
        # Engine for analysis
        self.engine = None
        self.engine_path = None
        
        # AI opponent
        self.ai_engine = None
        self.ai_elo = None
        self.ai_color = chess.BLACK
        self.ai_weak = False
        
        # Evaluation
        self.evaluation = 0
        self.accuracy_scores = {"white": [], "black": []}
        
        # Clock
        self.clock_enabled = False
        self.white_time = 600
        self.black_time = 600
        self.clock_running = False
        self.last_time = time.time()
        
        # UI elements
        self.buttons = self.create_buttons()
        self.font = pygame.font.Font(None, 24)
        self.small_font = pygame.font.Font(None, 18)
        self.button_font = pygame.font.Font(None, 16)
        
        # Colors and styles
        self.board_colors = {
            "default": (LIGHT_SQUARE, DARK_SQUARE),
            "green": ((238, 238, 210), (118, 150, 86)),
            "blue": ((222, 227, 230), (140, 162, 173)),
            "brown": ((240, 217, 181), (181, 136, 99))
        }
        self.current_board_color = "default"
        
        self.piece_styles = ["default", "classic", "modern", "neo"]
        self.current_piece_style = "default"
        
        # Load piece images
        self.load_pieces()
        
    def create_buttons(self):
        button_y = BOARD_SIZE + 10
        button_height = 30
        button_width = 70
        spacing = 5  # Adjusted spacing for 7 buttons in second row
        
        buttons = []
        x = 10
        
        print(f"Creating buttons. Board size: {BOARD_SIZE}, Button area starts at y: {button_y}")
        
        row1_buttons = [
            {"text": "New Game", "action": "new_game"},
            {"text": "Analysis", "action": "toggle_analysis"},
            {"text": "Drag Mode", "action": "toggle_drag"},
            {"text": "Click Mode", "action": "toggle_click"},
            {"text": "Flip Board", "action": "flip_board"},
            {"text": "Best Move", "action": "toggle_best_move"}
        ]
        
        for i, btn_data in enumerate(row1_buttons):
            rect = pygame.Rect(x + i * (button_width + spacing), button_y, button_width, button_height)
            buttons.append({
                "rect": rect,
                "text": btn_data["text"],
                "action": btn_data["action"]
            })
            print(f"Button '{btn_data['text']}' created at: {rect}")
        
        button_y += button_height + 5
        row2_buttons = [
            {"text": "Board Color", "action": "change_board_color"},
            {"text": "Pieces", "action": "change_pieces"},
            {"text": "Load PGN", "action": "load_pgn"},
            {"text": "Save PGN", "action": "save_pgn"},
            {"text": "Clock", "action": "toggle_clock"},
            {"text": "Accuracy", "action": "show_accuracy"},
            {"text": "Undo", "action": "undo_move"},
            {"text": "Play AI", "action": "play_ai"}
        ]
        
        for i, btn_data in enumerate(row2_buttons):
            rect = pygame.Rect(x + i * (button_width + spacing), button_y, button_width, button_height)
            buttons.append({
                "rect": rect,
                "text": btn_data["text"],
                "action": btn_data["action"]
            })
            print(f"Button '{btn_data['text']}' created at: {rect}")
        
        print(f"Total buttons created: {len(buttons)}")
        return buttons
    
    def load_pieces(self):
        """Create piece representations for different styles"""
        self.piece_colors = {
            'white': (255, 255, 255),
            'black': (50, 50, 50)
        }
        
        self.piece_surfaces = {}
        piece_size = SQUARE_SIZE - 10
        
        for style in self.piece_styles:
            self.piece_surfaces[style] = {}
            for color in ['white', 'black']:
                self.piece_surfaces[style][color] = {}
                piece_color = self.piece_colors[color]
                outline_color = (0, 0, 0) if color == 'white' else (255, 255, 255)
                
                if style == "default":
                    # Pawn
                    pawn_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.circle(pawn_surface, piece_color, (piece_size//2, piece_size//3), piece_size//6)
                    pygame.draw.rect(pawn_surface, piece_color, (piece_size//2 - piece_size//8, piece_size//3, piece_size//4, piece_size//2))
                    pygame.draw.circle(pawn_surface, outline_color, (piece_size//2, piece_size//3), piece_size//6, 2)
                    pygame.draw.rect(pawn_surface, outline_color, (piece_size//2 - piece_size//8, piece_size//3, piece_size//4, piece_size//2), 2)
                    self.piece_surfaces[style][color]['p'] = pawn_surface
                    
                    # Rook
                    rook_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(rook_surface, piece_color, (piece_size//4, piece_size//4, piece_size//2, piece_size//2))
                    for i in range(3):
                        x = piece_size//4 + i * piece_size//6
                        pygame.draw.rect(rook_surface, piece_color, (x, piece_size//6, piece_size//12, piece_size//8))
                    pygame.draw.rect(rook_surface, outline_color, (piece_size//4, piece_size//4, piece_size//2, piece_size//2), 2)
                    self.piece_surfaces[style][color]['r'] = rook_surface
                    
                    # Knight
                    knight_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    points = [
                        (piece_size//3, piece_size//6),
                        (2*piece_size//3, piece_size//4),
                        (2*piece_size//3, piece_size//2),
                        (piece_size//2, 2*piece_size//3),
                        (piece_size//3, 2*piece_size//3),
                        (piece_size//4, piece_size//2)
                    ]
                    pygame.draw.polygon(knight_surface, piece_color, points)
                    pygame.draw.polygon(knight_surface, outline_color, points, 2)
                    pygame.draw.circle(knight_surface, outline_color, (piece_size//2, piece_size//3), 2)
                    self.piece_surfaces[style][color]['n'] = knight_surface
                    
                    # Bishop
                    bishop_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    points = [
                        (piece_size//2, piece_size//6),
                        (piece_size//3, 2*piece_size//3),
                        (2*piece_size//3, 2*piece_size//3)
                    ]
                    pygame.draw.polygon(bishop_surface, piece_color, points)
                    pygame.draw.polygon(bishop_surface, outline_color, points, 2)
                    pygame.draw.line(bishop_surface, outline_color, 
                                   (piece_size//2 - 3, piece_size//6), (piece_size//2 + 3, piece_size//6), 2)
                    pygame.draw.line(bishop_surface, outline_color, 
                                   (piece_size//2, piece_size//6 - 3), (piece_size//2, piece_size//6 + 3), 2)
                    self.piece_surfaces[style][color]['b'] = bishop_surface
                    
                    # Queen
                    queen_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(queen_surface, piece_color, (piece_size//4, piece_size//2, piece_size//2, piece_size//4))
                    for i in range(5):
                        x = piece_size//4 + i * piece_size//8
                        height = piece_size//6 if i % 2 == 1 else piece_size//4
                        pygame.draw.rect(queen_surface, piece_color, (x, piece_size//2 - height, piece_size//16, height))
                    pygame.draw.rect(queen_surface, outline_color, (piece_size//4, piece_size//2, piece_size//2, piece_size//4), 2)
                    self.piece_surfaces[style][color]['q'] = queen_surface
                    
                    # King
                    king_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(king_surface, piece_color, (piece_size//3, piece_size//3, piece_size//3, piece_size//3))
                    pygame.draw.line(king_surface, outline_color, 
                                   (piece_size//2 - 4, piece_size//4), (piece_size//2 + 4, piece_size//4), 3)
                    pygame.draw.line(king_surface, outline_color, 
                                   (piece_size//2, piece_size//4 - 4), (piece_size//2, piece_size//4 + 4), 3)
                    pygame.draw.rect(king_surface, outline_color, (piece_size//3, piece_size//3, piece_size//3, piece_size//3), 2)
                    self.piece_surfaces[style][color]['k'] = king_surface
                
                elif style == "classic":
                    # Classic style: slightly different shapes, more rounded
                    pawn_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.circle(pawn_surface, piece_color, (piece_size//2, piece_size//4), piece_size//5)
                    pygame.draw.ellipse(pawn_surface, piece_color, (piece_size//3, piece_size//3, piece_size//3, piece_size//2))
                    pygame.draw.circle(pawn_surface, outline_color, (piece_size//2, piece_size//4), piece_size//5, 2)
                    pygame.draw.ellipse(pawn_surface, outline_color, (piece_size//3, piece_size//3, piece_size//3, piece_size//2), 2)
                    self.piece_surfaces[style][color]['p'] = pawn_surface
                    
                    rook_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(rook_surface, piece_color, (piece_size//3, piece_size//4, piece_size//3, piece_size//2))
                    pygame.draw.rect(rook_surface, outline_color, (piece_size//3, piece_size//4, piece_size//3, piece_size//2), 2)
                    self.piece_surfaces[style][color]['r'] = rook_surface
                    
                    knight_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    points = [
                        (piece_size//3, piece_size//5),
                        (3*piece_size//4, piece_size//4),
                        (2*piece_size//3, 2*piece_size//3),
                        (piece_size//3, 2*piece_size//3)
                    ]
                    pygame.draw.polygon(knight_surface, piece_color, points)
                    pygame.draw.polygon(knight_surface, outline_color, points, 2)
                    self.piece_surfaces[style][color]['n'] = knight_surface
                    
                    bishop_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.ellipse(bishop_surface, piece_color, (piece_size//3, piece_size//4, piece_size//3, 2*piece_size//3))
                    pygame.draw.ellipse(bishop_surface, outline_color, (piece_size//3, piece_size//4, piece_size//3, 2*piece_size//3), 2)
                    self.piece_surfaces[style][color]['b'] = bishop_surface
                    
                    queen_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.circle(queen_surface, piece_color, (piece_size//2, piece_size//3), piece_size//4)
                    pygame.draw.rect(queen_surface, piece_color, (piece_size//3, piece_size//2, piece_size//3, piece_size//3))
                    pygame.draw.circle(queen_surface, outline_color, (piece_size//2, piece_size//3), piece_size//4, 2)
                    pygame.draw.rect(queen_surface, outline_color, (piece_size//3, piece_size//2, piece_size//3, piece_size//3), 2)
                    self.piece_surfaces[style][color]['q'] = queen_surface
                    
                    king_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(king_surface, piece_color, (piece_size//3, piece_size//3, piece_size//3, piece_size//2))
                    pygame.draw.circle(king_surface, piece_color, (piece_size//2, piece_size//4), piece_size//6)
                    pygame.draw.rect(king_surface, outline_color, (piece_size//3, piece_size//3, piece_size//3, piece_size//2), 2)
                    pygame.draw.circle(king_surface, outline_color, (piece_size//2, piece_size//4), piece_size//6, 2)
                    self.piece_surfaces[style][color]['k'] = king_surface
                
                elif style == "modern":
                    pawn_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(pawn_surface, piece_color, (piece_size//3, piece_size//3, piece_size//3, 2*piece_size//3))
                    pygame.draw.rect(pawn_surface, outline_color, (piece_size//3, piece_size//3, piece_size//3, 2*piece_size//3), 2)
                    self.piece_surfaces[style][color]['p'] = pawn_surface
                    
                    rook_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(rook_surface, piece_color, (piece_size//4, piece_size//4, piece_size//2, 2*piece_size//3))
                    pygame.draw.rect(rook_surface, outline_color, (piece_size//4, piece_size//4, piece_size//2, 2*piece_size//3), 2)
                    self.piece_surfaces[style][color]['r'] = rook_surface
                    
                    knight_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.polygon(knight_surface, piece_color, [
                        (piece_size//3, piece_size//4),
                        (2*piece_size//3, piece_size//4),
                        (piece_size//2, 3*piece_size//4)
                    ])
                    pygame.draw.polygon(knight_surface, outline_color, [
                        (piece_size//3, piece_size//4),
                        (2*piece_size//3, piece_size//4),
                        (piece_size//2, 3*piece_size//4)
                    ], 2)
                    self.piece_surfaces[style][color]['n'] = knight_surface
                    
                    bishop_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.polygon(bishop_surface, piece_color, [
                        (piece_size//2, piece_size//5),
                        (piece_size//3, 3*piece_size//4),
                        (2*piece_size//3, 3*piece_size//4)
                    ])
                    pygame.draw.polygon(bishop_surface, outline_color, [
                        (piece_size//2, piece_size//5),
                        (piece_size//3, 3*piece_size//4),
                        (2*piece_size//3, 3*piece_size//4)
                    ], 2)
                    self.piece_surfaces[style][color]['b'] = bishop_surface
                    
                    queen_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(queen_surface, piece_color, (piece_size//3, piece_size//3, piece_size//3, 2*piece_size//3))
                    pygame.draw.circle(queen_surface, piece_color, (piece_size//2, piece_size//4), piece_size//8)
                    pygame.draw.rect(queen_surface, outline_color, (piece_size//3, piece_size//3, piece_size//3, 2*piece_size//3), 2)
                    pygame.draw.circle(queen_surface, outline_color, (piece_size//2, piece_size//4), piece_size//8, 2)
                    self.piece_surfaces[style][color]['q'] = queen_surface
                    
                    king_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(king_surface, piece_color, (piece_size//3, piece_size//4, piece_size//3, 3*piece_size//4))
                    pygame.draw.rect(king_surface, outline_color, (piece_size//3, piece_size//4, piece_size//3, 3*piece_size//4), 2)
                    self.piece_surfaces[style][color]['k'] = king_surface
                
                elif style == "neo":
                    pawn_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.polygon(pawn_surface, piece_color, [
                        (piece_size//2, piece_size//4),
                        (piece_size//3, 3*piece_size//4),
                        (2*piece_size//3, 3*piece_size//4)
                    ])
                    pygame.draw.polygon(pawn_surface, outline_color, [
                        (piece_size//2, piece_size//4),
                        (piece_size//3, 3*piece_size//4),
                        (2*piece_size//3, 3*piece_size//4)
                    ], 2)
                    self.piece_surfaces[style][color]['p'] = pawn_surface
                    
                    rook_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(rook_surface, piece_color, (piece_size//4, piece_size//5, piece_size//2, 3*piece_size//4))
                    for i in range(3):
                        pygame.draw.rect(rook_surface, piece_color, 
                                       (piece_size//4 + i*piece_size//6, piece_size//8, piece_size//12, piece_size//6))
                    pygame.draw.rect(rook_surface, outline_color, (piece_size//4, piece_size//5, piece_size//2, 3*piece_size//4), 2)
                    self.piece_surfaces[style][color]['r'] = rook_surface
                    
                    knight_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    points = [
                        (piece_size//3, piece_size//5),
                        (3*piece_size//4, piece_size//3),
                        (2*piece_size//3, 3*piece_size//4),
                        (piece_size//3, 2*piece_size//3)
                    ]
                    pygame.draw.polygon(knight_surface, piece_color, points)
                    pygame.draw.polygon(knight_surface, outline_color, points, 2)
                    pygame.draw.circle(knight_surface, outline_color, (2*piece_size//3, piece_size//3), 3)
                    self.piece_surfaces[style][color]['n'] = knight_surface
                    
                    bishop_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.polygon(bishop_surface, piece_color, [
                        (piece_size//2, piece_size//6),
                        (piece_size//4, 3*piece_size//4),
                        (3*piece_size//4, 3*piece_size//4)
                    ])
                    pygame.draw.polygon(bishop_surface, outline_color, [
                        (piece_size//2, piece_size//6),
                        (piece_size//4, 3*piece_size//4),
                        (3*piece_size//4, 3*piece_size//4)
                    ], 2)
                    self.piece_surfaces[style][color]['b'] = bishop_surface
                    
                    queen_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.polygon(queen_surface, piece_color, [
                        (piece_size//2, piece_size//5),
                        (piece_size//3, 3*piece_size//4),
                        (2*piece_size//3, 3*piece_size//4)
                    ])
                    for i in range(3):
                        pygame.draw.rect(queen_surface, piece_color, 
                                       (piece_size//3 + i*piece_size//8, piece_size//6, piece_size//12, piece_size//5))
                    pygame.draw.polygon(queen_surface, outline_color, [
                        (piece_size//2, piece_size//5),
                        (piece_size//3, 3*piece_size//4),
                        (2*piece_size//3, 3*piece_size//4)
                    ], 2)
                    self.piece_surfaces[style][color]['q'] = queen_surface
                    
                    king_surface = pygame.Surface((piece_size, piece_size), pygame.SRCALPHA)
                    pygame.draw.rect(king_surface, piece_color, (piece_size//3, piece_size//4, piece_size//3, 2*piece_size//3))
                    pygame.draw.polygon(king_surface, piece_color, [
                        (piece_size//2, piece_size//5),
                        (piece_size//3, piece_size//3),
                        (2*piece_size//3, piece_size//3)
                    ])
                    pygame.draw.rect(king_surface, outline_color, (piece_size//3, piece_size//4, piece_size//3, 2*piece_size//3), 2)
                    pygame.draw.polygon(king_surface, outline_color, [
                        (piece_size//2, piece_size//5),
                        (piece_size//3, piece_size//3),
                        (2*piece_size//3, piece_size//3)
                    ], 2)
                    self.piece_surfaces[style][color]['k'] = king_surface
    
    def setup_engine(self):
        """Setup chess engine for analysis"""
        try:
            engine_paths = [
                "stockfish",
                "/usr/local/bin/stockfish",
                "/usr/bin/stockfish",
                "/opt/homebrew/bin/stockfish",
                "C:\\Program Files\\Stockfish\\stockfish.exe",
                "stockfish.exe"
            ]
            
            for path in engine_paths:
                try:
                    self.engine = chess.engine.SimpleEngine.popen_uci(path)
                    self.engine_path = path
                    print(f"Engine loaded: {path}")
                    return True
                except:
                    continue
                    
        except Exception as e:
            print(f"Engine setup failed: {e}")
        
        print("No chess engine found. Analysis features disabled.")
        self.engine = None
        return False
    
    def get_best_move(self):
        """Get best move from engine"""
        if not self.engine:
            return None
        
        try:
            result = self.engine.analyse(self.board, chess.engine.Limit(time=0.1))
            return result.get("pv", [None])[0] if result.get("pv") else None
        except Exception as e:
            print(f"Error getting best move: {e}")
            return None
    
    def get_evaluation(self):
        """Get position evaluation from engine"""
        if not self.engine:
            return 0
        
        try:
            result = self.engine.analyse(self.board, chess.engine.Limit(time=0.1))
            score = result.get("score")
            if score:
                if score.is_mate():
                    return 1000 if score.white().mate() > 0 else -1000
                else:
                    return score.white().score() / 100 if score.white().score() else 0
        except Exception as e:
            print(f"Error getting evaluation: {e}")
        return 0
    
    def square_to_coords(self, square):
        """Convert chess square to screen coordinates"""
        file = chess.square_file(square)
        rank = chess.square_rank(square)
        
        if self.flipped:
            x = (7 - file) * SQUARE_SIZE
            y = rank * SQUARE_SIZE
        else:
            x = file * SQUARE_SIZE
            y = (7 - rank) * SQUARE_SIZE
        
        return x, y
    
    def coords_to_square(self, x, y):
        """Convert screen coordinates to chess square"""
        if x < 0 or x >= BOARD_SIZE or y < 0 or y >= BOARD_SIZE:
            return None
        
        file = x // SQUARE_SIZE
        rank = 7 - (y // SQUARE_SIZE)
        
        if self.flipped:
            file = 7 - file
            rank = 7 - rank
        
        return chess.square(file, rank)
    
    def draw_board(self):
        """Draw the chess board"""
        light_color, dark_color = self.board_colors[self.current_board_color]
        
        for rank in range(8):
            for file in range(8):
                color = light_color if (rank + file) % 2 == 0 else dark_color
                rect = pygame.Rect(file * SQUARE_SIZE, rank * SQUARE_SIZE, SQUARE_SIZE, SQUARE_SIZE)
                pygame.draw.rect(self.screen, color, rect)
                
                if file == 0:
                    rank_num = 8 - rank if not self.flipped else rank + 1
                    text = self.small_font.render(str(rank_num), True, (100, 100, 100))
                    self.screen.blit(text, (5, rank * SQUARE_SIZE + 5))
                
                if rank == 7:
                    file_letter = chr(ord('a') + file) if not self.flipped else chr(ord('h') - file)
                    text = self.small_font.render(file_letter, True, (100, 100, 100))
                    self.screen.blit(text, (file * SQUARE_SIZE + SQUARE_SIZE - 15, BOARD_SIZE - 20))
    
    def draw_pieces(self):
        """Draw chess pieces using the current style"""
        for square in chess.SQUARES:
            piece = self.board.piece_at(square)
            if piece and (not self.dragging_piece or square != self.selected_square):
                x, y = self.square_to_coords(square)
                
                color = 'white' if piece.color == chess.WHITE else 'black'
                piece_type = piece.symbol().lower()
                
                if (self.current_piece_style in self.piece_surfaces and 
                    color in self.piece_surfaces[self.current_piece_style] and 
                    piece_type in self.piece_surfaces[self.current_piece_style][color]):
                    piece_surface = self.piece_surfaces[self.current_piece_style][color][piece_type]
                    piece_rect = piece_surface.get_rect()
                    piece_rect.center = (x + SQUARE_SIZE // 2, y + SQUARE_SIZE // 2)
                    self.screen.blit(piece_surface, piece_rect)
    
    def draw_highlights(self):
        """Draw square highlights"""
        if self.selected_square is not None:
            x, y = self.square_to_coords(self.selected_square)
            highlight_surface = pygame.Surface((SQUARE_SIZE, SQUARE_SIZE), pygame.SRCALPHA)
            highlight_surface.fill(HIGHLIGHT_COLOR)
            self.screen.blit(highlight_surface, (x, y))
            
            for move in self.board.legal_moves:
                if move.from_square == self.selected_square:
                    x, y = self.square_to_coords(move.to_square)
                    highlight_surface = pygame.Surface((SQUARE_SIZE, SQUARE_SIZE), pygame.SRCALPHA)
                    highlight_surface.fill(GREEN_HIGHLIGHT)
                    self.screen.blit(highlight_surface, (x, y))
                    
                    center = (x + SQUARE_SIZE // 2, y + SQUARE_SIZE // 2)
                    if self.board.piece_at(move.to_square):
                        pygame.draw.circle(self.screen, (0, 255, 0), center, SQUARE_SIZE // 3, 3)
                    else:
                        pygame.draw.circle(self.screen, (0, 255, 0), center, 8)
    
    def draw_arrows(self):
        """Draw analysis arrows"""
        for arrow in self.arrows:
            from_x, from_y = self.square_to_coords(arrow['from'])
            to_x, to_y = self.square_to_coords(arrow['to'])
            
            from_center = (from_x + SQUARE_SIZE // 2, from_y + SQUARE_SIZE // 2)
            to_center = (to_x + SQUARE_SIZE // 2, to_y + SQUARE_SIZE // 2)
            
            dx = to_center[0] - from_center[0]
            dy = to_center[1] - from_center[1]
            length = math.sqrt(dx*dx + dy*dy)
            
            if length > 0:
                dx /= length
                dy /= length
                
                margin = SQUARE_SIZE // 4
                arrow_start = (from_center[0] + dx * margin, from_center[1] + dy * margin)
                arrow_end = (to_center[0] - dx * margin, to_center[1] - dy * margin)
                
                pygame.draw.line(self.screen, ARROW_COLOR[:3], arrow_start, arrow_end, 8)
                
                head_length = 20
                head_width = 12
                
                p1 = (arrow_end[0] - head_length * dx + head_width * dy,
                      arrow_end[1] - head_length * dy - head_width * dx)
                p2 = (arrow_end[0] - head_length * dx - head_width * dy,
                      arrow_end[1] - head_length * dy + head_width * dx)
                
                pygame.draw.polygon(self.screen, ARROW_COLOR[:3], [arrow_end, p1, p2])
    
    def draw_evaluation_bar(self):
        """Draw evaluation bar"""
        if not self.engine:
            return
            
        bar_width = 30
        bar_height = 200
        bar_x = BOARD_SIZE + 15
        bar_y = 20
        
        pygame.draw.rect(self.screen, (200, 200, 200), (bar_x, bar_y, bar_width, bar_height))
        pygame.draw.rect(self.screen, (0, 0, 0), (bar_x, bar_y, bar_width, bar_height), 2)
        
        eval_normalized = max(-10, min(10, self.evaluation)) / 10.0
        center_y = bar_y + bar_height // 2
        
        if eval_normalized > 0:
            white_height = int(eval_normalized * (bar_height // 2))
            pygame.draw.rect(self.screen, (255, 255, 255), 
                           (bar_x + 1, center_y - white_height, bar_width - 2, white_height))
        elif eval_normalized < 0:
            black_height = int(-eval_normalized * (bar_height // 2))
            pygame.draw.rect(self.screen, (50, 50, 50), 
                           (bar_x + 1, center_y, bar_width - 2, black_height))
        
        pygame.draw.line(self.screen, (100, 100, 100), 
                        (bar_x, center_y), (bar_x + bar_width, center_y), 2)
        
        eval_text = f"{self.evaluation:.1f}"
        text = self.small_font.render(eval_text, True, (0, 0, 0))
        text_rect = text.get_rect()
        self.screen.blit(text, (bar_x + bar_width + 5, center_y - text_rect.height // 2))
        
        label = self.small_font.render("Eval", True, (0, 0, 0))
        self.screen.blit(label, (bar_x, bar_y - 15))
    
    def draw_clock(self):
        """Draw game clock"""
        if not self.clock_enabled:
            return
        
        clock_x = BOARD_SIZE + 15
        clock_y = 250
        
        white_minutes = max(0, int(self.white_time // 60))
        white_seconds = max(0, int(self.white_time % 60))
        white_text = f"White: {white_minutes:02d}:{white_seconds:02d}"
        color = (255, 0, 0) if self.board.turn == chess.WHITE and self.clock_running else (0, 0, 0)
        text = self.font.render(white_text, True, color)
        self.screen.blit(text, (clock_x, clock_y))
        
        black_minutes = max(0, int(self.black_time // 60))
        black_seconds = max(0, int(self.black_time % 60))
        black_text = f"Black: {black_minutes:02d}:{black_seconds:02d}"
        color = (255, 0, 0) if self.board.turn == chess.BLACK and self.clock_running else (0, 0, 0)
        text = self.font.render(black_text, True, color)
        self.screen.blit(text, (clock_x, clock_y + 25))
    
    def draw_buttons(self):
        """Draw UI buttons"""
        for button in self.buttons:
            color = (200, 200, 200)
            
            if button["action"] == "toggle_analysis" and self.analysis_mode:
                color = (150, 255, 150)
            elif button["action"] == "toggle_drag" and self.move_mode == "drag":
                color = (150, 255, 150)
            elif button["action"] == "toggle_click" and self.move_mode == "click":
                color = (150, 255, 150)
            elif button["action"] == "toggle_best_move" and self.show_best_move:
                color = (150, 255, 150)
            elif button["action"] == "toggle_clock" and self.clock_enabled:
                color = (150, 255, 150)
            elif button["action"] == "play_ai" and self.game_mode == "pvc":
                color = (150, 255, 150)
            
            pygame.draw.rect(self.screen, color, button["rect"])
            pygame.draw.rect(self.screen, (0, 0, 0), button["rect"], 2)
            
            text = self.button_font.render(button["text"], True, (0, 0, 0))
            text_rect = text.get_rect(center=button["rect"].center)
            self.screen.blit(text, text_rect)
    
    def draw_best_move_hint(self):
        """Draw best move hint"""
        if self.show_best_move and self.engine:
            best_move = self.get_best_move()
            if best_move:
                from_x, from_y = self.square_to_coords(best_move.from_square)
                to_x, to_y = self.square_to_coords(best_move.to_square)
                
                from_center = (from_x + SQUARE_SIZE // 2, from_y + SQUARE_SIZE // 2)
                to_center = (to_x + SQUARE_SIZE // 2, to_y + SQUARE_SIZE // 2)
                
                dx = to_center[0] - from_center[0]
                dy = to_center[1] - from_center[1]
                length = math.sqrt(dx*dx + dy*dy)
                
                if length > 0:
                    dx /= length
                    dy /= length
                    
                    margin = SQUARE_SIZE // 4
                    arrow_start = (from_center[0] + dx * margin, from_center[1] + dy * margin)
                    arrow_end = (to_center[0] - dx * margin, to_center[1] - dy * margin)
                    
                    pygame.draw.line(self.screen, (0, 255, 0), arrow_start, arrow_end, 6)
                    
                    head_length = 15
                    head_width = 8
                    
                    p1 = (arrow_end[0] - head_length * dx + head_width * dy,
                          arrow_end[1] - head_length * dy - head_width * dx)
                    p2 = (arrow_end[0] - head_length * dx - head_width * dy,
                          arrow_end[1] - head_length * dy + head_width * dx)
                    
                    pygame.draw.polygon(self.screen, (0, 255, 0), [arrow_end, p1, p2])
    
    def update_clock(self):
        """Update game clock"""
        if not self.clock_enabled or not self.clock_running:
            return
        
        current_time = time.time()
        elapsed = current_time - self.last_time
        self.last_time = current_time
        
        if self.board.turn == chess.WHITE:
            self.white_time -= elapsed
            if self.white_time <= 0:
                self.white_time = 0
                self.clock_running = False
                print("White time expired!")
        else:
            self.black_time -= elapsed
            if self.black_time <= 0:
                self.black_time = 0
                self.clock_running = False
                print("Black time expired!")
    
    def undo_move(self):
        """Undo the last move"""
        try:
            if not self.move_history:
                print("No moves to undo")
                return
            
            last_move = self.board.pop()
            self.move_history.pop()
            
            # Adjust accuracy scores
            if self.accuracy_scores["white"] or self.accuracy_scores["black"]:
                if self.board.turn == chess.BLACK:
                    if self.accuracy_scores["white"]:
                        self.accuracy_scores["white"].pop()
                else:
                    if self.accuracy_scores["black"]:
                        self.accuracy_scores["black"].pop()
            
            if self.engine:
                self.evaluation = self.get_evaluation()
            
            if self.clock_enabled:
                self.clock_running = True
                self.last_time = time.time()
            
            self.selected_square = None
            self.dragging_piece = None
            print(f"Move undone: {last_move}")
        except Exception as e:
            print(f"Error undoing move: {e}")
            traceback.print_exc()
    
    def handle_button_click(self, pos):
        """Handle button clicks"""
        try:
            for button in self.buttons:
                if button["rect"].collidepoint(pos):
                    action = button["action"]
                    print(f"Button clicked: {action}")
                    
                    if action == "new_game":
                        self.board = chess.Board()
                        self.move_history = []
                        self.selected_square = None
                        self.arrows = []
                        self.accuracy_scores = {"white": [], "black": []}
                        self.white_time = 600
                        self.black_time = 600
                        if self.engine:
                            self.evaluation = self.get_evaluation()
                        print("New game started")
                        
                    elif action == "toggle_analysis":
                        self.analysis_mode = not self.analysis_mode
                        if self.analysis_mode and not self.engine:
                            self.setup_engine()
                        print(f"Analysis mode: {self.analysis_mode}")
                        
                    elif action == "toggle_drag":
                        self.move_mode = "drag"
                        self.selected_square = None
                        print("Drag mode activated")
                        
                    elif action == "toggle_click":
                        self.move_mode = "click"
                        self.selected_square = None
                        print("Click mode activated")
                        
                    elif action == "flip_board":
                        self.flipped = not self.flipped
                        print(f"Board flipped: {self.flipped}")
                        
                    elif action == "toggle_best_move":
                        self.show_best_move = not self.show_best_move
                        if self.show_best_move and not self.engine:
                            self.setup_engine()
                        print(f"Best move hint: {self.show_best_move}")
                        
                    elif action == "change_board_color":
                        colors = list(self.board_colors.keys())
                        current_idx = colors.index(self.current_board_color)
                        self.current_board_color = colors[(current_idx + 1) % len(colors)]
                        print(f"Board color changed to: {self.current_board_color}")
                        
                    elif action == "change_pieces":
                        styles = self.piece_styles
                        current_idx = styles.index(self.current_piece_style)
                        self.current_piece_style = styles[(current_idx + 1) % len(styles)]
                        print(f"Piece style changed to: {self.current_piece_style}")
                        
                    elif action == "load_pgn":
                        self.load_pgn_file()
                        
                    elif action == "save_pgn":
                        self.save_pgn_file()
                        
                    elif action == "toggle_clock":
                        self.clock_enabled = not self.clock_enabled
                        if self.clock_enabled:
                            self.clock_running = True
                            self.last_time = time.time()
                        else:
                            self.clock_running = False
                        print(f"Clock enabled: {self.clock_enabled}")
                        
                    elif action == "show_accuracy":
                        self.show_accuracy_analysis()
                    
                    elif action == "undo_move":
                        self.undo_move()
                    
                    elif action == "play_ai":
                        self.toggle_ai_mode()
                    
                    return True
            return False
        except Exception as e:
            print(f"Error handling button click: {e}")
            traceback.print_exc()
            return False
    
    def toggle_ai_mode(self):
        if self.game_mode == "pvc":
            self.game_mode = "pvp"
            print("Switched to Player vs Player mode")
            if self.ai_engine:
                try:
                    self.ai_engine.quit()
                except:
                    pass
                self.ai_engine = None
            self.ai_elo = None
            self.ai_weak = False
        else:
            if not self.engine_path:
                if not self.setup_engine():
                    print("Cannot start AI mode without chess engine")
                    return
            print("Enter desired ELO for computer opponent (800-3500): ")
            try:
                elo = int(input())
                elo = max(800, min(3500, elo))
            except:
                elo = 2000
                print("Invalid input, using default 2000 ELO")
            
            print("Choose AI color: white or black? ")
            color_input = input().lower()
            if color_input == 'white':
                self.ai_color = chess.WHITE
            else:
                self.ai_color = chess.BLACK
            
            # Create AI engine
            self.ai_weak = False
            if elo < 1320:
                self.ai_weak = True
                self.ai_engine = None
                print("Using random move AI for beginner level")
            else:
                try:
                    self.ai_engine = chess.engine.SimpleEngine.popen_uci(self.engine_path)
                except Exception as e:
                    print(f"Failed to create AI engine: {e}")
                    return
                
                # Configure AI strength
                try:
                    if elo > 3190:
                        self.ai_engine.configure({"UCI_LimitStrength": False})
                    else:
                        self.ai_engine.configure({
                            "UCI_LimitStrength": True,
                            "UCI_Elo": elo
                        })
                except Exception as e:
                    print(f"Failed to configure AI: {e}")
                    self.ai_engine.quit()
                    self.ai_engine = None
                    return
            
            self.ai_elo = elo
            self.game_mode = "pvc"
            print("Switched to Player vs Computer mode")
        
        # Reset game for new mode
        self.board = chess.Board()
        self.move_history = []
        self.selected_square = None
        self.arrows = []
        self.accuracy_scores = {"white": [], "black": []}
        self.white_time = 600
        self.black_time = 600
        if self.clock_enabled:
            self.clock_running = True
            self.last_time = time.time()
        if self.engine:
            self.evaluation = self.get_evaluation()
    
    def handle_ai_move(self):
        """Let AI make a move if it's their turn"""
        if self.game_mode != "pvc" or self.board.turn != self.ai_color or self.board.is_game_over():
            return
        
        try:
            if self.ai_weak:
                legal_moves = list(self.board.legal_moves)
                if legal_moves:
                    move = random.choice(legal_moves)
                    self.board.push(move)
                    self.move_history.append(move)
                    if self.engine:
                        self.evaluation = self.get_evaluation()
                    if self.clock_enabled:
                        self.last_time = time.time()
                    print(f"AI (random) move: {move}")
            else:
                # AI thinking time - small to avoid lag
                limit = chess.engine.Limit(time=0.5)
                result = self.ai_engine.play(self.board, limit)
                move = result.move
                if move:
                    self.board.push(move)
                    self.move_history.append(move)
                    if self.engine:
                        self.evaluation = self.get_evaluation()
                    if self.clock_enabled:
                        self.last_time = time.time()
                    print(f"AI move: {move}")
        except Exception as e:
            print(f"AI move error: {e}")
            traceback.print_exc()
    
    def handle_mouse_down(self, pos, button):
        """Handle mouse down events"""
        try:
            print(f"Mouse down at: {pos}, button: {button}")
            
            if pos[1] >= BOARD_SIZE:
                print("Click detected in button area")
                return self.handle_button_click(pos)
            
            square = self.coords_to_square(pos[0], pos[1])
            if square is None:
                return False
            
            if button == 1:
                if self.game_mode == "pvc" and self.board.turn == self.ai_color:
                    return False  # AI's turn, no input
                
                if self.move_mode == "drag":
                    piece = self.board.piece_at(square)
                    if piece and piece.color == self.board.turn:
                        self.selected_square = square
                        self.dragging_piece = piece
                        self.drag_pos = pos
                elif self.move_mode == "click":
                    if self.selected_square is None:
                        piece = self.board.piece_at(square)
                        if piece and piece.color == self.board.turn:
                            self.selected_square = square
                    else:
                        if square == self.selected_square:
                            self.selected_square = None
                        else:
                            self.try_move(self.selected_square, square)
                            self.selected_square = None
            
            elif button == 3 and self.analysis_mode:
                self.arrow_start = square
            
            return True
        except Exception as e:
            print(f"Error handling mouse down: {e}")
            traceback.print_exc()
            return False
    
    def handle_mouse_up(self, pos, button):
        """Handle mouse up events"""
        try:
            if button == 1 and self.move_mode == "drag" and self.dragging_piece:
                square = self.coords_to_square(pos[0], pos[1])
                if square is not None and self.selected_square is not None:
                    self.try_move(self.selected_square, square)
                
                self.dragging_piece = None
                self.selected_square = None
                self.drag_pos = None
            
            elif button == 3 and self.analysis_mode and self.arrow_start is not None:
                square = self.coords_to_square(pos[0], pos[1])
                if square is not None and square != self.arrow_start:
                    arrow = {'from': self.arrow_start, 'to': square}
                    if arrow in self.arrows:
                        self.arrows.remove(arrow)
                    else:
                        self.arrows.append(arrow)
                    print(f"Arrow {'removed' if arrow not in self.arrows else 'added'}: {chess.square_name(arrow['from'])} -> {chess.square_name(arrow['to'])}")
                self.arrow_start = None
        except Exception as e:
            print(f"Error handling mouse up: {e}")
            traceback.print_exc()
    
    def try_move(self, from_square, to_square):
        """Try to make a move"""
        try:
            move = chess.Move(from_square, to_square)
            
            if (self.board.piece_at(from_square) and
                self.board.piece_at(from_square).piece_type == chess.PAWN and 
                (chess.square_rank(to_square) == 7 or chess.square_rank(to_square) == 0)):
                move = chess.Move(from_square, to_square, promotion=chess.QUEEN)
            
            if move in self.board.legal_moves:
                if self.engine:
                    best_move = self.get_best_move()
                    if best_move:
                        accuracy = 100 if move == best_move else max(0, 100 - abs(self.get_move_centipawn_loss(move)))
                        if self.board.turn == chess.WHITE:
                            self.accuracy_scores["white"].append(accuracy)
                        else:
                            self.accuracy_scores["black"].append(accuracy)
                
                self.move_history.append(move)
                self.board.push(move)
                
                if self.engine:
                    self.evaluation = self.get_evaluation()
                
                if self.clock_enabled:
                    self.clock_running = True
                    self.last_time = time.time()
                
                print(f"Move made: {move}")
                return True
            return False
        except Exception as e:
            print(f"Move error: {e}")
            traceback.print_exc()
            return False
    
    def get_move_centipawn_loss(self, move):
        """Calculate centipawn loss for a move"""
        if not self.engine:
            return 0
        
        try:
            eval_before = self.get_evaluation()
            self.board.push(move)
            eval_after = -self.get_evaluation()
            self.board.pop()
            loss = max(0, eval_before - eval_after) * 100
            return min(loss, 500)
        except Exception as e:
            print(f"Error calculating centipawn loss: {e}")
            return 50
    
    def load_pgn_file(self):
        """Load game from PGN file"""
        try:
            if os.path.exists("game.pgn"):
                with open("game.pgn", "r") as f:
                    game = chess.pgn.read_game(f)
                    if game:
                        self.board = chess.Board()
                        self.move_history = []
                        self.accuracy_scores = {"white": [], "black": []}
                        
                        for move in game.mainline_moves():
                            self.move_history.append(move)
                            self.board.push(move)
                        
                        if self.engine:
                            self.evaluation = self.get_evaluation()
                        
                        print("PGN loaded successfully")
                        return True
            else:
                print("No game.pgn file found")
        except Exception as e:
            print(f"Error loading PGN: {e}")
            traceback.print_exc()
        return False
    
    def save_pgn_file(self):
        """Save game to PGN file"""
        try:
            game = chess.pgn.Game()
            game.headers["Event"] = "Chess Game"
            game.headers["Site"] = "Python Chess"
            game.headers["Date"] = datetime.now().strftime("%Y.%m.%d")
            game.headers["Round"] = "1"
            game.headers["White"] = "Player 1" if self.game_mode == "pvp" else ("Player" if self.ai_color == chess.BLACK else "AI")
            game.headers["Black"] = "Player 2" if self.game_mode == "pvp" else ("AI" if self.ai_color == chess.BLACK else "Player")
            game.headers["Result"] = "*"
            
            node = game
            board = chess.Board()
            
            for move in self.move_history:
                node = node.add_variation(move)
                board.push(move)
            
            if board.is_game_over():
                result = board.result()
                game.headers["Result"] = result
            
            with open("game.pgn", "w") as f:
                print(game, file=f)
            print("PGN saved successfully to game.pgn")
            return True
        except Exception as e:
            print(f"Error saving PGN: {e}")
            traceback.print_exc()
            return False
    
    def show_accuracy_analysis(self):
        """Show accuracy analysis"""
        try:
            if self.accuracy_scores["white"] or self.accuracy_scores["black"]:
                white_avg = sum(self.accuracy_scores["white"]) / len(self.accuracy_scores["white"]) if self.accuracy_scores["white"] else 0
                black_avg = sum(self.accuracy_scores["black"]) / len(self.accuracy_scores["black"]) if self.accuracy_scores["black"] else 0
                
                white_moves = len(self.accuracy_scores["white"])
                black_moves = len(self.accuracy_scores["black"])
                
                print(f"\n=== ACCURACY ANALYSIS ===")
                print(f"White: {white_avg:.1f}% average ({white_moves} moves)")
                print(f"Black: {black_avg:.1f}% average ({black_moves} moves)")
                
                if white_moves > 0:
                    print(f"White move accuracies: {[f'{acc:.0f}%' for acc in self.accuracy_scores['white'][-5:]]}")
                if black_moves > 0:
                    print(f"Black move accuracies: {[f'{acc:.0f}%' for acc in self.accuracy_scores['black'][-5:]]}")
                print("=========================\n")
            else:
                print("No accuracy data available (requires chess engine)")
        except Exception as e:
            print(f"Error in accuracy analysis: {e}")
            traceback.print_exc()
    
    def draw_game_info(self):
        """Draw game information"""
        info_x = BOARD_SIZE + 15
        info_y = 320
        
        try:
            if self.board.is_checkmate():
                winner = "Black" if self.board.turn == chess.WHITE else "White"
                status_text = f"Checkmate! {winner} wins"
                color = (255, 0, 0)
            elif self.board.is_stalemate():
                status_text = "Stalemate - Draw"
                color = (255, 165, 0)
            elif self.board.is_check():
                status_text = "Check!"
                color = (255, 0, 0)
            else:
                turn = "White" if self.board.turn == chess.WHITE else "Black"
                status_text = f"{turn} to move"
                color = (0, 0, 0)
            
            text = self.font.render(status_text, True, color)
            self.screen.blit(text, (info_x, info_y))
            
            move_count = len(self.move_history)
            move_text = f"Moves: {move_count}"
            text = self.small_font.render(move_text, True, (0, 0, 0))
            self.screen.blit(text, (info_x, info_y + 25))
            
            if self.move_history:
                last_move = self.move_history[-1]
                last_move_text = f"Last: {last_move}"
                text = self.small_font.render(last_move_text, True, (0, 0, 0))
                self.screen.blit(text, (info_x, info_y + 45))
            
            if self.analysis_mode:
                analysis_text = "Analysis Mode: ON"
                text = self.small_font.render(analysis_text, True, (0, 150, 0))
                self.screen.blit(text, (info_x, info_y + 65))
                
                arrows_text = f"Arrows: {len(self.arrows)}"
                text = self.small_font.render(arrows_text, True, (0, 0, 0))
                self.screen.blit(text, (info_x, info_y + 85))
            
            if self.game_mode == "pvc":
                ai_text = f"AI ELO: {self.ai_elo}"
                text = self.small_font.render(ai_text, True, (0, 0, 150))
                self.screen.blit(text, (info_x, info_y + 105))
        except Exception as e:
            print(f"Error drawing game info: {e}")
            traceback.print_exc()
    
    def run(self):
        """Main game loop"""
        clock = pygame.time.Clock()
        running = True
        
        engine_loaded = self.setup_engine()
        if engine_loaded and self.engine:
            self.evaluation = self.get_evaluation()
        
        print("Chess game started!")
        print("Controls:")
        print("- Left click to select/move pieces")
        print("- Right click (in analysis mode) to draw arrows")
        print("- Use buttons to change settings")
        print("- Hotkeys: N=new game, F=flip board, A=analysis mode")
        
        while running:
            try:
                for event in pygame.event.get():
                    if event.type == pygame.QUIT:
                        running = False
                    
                    elif event.type == pygame.MOUSEBUTTONDOWN:
                        self.handle_mouse_down(event.pos, event.button)
                    
                    elif event.type == pygame.MOUSEBUTTONUP:
                        self.handle_mouse_up(event.pos, event.button)
                    
                    elif event.type == pygame.MOUSEMOTION:
                        if self.dragging_piece:
                            self.drag_pos = event.pos
                    
                    elif event.type == pygame.KEYDOWN:
                        if event.key == pygame.K_n:
                            self.board = chess.Board()
                            self.move_history = []
                            self.selected_square = None
                            self.arrows = []
                            self.accuracy_scores = {"white": [], "black": []}
                            if self.engine:
                                self.evaluation = self.get_evaluation()
                            print("New game started (hotkey)")
                        elif event.key == pygame.K_f:
                            self.flipped = not self.flipped
                            print("Board flipped (hotkey)")
                        elif event.key == pygame.K_a:
                            self.analysis_mode = not self.analysis_mode
                            if self.analysis_mode and not self.engine:
                                self.setup_engine()
                            print(f"Analysis mode: {self.analysis_mode} (hotkey)")
                        elif event.key == pygame.K_c:
                            self.arrows = []
                            print("Arrows cleared (hotkey)")
                
                self.update_clock()
                self.handle_ai_move()
                
                self.screen.fill((255, 255, 255))
                self.draw_board()
                self.draw_highlights()
                self.draw_pieces()
                
                if self.analysis_mode:
                    self.draw_arrows()
                
                if self.show_best_move:
                    self.draw_best_move_hint()
                
                if self.engine:
                    self.evaluation = self.get_evaluation()
                    self.draw_evaluation_bar()
                
                self.draw_clock()
                self.draw_buttons()
                self.draw_game_info()
                
                if self.dragging_piece and self.drag_pos:
                    color = 'white' if self.dragging_piece.color == chess.WHITE else 'black'
                    piece_type = self.dragging_piece.symbol().lower()
                    
                    if (self.current_piece_style in self.piece_surfaces and 
                        color in self.piece_surfaces[self.current_piece_style] and 
                        piece_type in self.piece_surfaces[self.current_piece_style][color]):
                        piece_surface = self.piece_surfaces[self.current_piece_style][color][piece_type]
                        piece_rect = piece_surface.get_rect()
                        piece_rect.center = self.drag_pos
                        
                        temp_surface = piece_surface.copy()
                        temp_surface.set_alpha(180)
                        self.screen.blit(temp_surface, piece_rect)
                
                pygame.display.flip()
                clock.tick(60)
            
            except Exception as e:
                print(f"Error in game loop: {e}")
                traceback.print_exc()
        
        if self.engine:
            try:
                self.engine.quit()
            except:
                pass
        if self.ai_engine:
            try:
                self.ai_engine.quit()
            except:
                pass
        pygame.quit()
        print("Game ended")

if __name__ == "__main__":
    try:
        game = ChessGame()
        game.run()
    except KeyboardInterrupt:
        print("\nGame interrupted by user")
    except Exception as e:
        print(f"An error occurred: {e}")
        traceback.print_exc()
