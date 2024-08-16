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
