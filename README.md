<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI Motion Dance Challenge</title>
    <!-- Tailwind CSS for styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- MediaPipe Pose for motion tracking -->
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/pose/pose.js" crossorigin="anonymous"></script>
    <style>
        .mirror {
            transform: scaleX(-1);
        }
    </style>
</head>
<body class="bg-slate-900 text-white min-h-screen flex flex-col items-center justify-center p-4">

    <!-- Header Section -->
    <header class="text-center mb-6">
        <h1 class="text-4xl font-extrabold text-transparent bg-clip-text bg-gradient-to-r from-pink-500 to-purple-500">
            AI Motion Dance Challenge
        </h1>
        <p class="text-slate-400 mt-2">Match the pose on screen to earn points and win prizes!</p>
    </header>

    <!-- Main Game Container -->
    <main class="relative w-full max-w-4xl bg-slate-800 rounded-2xl overflow-hidden shadow-2xl border border-slate-700">
        
        <!-- Video & Canvas Display -->
        <div class="relative aspect-video bg-black flex items-center justify-center overflow-hidden">
            <video id="webcam" class="absolute inset-0 w-full h-full object-cover mirror" playsinline></video>
            <canvas id="output_canvas" class="absolute inset-0 w-full h-full object-cover mirror"></canvas>
            
            <!-- Game HUD Overlay -->
            <div id="hud" class="absolute top-4 left-4 right-4 flex justify-between items-center pointer-events-none z-10">
                <div class="bg-slate-900/80 backdrop-blur border border-slate-700 px-4 py-2 rounded-xl text-center">
                    <span class="text-xs uppercase tracking-wider text-slate-400 block">Score</span>
                    <span id="score-display" class="text-2xl font-black text-yellow-400">0</span>
                </div>
                <div class="bg-slate-900/80 backdrop-blur border border-slate-700 px-4 py-2 rounded-xl text-center">
                    <span class="text-xs uppercase tracking-wider text-slate-400 block">Target Pose</span>
                    <span id="pose-display" class="text-xl font-bold text-emerald-400">Ready?</span>
                </div>
                <div class="bg-slate-900/80 backdrop-blur border border-slate-700 px-4 py-2 rounded-xl text-center">
                    <span class="text-xs uppercase tracking-wider text-slate-400 block">Time Left</span>
                    <span id="timer-display" class="text-2xl font-black text-rose-400">30s</span>
                </div>
            </div>

            <!-- Start Screen Overlay -->
            <div id="start-overlay" class="absolute inset-0 bg-slate-900/90 backdrop-blur flex flex-col items-center justify-center p-6 z-20">
                <h2 class="text-2xl font-bold mb-4">Enter Player Info</h2>
                <input type="text" id="player-name" placeholder="Enter your name" class="px-4 py-3 bg-slate-800 border border-slate-600 rounded-xl mb-4 text-center focus:outline-none focus:border-purple-500 w-64 text-white">
                <button id="start-btn" class="px-8 py-3 bg-gradient-to-r from-pink-500 to-purple-600 hover:from-pink-600 hover:to-purple-700 text-white font-bold rounded-xl transition shadow-lg transform active:scale-95">
                    Start Game
                </button>
            </div>

            <!-- Game Over Overlay -->
            <div id="end-overlay" class="absolute inset-0 bg-slate-900/95 backdrop-blur flex flex-col items-center justify-center p-6 z-20 hidden">
                <h2 class="text-3xl font-black mb-2 text-transparent bg-clip-text bg-gradient-to-r from-yellow-400 to-amber-500">Game Over!</h2>
                <p id="final-stats" class="text-lg text-slate-300 mb-2"></p>
                <p id="prize-stats" class="text-md text-emerald-400 font-semibold mb-6"></p>
                <div id="sheet-status" class="text-xs text-slate-400 mb-6 italic">Logging score to Google Sheet...</div>
                <button id="restart-btn" class="px-8 py-3 bg-purple-600 hover:bg-purple-700 text-white font-bold rounded-xl transition shadow-lg">
                    Play Again
                </button>
            </div>
        </div>
    </main>

    <!-- Logic Script -->
    <script>
        // ==========================================
        // CONFIGURATION & GOOGLE SHEET INTEGRATION
        // ==========================================
        const GOOGLE_SHEET_SCRIPT_URL = "https://script.google.com/macros/s/AKfycbwcQZEVrNZq6uWqATZvUeCW0qkXHmFq0n5EhvwPFsjkxvEDEW__VWXtWQuvM2sFqn361Q/exec";

        // Game State Variables
        let score = 0;
        let timeLeft = 30;
        let gameInterval = null;
        let isGameActive = false;
        let currentPose = "Hands Up";
        let playerName = "Anonymous";

        // DOM Elements
        const videoElement = document.getElementById('webcam');
        const canvasElement = document.getElementById('output_canvas');
        const canvasCtx = canvasElement.getContext('2d');
        const scoreDisplay = document.getElementById('score-display');
        const timerDisplay = document.getElementById('timer-display');
        const poseDisplay = document.getElementById('pose-display');
        const startOverlay = document.getElementById('start-overlay');
        const endOverlay = document.getElementById('end-overlay');
        const startBtn = document.getElementById('start-btn');
        const restartBtn = document.getElementById('restart-btn');
        const playerNameInput = document.getElementById('player-name');
        const finalStats = document.getElementById('final-stats');
        const prizeStats = document.getElementById('prize-stats');
        const sheetStatus = document.getElementById('sheet-status');

        // Pose definitions
        const poses = ["Hands Up", "T-Pose", "Left Hand Up", "Right Hand Up"];

        // Initialize MediaPipe Pose
        const pose = new Pose({
            locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`
        });

        pose.setOptions({
            modelComplexity: 1,
            smoothLandmarks: true,
            minDetectionConfidence: 0.5,
            minTrackingConfidence: 0.5
        });

        pose.onResults(onResults);

        // Initialize Camera
        const camera = new Camera(videoElement, {
            onFrame: async () => {
                await pose.send({ image: videoElement });
            },
            width: 1280,
            height: 720
        });
        camera.start();

        // Canvas scaling
        function resizeCanvas() {
            canvasElement.width = videoElement.videoWidth || 640;
            canvasElement.height = videoElement.videoHeight || 480;
        }

        // Process Motion Results
        function onResults(results) {
            resizeCanvas();
            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);

            if (results.poseLandmarks && isGameActive) {
                // Draw skeleton joints
                drawConnectors(canvasCtx, results.poseLandmarks, POSE_CONNECTIONS, { color: '#00FF00', lineWidth: 4 });
                drawLandmarks(canvasCtx, results.poseLandmarks, { color: '#FF0000', lineWidth: 2 });

                // Pose Check Logic
                checkPoseMatch(results.poseLandmarks);
            }
            canvasCtx.restore();
        }

        // Detect current pose relative to landmarks
        function checkPoseMatch(landmarks) {
            const leftWrist = landmarks[15];
            const rightWrist = landmarks[16];
            const leftShoulder = landmarks[11];
            const rightShoulder = landmarks[12];

            let match = false;

            if (currentPose === "Hands Up" && leftWrist.y < leftShoulder.y && rightWrist.y < rightShoulder.y) {
                match = true;
            } else if (currentPose === "T-Pose" && Math.abs(leftWrist.y - leftShoulder.y) < 0.1 && Math.abs(rightWrist.y - rightShoulder.y) < 0.1) {
                match = true;
            } else if (currentPose === "Left Hand Up" && leftWrist.y < leftShoulder.y && rightWrist.y > rightShoulder.y) {
                match = true;
            } else if (currentPose === "Right Hand Up" && rightWrist.y < rightShoulder.y && leftWrist.y > leftShoulder.y) {
                match = true;
            }

            if (match) {
                score += 10;
                scoreDisplay.textContent = score;
                // Switch to next random pose
                currentPose = poses[Math.floor(Math.random() * poses.length)];
                poseDisplay.textContent = currentPose;
            }
        }

        // Start Game Function
        function startGame() {
            playerName = playerNameInput.value.trim() || "Anonymous";
            score = 0;
            timeLeft = 30;
            isGameActive = true;
            
            scoreDisplay.textContent = score;
            timerDisplay.textContent = `${timeLeft}s`;
            currentPose = poses[Math.floor(Math.random() * poses.length)];
            poseDisplay.textContent = currentPose;

            startOverlay.classList.add('hidden');
            endOverlay.classList.add('hidden');

            gameInterval = setInterval(() => {
                timeLeft--;
                timerDisplay.textContent = `${timeLeft}s`;
                if (timeLeft <= 0) {
                    endGame();
                }
            }, 1000);
        }

        // End Game & Send Data to Google Sheet
        async function endGame() {
            clearInterval(gameInterval);
            isGameActive = false;

            const prizesWon = Math.floor(score / 50);
            const gameResult = score >= 100 ? "Winner" : "Completed";

            finalStats.textContent = `Player: ${playerName} | Score: ${score}`;
            prizeStats.textContent = `Prizes Earned: ${prizesWon}`;
            sheetStatus.textContent = "Logging score to Google Sheet...";

            endOverlay.classList.remove('hidden');

            // Send score data to your Google Apps Script endpoint
            await sendScoreToGoogleSheet(playerName, score, prizesWon, gameResult);
        }

        // Send Score Data via POST request
        async function sendScoreToGoogleSheet(pName, pScore, pPrizes, pResult) {
            const payload = {
                playerName: pName,
                score: pScore,
                prizes: pPrizes,
                result: pResult
            };

            try {
                await fetch(GOOGLE_SHEET_SCRIPT_URL, {
                    method: "POST",
                    mode: "no-cors",
                    headers: {
                        "Content-Type": "application/json"
                    },
                    body: JSON.stringify(payload)
                });
                sheetStatus.textContent = "Score successfully logged to Google Sheet!";
                sheetStatus.classList.replace("text-slate-400", "text-emerald-400");
            } catch (error) {
                console.error("Error logging score:", error);
                sheetStatus.textContent = "Failed to log score. Check console for details.";
                sheetStatus.classList.replace("text-slate-400", "text-rose-400");
            }
        }

        // Event Listeners
        startBtn.addEventListener('click', startGame);
        restartBtn.addEventListener('click', startGame);
    </script>
</body>
</html>
