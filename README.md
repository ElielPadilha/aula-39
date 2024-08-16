<!DOCTYPE html>
<html>
<head>
    <title>Jogo da Velha</title>
    <style>
        .board { display: grid; grid-template-columns: repeat(3, 100px); gap: 5px; }
        .cell { width: 100px; height: 100px; display: flex; align-items: center; justify-content: center; font-size: 2em; border: 1px solid #000; cursor: pointer; }
        .cell.X { color: red; }
        .cell.O { color: blue; }
    </style>
</head>
<body>
    <h1>Jogo da Velha</h1>
    <div id="status">Aguardando jogadores...</div>
    <div id="board" class="board">
        <div class="cell" data-index="0"></div>
        <div class="cell" data-index="1"></div>
        <div class="cell" data-index="2"></div>
        <div class="cell" data-index="3"></div>
        <div class="cell" data-index="4"></div>
        <div class="cell" data-index="5"></div>
        <div class="cell" data-index="6"></div>
        <div class="cell" data-index="7"></div>
        <div class="cell" data-index="8"></div>
    </div>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.0.1/socket.io.min.js"></script>
    <script>
        const socket = io('http://' + window.location.hostname + ':8000');
        let symbol = prompt("Escolha seu símbolo: X ou O");
        let room = prompt("Escolha um nome para a sala:");

        socket.emit('join_game', { room, symbol });

        socket.on('game_update', (data) => {
            document.getElementById('status').textContent = `É a vez de ${data.current_turn}`;
            const cells = document.querySelectorAll('.cell');
            cells.forEach((cell, index) => {
                cell.textContent = data.board[index];
                cell.className = `cell ${data.board[index]}`;
            });
        });

        socket.on('update_board', (data) => {
            document.querySelector(`.cell[data-index="${data.index}"]`).textContent = data.symbol;
            document.querySelector(`.cell[data-index="${data.index}"]`).className = `cell ${data.symbol}`;
        });

        socket.on('turn_change', (data) => {
            document.getElementById('status').textContent = `É a vez de ${data.current_turn}`;
        });

        socket.on('game_over', (data) => {
            document.getElementById('status').textContent = data.winner === 'Empate' ? 'O jogo terminou em empate! Reiniciando...' : `Jogador ${data.winner} venceu! Reiniciando...`;
            setTimeout(() => {
                // Após 2 segundos, o jogo é reiniciado e o status atualizado
                socket.emit('join_game', { room, symbol });
            }, 2000);
        });

        socket.on('time_up', (data) => {
            document.getElementById('status').textContent = data.message;
        });

        document.getElementById('board').addEventListener('click', (event) => {
            const cell = event.target;
            if (cell.classList.contains('cell') && cell.textContent === '' && document.getElementById('status').textContent.includes(symbol)) {
                const index = cell.dataset.index;
                socket.emit('make_move', { room, index, symbol });
            }
        });
    </script>
</body>
</html>






from flask import Flask, render_template, request
from flask_socketio import SocketIO, emit, join_room, leave_room
from flask_cors import CORS
import threading
import time

app = Flask(__name__)
app.config['SECRET_KEY'] = 'mysecret'
socketio = SocketIO(app, cors_allowed_origins="*")
CORS(app)

# Estado do tabuleiro e controle de jogadores
games = {}
TURN_TIME_LIMIT = 10  # Tempo limite em segundos para cada turno

@app.route('/')
def index():
    return render_template('jogo.html')

@socketio.on('join_game')
def join_game(data):
    room = data['room']
    player_symbol = data['symbol']
    socket_id = request.sid
    join_room(room)
    
    if room not in games:
        games[room] = {
            'board': ['' for _ in range(9)],
            'current_turn': 'X',
            'players': {'X': {'sid': None, 'color': 'red'}, 'O': {'sid': None, 'color': 'blue'}},
            'timer': None
        }
        
    games[room]['players'][player_symbol]['sid'] = socket_id
    emit('game_update', {
        'board': games[room]['board'],
        'current_turn': games[room]['current_turn'],
        'color': games[room]['players'][player_symbol]['color']
    }, room=room)

@socketio.on('make_move')
def handle_move(data):
    room = data['room']
    index = int(data['index'])
    symbol = data['symbol']
    socket_id = request.sid
    
    if room in games:
        game = games[room]
        
        if game['board'][index] == '' and symbol == game['current_turn']:
            game['board'][index] = symbol
            emit('update_board', {
                'index': index,
                'symbol': symbol,
                'color': game['players'][symbol]['color']
            }, room=room)
            
            if check_winner(game['board'], symbol):
                emit('game_over', {'winner': symbol}, room=room)
                reset_board(room)
            elif '' not in game['board']:
                emit('game_over', {'winner': 'Empate'}, room=room)
                # Reinicia o jogo após um breve atraso
                socketio.sleep(2)  # Atraso para que os jogadores vejam a mensagem de empate
                reset_board(room)
            else:
                game['current_turn'] = 'O' if symbol == 'X' else 'X'
                start_turn_timer(room)
                emit('turn_change', {
                    'current_turn': game['current_turn'],
                    'color': game['players'][game['current_turn']]['color']
                }, room=room)
        else:
            emit('error', {'message': 'Invalid move or not your turn'}, room=socket_id)
    else:
        emit('error', {'message': 'Invalid game room'}, room=socket_id)

def check_winner(board, symbol):
    win_conditions = [
        [0, 1, 2], [3, 4, 5], [6, 7, 8], # Linhas
        [0, 3, 6], [1, 4, 7], [2, 5, 8], # Colunas
        [0, 4, 8], [2, 4, 6]             # Diagonais
    ]
    return any(all(board[i] == symbol for i in condition) for condition in win_conditions)

def reset_board(room):
    game = games[room]
    game['board'] = ['' for _ in range(9)]
    game['current_turn'] = 'X'
    game['timer'] = None

def start_turn_timer(room):
    def timer_thread():
        time.sleep(TURN_TIME_LIMIT)
        game = games.get(room)
        if game and game['current_turn']:
            emit('time_up', {'message': 'Turn time expired, switching player.'}, room=room)
            game['current_turn'] = 'O' if game['current_turn'] == 'X' else 'X'
            emit('turn_change', {
                'current_turn': game['current_turn'],
                'color': game['players'][game['current_turn']]['color']
            }, room=room)
            start_turn_timer(room)  # Restart the timer for the next turn
    
    game = games[room]
    if game['timer']:
        game['timer'].cancel()
    game['timer'] = threading.Timer(TURN_TIME_LIMIT, timer_thread)
    game['timer'].start()

@socketio.on('disconnect')
def handle_disconnect():
    for room, game in games.items():
        for symbol, player in game['players'].items():
            if player['sid'] == request.sid:
                leave_room(room)
                game['players'][symbol]['sid'] = None
                if all(player['sid'] is None for player in game['players'].values()):
                    del games[room]
                break

if __name__ == '__main__':
    socketio.run(app, port=8000, debug=True, host="0.0.0.0")
