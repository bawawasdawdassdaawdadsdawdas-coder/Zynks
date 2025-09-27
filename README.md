<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Zynk - Chat Privado</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; padding: 0; background: #f4f4f4; }
        .container { max-width: 1000px; margin: auto; padding: 20px; }
        .hidden { display: none; }
        .chat-box { border: 1px solid #ccc; padding: 10px; height: 400px; overflow-y: auto; background: white; }
        .message { margin: 5px 0; }
        .message strong { color: #333; }
        .sidebar { float: left; width: 25%; height: 400px; overflow-y: auto; background: #eee; padding: 10px; }
        .main { float: left; width: 70%; margin-left: 5%; }
        .clearfix { clear: both; }
        button { padding: 8px 12px; margin: 5px; cursor: pointer; }
        input { padding: 8px; margin: 5px 0; width: 100%; }
        .room { padding: 5px; cursor: pointer; border-bottom: 1px solid #ccc; }
        .room:hover { background: #ddd; }
    </style>
</head>
<body>
    <div class="container">
        <!-- Tela de Login -->
        <div id="loginScreen">
            <h2>Login no Zynk</h2>
            <input type="text" id="username" placeholder="Nome de usuário">
            <button onclick="login()">Entrar</button>
            <button onclick="showRegister()">Criar Conta</button>
            <button onclick="showAdminLogin()">Login Admin</button>
        </div>

        <!-- Tela de Registro -->
        <div id="registerScreen" class="hidden">
            <h2>Criar Conta</h2>
            <input type="text" id="newUsername" placeholder="Nome de usuário">
            <button onclick="register()">Registrar</button>
            <button onclick="showLogin()">Voltar</button>
        </div>

        <!-- Tela de Login Admin -->
        <div id="adminLoginScreen" class="hidden">
            <h2>Login Administrador</h2>
            <input type="password" id="adminPassword" placeholder="Senha do admin">
            <button onclick="adminLogin()">Entrar</button>
            <button onclick="showLogin()">Voltar</button>
        </div>

        <!-- Tela de Usuário -->
        <div id="userScreen" class="hidden">
            <h2 id="welcomeUser"></h2>
            <div class="sidebar">
                <h3>Suas Conversas</h3>
                <div id="roomList"></div>
            </div>
            <div class="main">
                <div class="chat-box" id="chatBox"></div>
                <input type="text" id="messageInput" placeholder="Digite sua mensagem...">
                <button onclick="sendMessage()">Enviar</button>
            </div>
            <div class="clearfix"></div>
            <button onclick="logout()">Sair</button>
        </div>

        <!-- Tela de Admin -->
        <div id="adminScreen" class="hidden">
            <h2>Painel do Administrador</h2>
            <div class="sidebar">
                <h3>Conversas</h3>
                <div id="adminRoomList"></div>
            </div>
            <div class="main">
                <div class="chat-box" id="adminChatBox"></div>
            </div>
            <div class="clearfix"></div>
            <h3>Estatísticas</h3>
            <div id="stats"></div>
            <button onclick="logout()">Sair</button>
        </div>
    </div>

    <script>
        // ============ BANCO DE DADOS LOCAL ============
        let users = JSON.parse(localStorage.getItem('zynkUsers')) || {};
        let messages = JSON.parse(localStorage.getItem('zynkMessages')) || {};
        let rooms = JSON.parse(localStorage.getItem('zynkRooms')) || {};
        let currentUser = JSON.parse(localStorage.getItem('zynkCurrentUser')) || null;
        let currentRoom = null;

        const adminPassword = "12345"; // senha admin fixa

        // ============ SALVAR ============
        function saveData() {
            localStorage.setItem('zynkUsers', JSON.stringify(users));
            localStorage.setItem('zynkMessages', JSON.stringify(messages));
            localStorage.setItem('zynkRooms', JSON.stringify(rooms));
            localStorage.setItem('zynkCurrentUser', JSON.stringify(currentUser));
        }

        // ============ LOGIN & REGISTRO ============
        function login() {
            const username = document.getElementById("username").value.trim();
            if (users[username]) {
                currentUser = { name: username };
                saveData();
                showUserScreen();
            } else {
                alert("Usuário não encontrado.");
            }
        }

        function register() {
            const username = document.getElementById("newUsername").value.trim();
            if (!username) return alert("Digite um nome válido!");
            if (users[username]) return alert("Usuário já existe!");
            users[username] = { name: username, created: new Date().toISOString() };
            rooms[username] = { name: `Sala de ${username}`, owner: username, created: new Date().toISOString() };
            messages[username] = [];
            saveData();
            alert("Conta criada com sucesso!");
            showLogin();
        }

        function adminLogin() {
            const pass = document.getElementById("adminPassword").value;
            if (pass === adminPassword) {
                currentUser = { name: "admin" };
                saveData();
                showAdminScreen();
            } else {
                alert("Senha incorreta.");
            }
        }

        function logout() {
            currentUser = null;
            saveData();
            showLogin();
        }

        // ============ INTERFACE ============
        function showLogin() {
            document.getElementById("loginScreen").classList.remove("hidden");
            document.getElementById("registerScreen").classList.add("hidden");
            document.getElementById("userScreen").classList.add("hidden");
            document.getElementById("adminScreen").classList.add("hidden");
            document.getElementById("adminLoginScreen").classList.add("hidden");
        }

        function showRegister() {
            document.getElementById("loginScreen").classList.add("hidden");
            document.getElementById("registerScreen").classList.remove("hidden");
        }

        function showAdminLogin() {
            document.getElementById("loginScreen").classList.add("hidden");
            document.getElementById("adminLoginScreen").classList.remove("hidden");
        }

        function showUserScreen() {
            document.getElementById("loginScreen").classList.add("hidden");
            document.getElementById("registerScreen").classList.add("hidden");
            document.getElementById("userScreen").classList.remove("hidden");
            document.getElementById("adminScreen").classList.add("hidden");
            document.getElementById("adminLoginScreen").classList.add("hidden");
            document.getElementById("welcomeUser").innerText = "Bem-vindo, " + currentUser.name;
            loadRooms();
        }

        function showAdminScreen() {
            document.getElementById("loginScreen").classList.add("hidden");
            document.getElementById("registerScreen").classList.add("hidden");
            document.getElementById("userScreen").classList.add("hidden");
            document.getElementById("adminScreen").classList.remove("hidden");
            document.getElementById("adminLoginScreen").classList.add("hidden");
            loadAdminRooms();
            showStats();
        }

        // ============ CONVERSAS ============
        function loadRooms() {
            const roomList = document.getElementById("roomList");
            roomList.innerHTML = "";
            for (let roomId in rooms) {
                if (rooms[roomId].owner === currentUser.name) {
                    let div = document.createElement("div");
                    div.className = "room";
                    div.innerText = rooms[roomId].name;
                    div.onclick = () => openRoom(roomId);
                    roomList.appendChild(div);
                }
            }
        }

        function loadAdminRooms() {
            const roomList = document.getElementById("adminRoomList");
            roomList.innerHTML = "";
            for (let roomId in rooms) {
                let div = document.createElement("div");
                div.className = "room";
                div.innerText = rooms[roomId].name;
                div.onclick = () => openAdminRoom(roomId);
                roomList.appendChild(div);
            }
        }

        function openRoom(roomId) {
            currentRoom = roomId;
            loadMessages();
        }

        function openAdminRoom(roomId) {
            currentRoom = roomId;
            loadAdminMessages();
        }

        // ============ MENSAGENS ============
        function sendMessage() {
            const input = document.getElementById("messageInput");
            const text = input.value.trim();
            if (!text || !currentRoom) return;
            messages[currentRoom].push({ sender: currentUser.name, text, time: new Date().toISOString() });
            input.value = "";
            saveData();
            loadMessages();
        }

        function loadMessages() {
            if (!currentRoom) return;
            const chatBox = document.getElementById("chatBox");
            chatBox.innerHTML = "";
            messages[currentRoom].forEach(msg => {
                let div = document.createElement("div");
                div.className = "message";
                div.innerHTML = `<strong>${msg.sender}</strong>: ${msg.text} <small>(${formatTime(msg.time)})</small>`;
                chatBox.appendChild(div);
            });
            chatBox.scrollTop = chatBox.scrollHeight;
        }

        function loadAdminMessages() {
            if (!currentRoom) return;
            const chatBox = document.getElementById("adminChatBox");
            chatBox.innerHTML = "";
            messages[currentRoom].forEach(msg => {
                let div = document.createElement("div");
                div.className = "message";
                div.innerHTML = `<strong>${msg.sender}</strong>: ${msg.text} <small>(${formatTime(msg.time)})</small>`;
                chatBox.appendChild(div);
            });
            chatBox.scrollTop = chatBox.scrollHeight;
        }

        // ============ ESTATÍSTICAS ============
        function showStats() {
            let totalUsers = Object.keys(users).length;
            let totalRooms = Object.keys(rooms).length;
            let totalMessages = 0;
            for (let r in messages) totalMessages += messages[r].length;
            document.getElementById("stats").innerHTML = `
                Usuários: ${totalUsers} <br>
                Salas: ${totalRooms} <br>
                Mensagens: ${totalMessages}
            `;
        }

        // ============ FORMATAR HORA ============
        function formatTime(timestamp) {
            return new Date(timestamp).toLocaleTimeString('pt-BR', { 
                hour: '2-digit',
                minute: '2-digit'
            });
        }

        // ============ AUTO REFRESH ============
        setInterval(() => {
            if (currentUser && currentUser.name === "admin" && currentRoom) {
                loadAdminMessages();
            } else if (currentRoom) {
                loadMessages();
            }
        }, 2000);

        // ============ AUTO LOGIN ============
        if (currentUser) {
            if (currentUser.name === "admin") showAdminScreen();
            else showUserScreen();
        }
    </script>
</body>
</html>
