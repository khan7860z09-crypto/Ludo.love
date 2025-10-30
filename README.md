<!DOCTYPE html>
<html lang="hi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Love Ludo - Pyaara Khel</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Use Inter font family -->
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #fce7f3; / Light pink background /
        }
        / Custom styles for the Ludo board /
        .ludo-board {
            display: grid;
            grid-template-columns: repeat(15, 1fr);
            grid-template-rows: repeat(15, 1fr);
            width: 90vw;
            max-width: 450px;
            height: 90vw;
            max-height: 450px;
            margin: 0 auto;
            border: 4px solid #f97316; / Orange accent /
            box-shadow: 0 10px 20px rgba(236, 72, 153, 0.5); / Pink shadow /
            background-color: #fff;
        }

        .cell {
            border: 1px solid #fbcfe8; / Light pink borders /
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 0.6rem;
            position: relative; / Crucial for piece positioning /
            overflow: hidden; / Prevent pieces from sticking out too much /
        }

        .home-area {
            background-color: #fecaca; / Red-pink home /
            border: none;
        }
        .safe-spot {
            background-color: #f9a8d4; / Darker pink safe spot /
        }
        .start-spot-pink {
            background-color: #f43f5e; / Pink Start /
            border: 3px solid #f97316;
        }
        .start-spot-blue {
            background-color: #3b82f6; / Blue Start /
            border: 3px solid #f97316;
        }
        .path-cell {
            background-color: #ffe4e6; / Very light pink path /
        }

        .piece {
            width: 60%;
            height: 60%;
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            border: 2px solid #fff;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
            transition: transform 0.1s ease-in-out;
            position: absolute; / Allows for stacking adjustments /
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%); / Center piece initially /
            z-index: 10; 
        }
        .piece:hover {
            transform: translate(-50%, -50%) scale(1.1);
        }
        .piece.active {
            animation: pulse 1s infinite alternate;
        }
        @keyframes pulse {
            from { box-shadow: 0 0 0 0 rgba(236, 72, 153, 0.7); }
            to { box-shadow: 0 0 0 8px rgba(236, 72, 153, 0); }
        }

        .dice {
            font-size: 2rem;
            color: #fff;
            width: 80px;
            height: 80px;
            display: flex;
            align-items: center;
            justify-content: center;
            background: linear-gradient(45deg, #f43f5e, #ec4899);
            border-radius: 12px;
            box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
            cursor: pointer;
            transition: transform 0.2s;
        }
        .dice:active {
            transform: scale(0.95);
        }
        .chat-box {
            max-height: 200px;
            overflow-y: auto;
            border: 1px solid #fbcfe8;
            background-color: #fff;
        }
        .player-info {
            border-radius: 0.75rem;
        }
    </style>
    <!-- Firebase Imports -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, onSnapshot, setDoc, updateDoc, arrayUnion, deleteDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        
        // Firestore setup variables
        const appId = typeof app_id !== 'undefined' ? app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof firebase_config !== 'undefined' ? firebase_config : '{}');
        const initialAuthToken = typeof initial_auth_token !== 'undefined' ? initial_auth_token : null;

        let db, auth, gameRef;
        let gameState = null;
        let gameId = null;
        let myId = null;
        let myPlayerIndex = -1;
        
        // --- Game Setup Data ---
        const PLAYER_NAMES = ["Pyaara", "Jaanu"];
        const PLAYER_COLORS = ["#f43f5e", "#3b82f6"]; // Pink (Red) and Blue (Green)
        
        // Ludo Board Path Coordinates (15x15 grid, top-left is 0,0)
        const PATH = {
            // Pink path (Player 0) - Starts at (6, 1) and enters home at (7, 1)
            'pink': [
                [6, 1], [6, 2], [6, 3], [6, 4], [6, 5], [5, 6], [4, 6], [3, 6], [2, 6], [1, 6], 
                [0, 7], // 10
                [1, 8], [2, 8], [3, 8], [4, 8], [5, 8], [6, 9], [6, 10], [6, 11], [6, 12], [6, 13], 
                [7, 14], // 21
                [8, 13], [8, 12], [8, 11], [8, 10], [8, 9], [9, 8], [10, 8], [11, 8], [12, 8], [13, 8],
                [14, 7], // 32
                [13, 6], [12, 6], [11, 6], [10, 6], [9, 6], [8, 5], [8, 4], [8, 3], [8, 2], [8, 1],
                [7, 0], // 43 - Entry before final path
                // Pink Home Path (44-49)
                [7, 1], [7, 2], [7, 3], [7, 4], [7, 5], [7, 6]
            ],
            // Blue path (Player 1) - Starts at (8, 13) and enters home at (13, 7)
            'blue': [
                [8, 13], [8, 12], [8, 11], [8, 10], [8, 9], [9, 8], [10, 8], [11, 8], [12, 8], [13, 8],
                [14, 7], // 10
                [13, 6], [12, 6], [11, 6], [10, 6], [9, 6], [8, 5], [8, 4], [8, 3], [8, 2], [8, 1],
                [7, 0], // 21
                [6, 1], [6, 2], [6, 3], [6, 4], [6, 5], [5, 6], [4, 6], [3, 6], [2, 6], [1, 6], 
                [0, 7], // 32
                [1, 8], [2, 8], [3, 8], [4, 8], [5, 8], [6, 9], [6, 10], [6, 11], [6, 12], [6, 13],
                [7, 14], // 43 - Entry before final path
                // Blue Home Path (44-49)
                [13, 7], [12, 7], [11, 7], [10, 7], [9, 7], [8, 7]
            ]
        };

        const START_POSITIONS = {
            'pink': [[1, 1], [1, 4], [4, 1], [4, 4]], // Home positions
            'blue': [[10, 10], [10, 13], [13, 10], [13, 13]] // Home positions
        };
        const WIN_POSITION = 50; // Piece is safe inside the home
        // Coordinates for all safe spots (start spots and starred path spots)
        const SAFE_SPOTS_COORDS = ['6,1', '8,13', '1,8', '6,13', '8,1', '13,6', '1,6', '13,8', '7,1', '7,2', '7,3', '7,4', '7,5', '7,6', '13,7', '12,7', '11,7', '10,7', '9,7', '8,7'];
        
        // --- Utility Functions for TTS (Voice Messages) ---
        function base64ToArrayBuffer(base64) {
            const binaryString = atob(base64);
            const len = binaryString.length;
            const bytes = new Uint8Array(len);
            for (let i = 0; i < len; i++) {
                bytes[i] = binaryString.charCodeAt(i);
            }
            return bytes.buffer;
        }

        function pcmToWav(pcmData, sampleRate) {
            const numChannels = 1;
            const bytesPerSample = 2;
            const blockAlign = numChannels * bytesPerSample;
            const byteRate = sampleRate * blockAlign;
            const dataSize = pcmData.length * bytesPerSample;
            const buffer = new ArrayBuffer(44 + dataSize);
            const view = new DataView(buffer);
            view.setUint32(0, 0x52494646, true);
            view.setUint32(4, 36 + dataSize, true);
            view.setUint32(8, 0x57415645, true);
            view.setUint32(12, 0x666d7420, true);
            view.setUint32(16, 16, true);
            view.setUint16(20, 1, true);
            view.setUint16(22, numChannels, true);
            view.setUint32(24, sampleRate, true);
            view.setUint32(28, byteRate, true);
            view.setUint16(32, blockAlign, true);
            view.setUint16(34, 16, true);
            view.setUint32(36, 0x64617461, true);
            view.setUint32(40, dataSize, true);
            
            let offset = 44;
            for (let i = 0; i < pcmData.length; i++) {
                view.setInt16(offset, pcmData[i], true);
                offset += 2;
            }
            return new Blob([view], { type: 'audio/wav' });
        }

        async function playTTS(text, voice = 'Puck') {
            const loadingIndicator = document.getElementById('loading-tts');
            const playButton = document.getElementById('tts-play-button');
            if (loadingIndicator && playButton) {
                playButton.classList.add('hidden');
                loadingIndicator.classList.remove('hidden');
            }

            const payload = {
                contents: [{ parts: [{ text: text }] }],
                generationConfig: {
                    responseModalities: ["AUDIO"],
                    speechConfig: { voiceConfig: { prebuiltVoiceConfig: { voiceName: voice } } }
                },
                model: "gemini-2.5-flash-preview-tts"
            };

            const apiKey = "";
            const apiUrl = “https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent?key=${apiKey}”;

            try {
                const response = await fetch(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });

                const result = await response.json();
                const part = result?.candidates?.[0]?.content?.parts?.[0];
                const audioData = part?.inlineData?.data;
                const mimeType = part?.inlineData?.mimeType;

                if (audioData && mimeType && mimeType.startsWith("audio/")) {
                    const match = mimeType.match(/rate=(\d+)/);
                    const sampleRate = match ? parseInt(match[1], 10) : 24000;
                    const pcmData = base64ToArrayBuffer(audioData);
                    const pcm16 = new Int16Array(pcmData);
                    const wavBlob = pcmToWav(pcm16, sampleRate);
                    const audioUrl = URL.createObjectURL(wavBlob);
                    
                    const audio = new Audio(audioUrl);
                    audio.play();

                    audio.onended = () => {
                        if (loadingIndicator && playButton) {
                            playButton.classList.remove('hidden');
                            loadingIndicator.classList.add('hidden');
                        }
                        URL.revokeObjectURL(audioUrl);
                    };
                } else {
                    console.error("TTS API did not return valid audio data.");
                }

            } catch (error) {
                console.error("Error calling TTS API:", error);
            } finally {
                if (loadingIndicator && playButton) {
                    playButton.classList.remove('hidden');
                    loadingIndicator.classList.add('hidden');
                }
            }
        }
        
        // --- Firebase/Auth/Init ---
        async function initializeFirebase() {
            if (Object.keys(firebaseConfig).length === 0) {
                console.error("Firebase config is missing.");
                document.getElementById('status-message').textContent = "त्रुटि: Firebase कॉन्फ़िगरेशन नहीं मिला।";
                return;
            }
            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                setLogLevel('Debug');
                
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }

                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        myId = user.uid;
                        document.getElementById('status-message').textContent = "आप सफलतापूर्वक जुड़ गए हैं!";
                        document.getElementById('connection-modal').classList.remove('hidden');
                        
                    } else {
                        myId = null;
                        document.getElementById('status-message').textContent = "प्रमाणीकरण विफल।";
                    }
                });

            } catch (error) {
                console.error("Firebase initialization or auth error:", error);
                document.getElementById('status-message').textContent = "त्रुटि: Firebase शुरू करने में समस्या। कंसोल देखें।";
            }
        }

        // --- Game Core Functions ---

        function getGamePath(id) {
            return “/artifacts/${appId}/public/data/ludoGames/${id}”;
        }
        
        async function createNewGame() {
            const newGameId = Math.random().toString(36).substring(2, 8).toUpperCase();
            gameId = newGameId;
            
            const newGameState = {
                status: 'waiting',
                diceValue: 0,
                currentTurn: myId, 
                chat: [],
                winner: null,
                players: [
                    { id: myId, name: PLAYER_NAMES[0], color: PLAYER_COLORS[0], pieces: [0, 0, 0, 0], boardPath: 'pink' },
                ]
            };

            gameRef = doc(db, getGamePath(gameId));
            await setDoc(gameRef, newGameState);
            showMessage(“नया गेम बन गया है! Love ID: ${gameId}”, '#4ade80');
            joinGame(gameId); 
        }

        async function joinGame(id) {
            gameId = id;
            gameRef = doc(db, getGamePath(gameId));
            
            document.getElementById('game-id-display').textContent = gameId;
            document.getElementById('connection-modal').classList.add('hidden');
            document.getElementById('game-container').classList.remove('hidden');
            
            const docSnap = await getDoc(gameRef);
            if (!docSnap.exists()) {
                showMessage(“गेम ID ${gameId} मौजूद नहीं है। कृपया ID जांचें।”, '#f97316');
                document.getElementById('connection-modal').classList.remove('hidden');
                return;
            }
            
            const currentGameState = docSnap.data();
            
            const isPlayerPresent = currentGameState.players.some(p => p.id === myId);
            
            if (!isPlayerPresent && currentGameState.players.length === 1 && currentGameState.status === 'waiting') {
                // Second player (Jaanu) joining
                const updatedPlayers = [...currentGameState.players, { 
                    id: myId, 
                    name: PLAYER_NAMES[1], 
                    color: PLAYER_COLORS[1], 
                    pieces: [0, 0, 0, 0],
                    boardPath: 'blue' 
                }];

                await setDoc(gameRef, { 
                    ...currentGameState, 
                    players: updatedPlayers,
                    status: 'playing' 
                });
                showMessage(“आप ${PLAYER_NAMES[1]} बनकर गेम में शामिल हो गए हैं। खेल शुरू!”, '#4ade80');
            } else if (isPlayerPresent) {
                showMessage(“आप पहले ही गेम में मौजूद हैं।”, '#3b82f6');
            } else if (currentGameState.players.length >= 2) {
                showMessage(“गेम पहले ही भरा हुआ है।”, '#f97316');
            } else {
                showMessage(“पार्टनर के जुड़ने का इंतजार है... Love ID: ${gameId}”, '#f97316');
            }
            
            // Set up real-time listener for the game state
            onSnapshot(gameRef, (snap) => {
                if (snap.exists()) {
                    gameState = snap.data();
                    updateLocalPlayerInfo();
                    renderGame();
                    renderChat();
                    
                    if (gameState.status === 'finished' && gameState.winner) {
                        showMessage(“${gameState.winner} जीत गया! बधाई हो!”, '#4ade80');
                    } else if (gameState.status === 'playing' && gameState.players.length === 2 && !gameState.alertedStart) {
                         showMessage("दोनों खिलाड़ी जुड़ गए हैं! पासा फेंकें।", '#10b988');
                         updateDoc(gameRef, { alertedStart: true }); // Prevent repeated alert
                    }


                } else {
                    console.warn("Game document does not exist anymore.");
                    showMessage("गेम बंद हो गया है।");
                }
            }, (error) => {
                console.error("Error listening to game state:", error);
                showMessage("गेम डेटा सुनने में त्रुटि।");
            });
        }
        
        function updateLocalPlayerInfo() {
            if (!gameState || !myId) return;
            myPlayerIndex = gameState.players.findIndex(p => p.id === myId);
            const myPlayer = gameState.players[myPlayerIndex];
            
            if (myPlayer) {
                document.getElementById('my-name').textContent = “${myPlayer.name}”;
                document.getElementById('my-name-display').style.backgroundColor = myPlayer.color;
            }
        }
        
        function showMessage(msg, color = '#f97316') {
            const messageBox = document.getElementById('game-message');
            messageBox.textContent = msg;
            messageBox.style.backgroundColor = color;
            messageBox.classList.remove('hidden');
            clearTimeout(messageBox.dataset.timeout);
            messageBox.dataset.timeout = setTimeout(() => { messageBox.classList.add('hidden'); }, 4000);
        }

        // --- Chat Functions ---
        async function sendChatMessage() {
            const chatInput = document.getElementById('chat-input');
            const messageText = chatInput.value.trim();
            
            if (messageText === "" || !gameRef) return;

            const myPlayer = gameState.players.find(p => p.id === myId);
            const message = {
                sender: myPlayer ? myPlayer.name : 'Guest',
                text: messageText,
                timestamp: Date.now()
            };

            await updateDoc(gameRef, {
                chat: arrayUnion(message)
            });

            chatInput.value = '';
            document.getElementById('chat-box').scrollTop = document.getElementById('chat-box').scrollHeight;
        }

        function renderChat() {
            const chatBox = document.getElementById('chat-box');
            
            if (gameState && gameState.chat) {
                // Simple optimization: only update if new messages arrived
                if (chatBox.children.length === gameState.chat.length) return; 
                
                chatBox.innerHTML = '';
                
                gameState.chat.forEach(msg => {
                    const msgElement = document.createElement('div');
                    msgElement.className = 'p-2 rounded-lg my-1 max-w-[80%] break-words';
                    const isMe = gameState.players.find(p => p.id === myId)?.name === msg.sender;
                    
                    if (isMe) {
                        msgElement.className += ' bg-pink-500 text-white self-end ml-auto';
                    } else {
                        msgElement.className +            <input type="text" id="game-id-input" placeholder="Love ID (जैसे: A1B2C3)" 
                   class="w-full p-3 border border-pink-300 rounded-xl mb-3 focus:ring-2 focus:ring-pink-500 uppercase text-center font-mono">
            
            <button id="join-game-btn" class="w-full bg-blue-500 text-white p-3 rounded-xl font-semibold hover:bg-blue-600 transition shadow-md">
                खेल में शामिल हों
            </button>
        </div>
    </div>

    <!-- Main Game Container -->
    <div id="game-container" class="hidden w-full max-w-lg">
        
        <!-- Header & Info -->
        <div class="flex justify-between items-center mb-4 p-3 bg-white rounded-xl shadow-lg">
            <div class="flex flex-col">
                <p class="text-xs text-gray-500">Love ID:</p>
                <code id="game-id-display" class="font-mono text-pink-600 text-lg font-bold">...</code>
            </div>
            <div id="my-name-display" class="player-info p-2 px-4 text-white font-bold shadow-inner">
                मेरा नाम: <span id="my-name">...</span>
            </div>
        </div>

        <!-- Game Area (Board and Controls) -->
        <div class="flex flex-col space-y-4">
            
            <!-- Ludo Board -->
            <div id="ludo-board" class="ludo-board relative">
                <!-- Board cells will be rendered by JS -->
            </div>

            <!-- Controls (Dice and Turn) -->
            <div class="flex items-center justify-around p-4 bg-white rounded-xl shadow-lg">
                <div class="flex flex-col items-center">
                    <div id="turn-message" class="text-sm font-semibold p-2 rounded-lg text-white mb-2" style="background-color:#ef4444;">...</div>
                    <div id="dice-roll-button" class="dice shadow-xl bg-pink-500 hover:bg-pink-600" onclick="rollDice()">
                        🎲
                    </div>
                    <p class="text-xs text-gray-500 mt-2">पासा फेंकें</p>
                </div>
            </div>
        </div>
        
        <!-- Romantic Music Player -->
        <div class="mt-6 bg-white p-4 rounded-xl shadow-lg">
            <h3 class="text-xl font-bold text-red-500 mb-3 border-b pb-2">🎶 रोमांटिक म्यूजिक प्लेयर</h3>
            <p class="text-sm text-gray-600 mb-2">गाने का URL (जैसे: “.mp3” या “.wav”) यहाँ डालें:</p>
            <div class="flex mb-3">
                <input type="url" id="music-url-input" placeholder="https://example.com/song.mp3" 
                       class="flex-grow p-2 border border-pink-300 rounded-l-lg focus:outline-none focus:ring-2 focus:ring-red-500"
                       value="">
                <button id="play-music-btn" class="bg-red-500 text-white p-2 font-semibold hover:bg-red-600 transition">
                    ▶️ प्ले
                </button>
                <button id="pause-music-btn" class="bg-gray-500 text-white p-2 rounded-r-lg font-semibold hover:bg-gray-600 transition">
                    ⏸️ पॉज
                </button>
            </div>
            <p id="music-status" class="text-sm text-center text-gray-500">कोई गाना नहीं चल रहा है।</p>
        </div>


        <!-- Real-time Chat -->
        <div class="mt-6 bg-white p-4 rounded-xl shadow-lg">
            <h3 class="text-xl font-bold text-pink-600 mb-3 border-b pb-2">💬 बातचीत (Chat)</h3>
            <div id="chat-box" class="chat-box flex flex-col space-y-2 p-2 mb-3 rounded-lg">
                <!-- Chat messages will be rendered here -->
            </div>
            
            <div class="flex">
                <input type="text" id="chat-input" placeholder="प्यार भरा संदेश लिखें..." 
                       class="flex-grow p-2 border border-pink-300 rounded-l-lg focus:outline-none focus:ring-2 focus:ring-pink-500">
                <button id="chat-send-btn" class="bg-pink-500 text-white p-2 rounded-r-lg font-semibold hover:bg-pink-600 transition">
                    भेजें
                </button>
            </div>

             <!-- TTS (Text-to-Speech) Feature -->
            <div class="mt-4 pt-4 border-t border-gray-200">
                <h4 class="text-md font-semibold text-gray-700 mb-2">अपनी बात सुनाएं (Voice Message)</h4>
                <div class="flex items-center">
                    <input type="text" id="tts-input" placeholder="कुछ बोलकर भेजें (सिर्फ आपको सुनाई देगा)" 
                           class="flex-grow p-2 border border-pink-300 rounded-l-lg focus:outline-none focus:ring-2 focus:ring-pink-500">
                    <button id="tts-play-button" class="bg-blue-500 text-white p-2 rounded-r-lg font-semibold hover:bg-blue-600 transition flex items-center justify-center" title="मैसेज बोलकर सुनें">
                         <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
                            <path stroke-linecap="round" stroke-linejoin="round" d="M19.114 5.636A9 9 0 0 1 12 2.25a9 9 0 0 1 6.886 3.386M17.482 8.766A6.75 6.75 0 0 1 12 6a6.75 6.75 0 0 1 5.482 2.766M16 12a4 4 0 1 1-8 0 4 4 0 0 1 8 0Z" />
                        </svg>

                    </button>
                     <div id="loading-tts" class="hidden bg-blue-500 text-white p-2 rounded-r-lg font-semibold flex items-center justify-center w-[44px] h-[44px]">
                         <svg class="animate-spin h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                        </svg>
                    </div>
                </div>
            </div>
        </div>
        
    </div>
</body>
</html>
