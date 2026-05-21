# Open-Class
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>스카이워크 탈출 게임</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js"></script>
    <style>
        /* 동글동글하고 가독성 좋은 나눔스퀘어라운드 폰트 적용 */
        @font-face {
            font-family: 'NanumSquareRound';
            src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/noonfonts_two@1.0/NanumSquareRound.woff') format('woff');
            font-weight: normal;
            font-style: normal;
        }
        
        body, html {
            margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden;
            font-family: 'NanumSquareRound', sans-serif;
            background-color: #87CEEB; /* 하늘색 배경 */
            touch-action: none; /* 모바일 새로고침 방지 */
            word-break: keep-all; /* 단어 중간에서 줄바꿈 방지 */
        }
        
        #game-container { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }
        canvas { display: block; width: 100%; height: 100%; }
        
        /* UI Overlays */
        .overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0, 0, 0, 0.7);
            display: flex; flex-direction: column; justify-content: center; align-items: center;
            z-index: 50; visibility: hidden; opacity: 0; transition: opacity 0.3s;
        }
        .overlay.active { visibility: visible; opacity: 1; }
        
        .game-panel {
            background: white; border-radius: 20px; padding: clamp(20px, 3vh, 40px);
            box-shadow: 0 15px 30px rgba(0,0,0,0.5);
            max-width: 95%; width: 1000px;
            max-height: 95vh; overflow-y: auto;
            position: relative;
        }

        .match-container { display: flex; justify-content: space-between; position: relative; margin-top: 2vh; }
        .match-col { display: flex; flex-direction: column; gap: 20px; z-index: 10; }
        .match-col.words { width: 15%; } /* 낱말 칸은 좁게 */
        .match-col.meanings { width: 60%; } /* 뜻 칸은 넓게 */
        
        .match-item {
            background: #f0f9ff; border: 3px solid #bae6fd; padding: 15px 20px; border-radius: 12px;
            text-align: center; position: relative; cursor: pointer; user-select: none;
            font-size: 1.3rem; font-weight: bold; color: #1e3a8a;
            display: flex; align-items: center; justify-content: center;
        }
        .match-item.right { text-align: left; justify-content: flex-start; font-size: 1.1rem; font-weight: normal; color: #333; }
        
        .dot {
            width: 24px; height: 24px; background: #3b82f6; border-radius: 50%;
            position: absolute; top: 50%; transform: translateY(-50%);
            cursor: crosshair; transition: transform 0.2s, background-color 0.2s;
            box-shadow: 0 0 8px rgba(0,0,0,0.4);
        }
        .dot:active, .dot.active { transform: translateY(-50%) scale(1.5); background: #f59e0b; }
        /* 점을 상자 바깥으로 충분히 빼서 선 연결 여백 확보 */
        .match-item.left .dot { right: -40px; } 
        .match-item.right .dot { left: -40px; }
        
        #lines-svg {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            pointer-events: none; z-index: 100; /* 선이 맨 위로 오도록 */
            overflow: visible;
        }
        line { stroke: #3b82f6; stroke-width: 6; stroke-linecap: round; }
        line.correct { stroke: #10b981; }

        .word-bank { display: flex; flex-wrap: wrap; gap: 15px; justify-content: center; margin-bottom: 25px; min-height: 50px; }
        .draggable-word {
            background: #fde047; padding: 12px 20px; border-radius: 12px; font-weight: bold; font-size: 1.3rem;
            cursor: grab; user-select: none; box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            transition: transform 0.2s; touch-action: none;
        }
        .draggable-word:active { cursor: grabbing; transform: scale(1.1); }
        .sentence-box { background: #f3f4f6; padding: 20px; border-radius: 12px; margin-bottom: 20px; font-size: 1.4rem; line-height: 1.8; word-break: keep-all; }
        .drop-zone {
            display: inline-block; width: 110px; height: 45px; border: 3px dashed #9ca3af;
            border-radius: 8px; vertical-align: middle; background: white; transition: background 0.2s;
            text-align: center; line-height: 40px; font-weight: bold; color: #15803d;
        }
        .drop-zone.hover { background: #dcfce7; border-color: #22c55e; }
        .drop-zone.filled { border-style: solid; background: #fde047; border-color: #eab308; }

        /* 오답 흔들림 애니메이션 */
        @keyframes shake {
            0% { transform: translateX(0); }
            25% { transform: translateX(-10px) rotate(-5deg); background-color: #fca5a5; }
            50% { transform: translateX(10px) rotate(5deg); background-color: #fca5a5; }
            75% { transform: translateX(-10px) rotate(-5deg); background-color: #fca5a5; }
            100% { transform: translateX(0); }
        }
        .wrong-shake { animation: shake 0.5s ease-in-out; }

        #hud { position: absolute; top: 20px; left: 20px; z-index: 10; display: none; }
        .password-slot {
            display: inline-block; width: 60px; height: 60px; background: rgba(255,255,255,0.9);
            border: 4px solid #333; border-radius: 12px; text-align: center; line-height: 52px;
            font-size: 30px; font-weight: bold; margin-right: 15px; box-shadow: 0 4px 6px rgba(0,0,0,0.3);
        }
        #audio-toggle {
            position: absolute; top: 20px; right: 20px; z-index: 10;
            background: white; border-radius: 50%; width: 60px; height: 60px;
            font-size: 28px; cursor: pointer; border: none; box-shadow: 0 4px 6px rgba(0,0,0,0.3);
        }

        /* 상장 스타일 - 문장 한 줄 고정 및 크기 확장 */
        .certificate {
            border: 15px solid #d4af37; background: #fffdf0; padding: clamp(20px, 4vh, 50px); text-align: center;
            border-radius: 10px; position: relative; width: 100%; box-sizing: border-box;
            background-image: repeating-linear-gradient(45deg, rgba(212, 175, 55, 0.05) 0px, rgba(212, 175, 55, 0.05) 20px, transparent 20px, transparent 40px);
        }
        .cert-title { font-size: clamp(2.5rem, 5vh, 4rem); color: #b8860b; margin-bottom: clamp(15px, 3vh, 30px); text-shadow: 2px 2px 4px rgba(0,0,0,0.1); font-weight: bold; }
        .cert-content { line-height: 1.8; color: #333; margin-bottom: clamp(20px, 4vh, 40px); }
        .cert-content p { white-space: nowrap; margin-bottom: clamp(5px, 1vh, 10px); font-size: clamp(1rem, 2.3vw, 1.8rem); } 
        .stamp {
            position: absolute; bottom: 40px; right: 40px; width: 120px; height: 120px;
            border: 6px solid #dc2626; border-radius: 50%; color: #dc2626; font-size: 1.8rem;
            display: flex; align-items: center; justify-content: center; transform: rotate(-15deg);
            font-weight: bold; box-shadow: 0 0 15px rgba(220, 38, 38, 0.2);
        }
    </style>
</head>
<body>

    <!-- Sound Toggle -->
    <button id="audio-toggle">🔊</button>

    <!-- HUD (비밀번호 수집 현황) -->
    <div id="hud">
        <div class="password-slot" id="pw-1">?</div>
        <div class="password-slot" id="pw-2">?</div>
        <div class="password-slot" id="pw-3">?</div>
    </div>

    <!-- 3D Canvas -->
    <div id="game-container"></div>

    <!-- Start Screen -->
    <div id="start-screen" class="overlay active">
        <div class="game-panel text-center" style="max-width: 800px;">
            <h1 class="text-4xl md:text-5xl font-bold mb-4 md:mb-6 text-blue-600">스카이워크 탈출 게임</h1>
            <p class="text-xl md:text-2xl mb-6 md:mb-8 leading-relaxed text-gray-700">스카이워크를 건너며 문제를 풀고<br>자물쇠의 비밀번호를 알아내세요!<br>총 3개의 관문이 있습니다.</p>
            <button onclick="startGame()" class="bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-8 md:py-4 md:px-10 rounded-full text-2xl md:text-3xl transition transform hover:scale-105 shadow-lg">게임 시작</button>
        </div>
    </div>

    <!-- 메세지 팝업 -->
    <div id="message-screen" class="overlay">
        <div class="game-panel text-center" style="max-width: 700px;">
            <h2 id="message-title" class="text-4xl md:text-5xl font-bold mb-4 md:mb-6 text-green-600">성공!</h2>
            <p id="message-text" class="text-xl md:text-2xl mb-6 md:mb-8 text-gray-800"></p>
            <button onclick="closeMessage()" class="bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-8 rounded-full text-xl md:text-2xl shadow-md">계속하기</button>
        </div>
    </div>

    <!-- Stage 1 Screen: 선긋기 -->
    <div id="stage1-screen" class="overlay">
        <div class="game-panel">
            <h2 class="text-3xl md:text-4xl font-bold mb-4 md:mb-6 text-center text-blue-800">1단계: 낱말과 뜻 연결하기</h2>
            <p class="text-lg md:text-xl text-gray-600 text-center mb-4 md:mb-6">점과 점을 드래그하여 알맞은 뜻과 연결하세요.</p>
            <div class="match-container" id="s1-container">
                <svg id="lines-svg"></svg>
                <div class="match-col words" id="s1-left"></div>
                <div class="match-col meanings" id="s1-right"></div>
            </div>
        </div>
    </div>

    <!-- Stage 2 Screen: 객관식 -->
    <div id="stage2-screen" class="overlay">
        <div class="game-panel text-center" style="max-width: 800px;">
            <h2 class="text-3xl md:text-4xl font-bold mb-4 md:mb-6 text-blue-800">2단계: 알맞은 낱말 고르기</h2>
            <p class="text-lg md:text-xl text-gray-600 mb-6 md:mb-8">다음 뜻에 맞는 올바른 낱말을 고르세요. (<span id="s2-progress">1/3</span>)</p>
            <div id="s2-question" class="text-xl md:text-2xl font-bold bg-blue-100 p-4 md:p-6 rounded-2xl mb-6 md:mb-8 text-blue-900 shadow-inner"></div>
            <div id="s2-options" class="flex flex-col gap-3 md:gap-5"></div>
        </div>
    </div>

    <!-- Stage 3 Screen: 빈칸 채우기 -->
    <div id="stage3-screen" class="overlay">
        <div class="game-panel">
            <h2 class="text-3xl md:text-4xl font-bold mb-4 md:mb-6 text-center text-blue-800">3단계: 문장 완성하기</h2>
            <p class="text-lg md:text-xl text-gray-600 text-center mb-4 md:mb-8">아래 낱말을 드래그하여 알맞은 빈칸에 쏙 넣어보세요.</p>
            <div class="word-bank" id="s3-word-bank"></div>
            <div id="s3-sentences"></div>
        </div>
    </div>

    <!-- Final Stage Screen: 비밀번호 입력 -->
    <div id="final-screen" class="overlay">
        <div class="game-panel text-center" style="max-width: 700px;">
            <h2 class="text-4xl md:text-5xl font-bold mb-4 md:mb-6 text-red-600">🔒 굳게 닫힌 철문</h2>
            <p class="text-xl md:text-2xl mb-6 md:mb-8 text-gray-800">지금까지 모은 비밀번호 3자리를 순서대로 입력하세요!</p>
            <div class="flex justify-center gap-4 md:gap-6 mb-8 md:mb-10">
                <input type="number" id="final-pw1" class="w-20 h-20 md:w-24 md:h-24 text-center text-4xl md:text-5xl font-bold border-4 border-gray-400 rounded-2xl bg-gray-50 focus:bg-white focus:border-red-500 outline-none" min="0" max="9">
                <input type="number" id="final-pw2" class="w-20 h-20 md:w-24 md:h-24 text-center text-4xl md:text-5xl font-bold border-4 border-gray-400 rounded-2xl bg-gray-50 focus:bg-white focus:border-red-500 outline-none" min="0" max="9">
                <input type="number" id="final-pw3" class="w-20 h-20 md:w-24 md:h-24 text-center text-4xl md:text-5xl font-bold border-4 border-gray-400 rounded-2xl bg-gray-50 focus:bg-white focus:border-red-500 outline-none" min="0" max="9">
            </div>
            <button onclick="checkFinalPassword()" class="bg-red-500 hover:bg-red-600 text-white font-bold py-3 px-8 md:py-4 md:px-10 rounded-full text-2xl md:text-3xl shadow-lg">열기</button>
        </div>
    </div>

    <!-- 상장 화면 (성공 엔딩) -->
    <div id="certificate-screen" class="overlay">
        <div class="game-panel" style="max-width: 950px;">
            <div class="certificate">
                <h1 class="cert-title">상 장</h1>
                <div class="cert-content">
                    <p>위 학생은 스카이워크의 아찔한 높이에도 불구하고</p>
                    <p>뛰어난 국어 실력과 지혜를 발휘하여</p>
                    <p>모든 문제를 훌륭하게 해결하고 무사히 탈출하였기에</p>
                    <p>이 상장을 수여합니다.</p>
                    <br>
                    <p class="font-bold text-3xl md:text-4xl text-blue-800">참 잘했습니다!</p>
                </div>
                <div class="stamp">참 잘함</div>
            </div>
            <div class="text-center mt-6 md:mt-8">
                <button onclick="location.reload()" class="bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-8 md:px-10 rounded-full text-xl md:text-2xl shadow-lg">다시 하기</button>
            </div>
        </div>
    </div>

    <script>
        const GAME_DATA = {
            passwords: [], // 시작 시 랜덤 생성
            s1: {
                words: ["인기", "절벽", "주변", "구조물", "전망대"],
                meanings: [
                    "어떤 대상에 쏠리는 대중의 높은 관심이나 좋아하는 기운.",
                    "바위가 깎아 세운 것처럼 아주 높이 솟아 있는 험한 낭떠러지.",
                    "어떤 대상의 둘레.",
                    "일정한 설계에 따라 여러 가지 재료를 얽어서 만든 물건.",
                    "멀리 내다볼 수 있도록 높이 만든 대."
                ]
            },
            s2: [
                { q: "전체의 상태나 성질을 어느 하나로 잘 나타내다.", a: "대표하다", options: ["대표하다", "아찔하다", "짜릿하다"] },
                { q: "자기 정신이 아득하고 조금 어지럽다.", a: "아찔하다", options: ["대표하다", "아찔하다", "짜릿하다"] },
                { q: "조금 자린 듯하다. ‘자릿하다’보다 센 느낌을 준다.", a: "짜릿하다", options: ["대표하다", "아찔하다", "짜릿하다"] }
            ],
            s3: {
                words: ["감상", "절벽", "인기", "주변", "풍경", "전망대"],
                sentences: [
                    { t1: "밤하늘에 크고 밝게 뜬 보름달 ", t2: "으로 반짝반짝 빛나는 작은 별들이 모여 있는 것을 보았어요.", a: "주변" },
                    { t1: "따뜻한 봄이 찾아와 노란 유채꽃이 들판에 가득 피어난 ", t2: "을 사진기로 찰칵 찍어 간직했습니다.", a: "풍경" },
                    { t1: "우리 학교 도서관에 새로 들어온 과학 만화책은 서로 빌리려고 줄을 설 정도로 ", t2: "가 최고랍니다.", a: "인기" }
                ]
            }
        };

        const state = {
            currentStage: 0,
            collectedPasswords: [],
            isWalking: false,
            // 웨이포인트 간격을 15로 좁혀서 이동 시간 단축 (약 2초)
            waypoints: [-15, -30, -45, -60, -75], 
            s1_lines: [],
            s2_currentIndex: 0,
            s3_solvedCount: 0
        };

        let onMessageClose = null;

        // --- 오디오 시스템 (Tone.js) ---
        let isMuted = false;
        const AudioSystem = {
            synths: {},
            init: async function() {
                await Tone.start();
                this.synths.bgm = new Tone.PolySynth(Tone.Synth).toDestination();
                this.synths.bgm.volume.value = -15;
                this.synths.fx = new Tone.Synth().toDestination();
                this.synths.step = new Tone.MembraneSynth().toDestination();
                this.synths.step.volume.value = -10;
                
                const notes = ["C4", "E4", "G4", "B4", "A4", "F4"];
                let idx = 0;
                Tone.Transport.scheduleRepeat(time => {
                    if(!isMuted) this.synths.bgm.triggerAttackRelease(notes[idx % notes.length], "8n", time);
                    idx++;
                }, "4n");
                Tone.Transport.start();
            },
            playStep: function() {
                if (isMuted || !this.synths.step) return;
                this.synths.step.triggerAttackRelease("C2", "32n");
            },
            playFall: function() {
                if (isMuted || !this.synths.step) return;
                const now = Tone.now();
                this.synths.step.triggerAttackRelease("G1", "16n", now);
                this.synths.step.triggerAttackRelease("C1", "8n", now + 0.1);
            },
            playUnlock: function() {
                if (isMuted || !this.synths.fx) return;
                this.synths.fx.triggerAttackRelease("C6", "16n");
            },
            playCorrect: function() {
                if (isMuted || !this.synths.fx) return;
                const now = Tone.now();
                this.synths.fx.triggerAttackRelease("C5", "8n", now);
                this.synths.fx.triggerAttackRelease("E5", "8n", now + 0.1);
                this.synths.fx.triggerAttackRelease("G5", "4n", now + 0.2);
            },
            playWrong: function() {
                if (isMuted || !this.synths.fx) return;
                const now = Tone.now();
                this.synths.fx.triggerAttackRelease("Eb3", "8n", now);
                this.synths.fx.triggerAttackRelease("C3", "4n", now + 0.15);
            },
            playTriumph: function() {
                if (isMuted || !this.synths.fx) return;
                const now = Tone.now();
                this.synths.fx.triggerAttackRelease("G4", "8n", now);
                this.synths.fx.triggerAttackRelease("C5", "8n", now + 0.15);
                this.synths.fx.triggerAttackRelease("E5", "8n", now + 0.3);
                this.synths.fx.triggerAttackRelease("G5", "2n", now + 0.45);
            }
        };

        document.getElementById('audio-toggle').addEventListener('click', (e) => {
            isMuted = !isMuted;
            e.target.innerText = isMuted ? '🔇' : '🔊';
        });

        function shuffle(array) {
            let currentIndex = array.length, randomIndex;
            while (currentIndex != 0) {
                randomIndex = Math.floor(Math.random() * currentIndex);
                currentIndex--;
                [array[currentIndex], array[randomIndex]] = [array[randomIndex], array[currentIndex]];
            }
            return array;
        }

        const scene = new THREE.Scene();
        scene.fog = new THREE.FogExp2(0x87CEEB, 0.015);
        
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        document.getElementById('game-container').appendChild(renderer.domElement);

        const ambientLight = new THREE.AmbientLight(0xffffff, 0.8);
        scene.add(ambientLight);
        const dirLight = new THREE.DirectionalLight(0xffffff, 0.6);
        dirLight.position.set(10, 20, 10);
        dirLight.castShadow = true;
        scene.add(dirLight);

        // 스카이워크 다리
        const pathMat = new THREE.MeshStandardMaterial({ 
            color: 0x88ccff, transparent: true, opacity: 0.6, roughness: 0.1, metalness: 0.8 
        });
        const pathGeo = new THREE.BoxGeometry(6, 0.5, 150);
        const path = new THREE.Mesh(pathGeo, pathMat);
        path.position.set(0, -0.5, -45);
        path.receiveShadow = true;
        scene.add(path);

        // 난간
        const railMat = new THREE.MeshStandardMaterial({ color: 0xcccccc, metalness: 0.8 });
        const railL = new THREE.Mesh(new THREE.BoxGeometry(0.2, 1.5, 150), railMat);
        railL.position.set(-3, 0.5, -45);
        scene.add(railL);
        const railR = new THREE.Mesh(new THREE.BoxGeometry(0.2, 1.5, 150), railMat);
        railR.position.set(3, 0.5, -45);
        scene.add(railR);

        // 깊은 숲 바닥과 나무 생성
        const groundGeo = new THREE.PlaneGeometry(600, 600);
        const groundMat = new THREE.MeshStandardMaterial({ color: 0x2d4c1e, roughness: 1 });
        const ground = new THREE.Mesh(groundGeo, groundMat);
        ground.rotation.x = -Math.PI / 2;
        ground.position.y = -80;
        scene.add(ground);

        const treeGeo = new THREE.ConeGeometry(3, 12, 4);
        const trunkGeo = new THREE.CylinderGeometry(0.8, 0.8, 4);
        const treeMat = new THREE.MeshStandardMaterial({ color: 0x1B4D3E, flatShading: true });
        const trunkMat = new THREE.MeshStandardMaterial({ color: 0x4A3728 });
        
        for (let i = 0; i < 200; i++) {
            const x = (Math.random() - 0.5) * 450;
            const z = (Math.random() - 0.5) * 450;
            const trunk = new THREE.Mesh(trunkGeo, trunkMat);
            trunk.position.set(x, -78, z);
            scene.add(trunk);
            const leaves = new THREE.Mesh(treeGeo, treeMat);
            leaves.position.set(x, -70, z);
            scene.add(leaves);
        }

        // 구름
        const cloudGeo = new THREE.DodecahedronGeometry(5, 0);
        const cloudMat = new THREE.MeshStandardMaterial({ color: 0xffffff, flatShading: true, transparent: true, opacity: 0.9 });
        for(let i = 0; i < 60; i++) {
            const cloud = new THREE.Mesh(cloudGeo, cloudMat);
            cloud.position.set(
                (Math.random() - 0.5) * 300,
                -40 + Math.random() * 50, 
                (Math.random() - 0.5) * 300
            );
            cloud.scale.set(1 + Math.random() * 2, 0.4 + Math.random() * 0.4, 1 + Math.random() * 2);
            scene.add(cloud);
        }

        // 캐릭터 (마인크래프트 풍)
        const character = new THREE.Group();
        const headMat = new THREE.MeshStandardMaterial({ color: 0xffccaa });
        const bodyMat = new THREE.MeshStandardMaterial({ color: 0x3b82f6 });
        const pantsMat = new THREE.MeshStandardMaterial({ color: 0x1e3a8a });

        const head = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.8, 0.8), headMat);
        head.position.y = 2.4;
        character.add(head);

        const torso = new THREE.Mesh(new THREE.BoxGeometry(0.8, 1.2, 0.4), bodyMat);
        torso.position.y = 1.4;
        character.add(torso);

        const leftArm = new THREE.Mesh(new THREE.BoxGeometry(0.3, 1.2, 0.3), headMat);
        leftArm.position.set(-0.6, 1.4, 0);
        character.add(leftArm);

        const rightArm = new THREE.Mesh(new THREE.BoxGeometry(0.3, 1.2, 0.3), headMat);
        rightArm.position.set(0.6, 1.4, 0);
        character.add(rightArm);

        const leftLeg = new THREE.Mesh(new THREE.BoxGeometry(0.35, 1.2, 0.35), pantsMat);
        leftLeg.position.set(-0.22, 0.4, 0);
        character.add(leftLeg);

        const rightLeg = new THREE.Mesh(new THREE.BoxGeometry(0.35, 1.2, 0.35), pantsMat);
        rightLeg.position.set(0.22, 0.4, 0);
        character.add(rightLeg);

        // 180도 회전하여 앞을 보게 함
        character.rotation.y = Math.PI;
        character.position.set(0, 0, state.waypoints[0]);
        scene.add(character);

        // 장애물 상자 (불투명 빨간색)
        const obstacleMat = new THREE.MeshStandardMaterial({ color: 0xe53e3e, roughness: 0.7, metalness: 0.2 });
        const obstacles = [];
        const fallingObstacles = [];
        
        for(let i=1; i<=3; i++) {
            const obs = new THREE.Mesh(new THREE.BoxGeometry(2, 2, 2), obstacleMat);
            // 장애물은 도착 지점보다 1.5 앞에 배치하여 길을 막음
            obs.position.set(0, 1, state.waypoints[i] - 1.5); 
            obs.castShadow = true;
            scene.add(obs);
            obstacles.push(obs);
        }

        // 마지막 철문
        const doorGroup = new THREE.Group();
        const doorGeo = new THREE.BoxGeometry(6, 4, 0.5);
        const doorMat = new THREE.MeshStandardMaterial({ color: 0x4a5568, metalness: 0.9, roughness: 0.3 });
        const doorLeft = new THREE.Mesh(doorGeo, doorMat);
        doorLeft.position.set(-3, 2, state.waypoints[4] - 2);
        const doorRight = new THREE.Mesh(doorGeo, doorMat);
        doorRight.position.set(3, 2, state.waypoints[4] - 2);
        doorGroup.add(doorLeft);
        doorGroup.add(doorRight);
        scene.add(doorGroup);

        // 눈부신 황금 자물쇠 생성
        const padlockGroup = new THREE.Group();
        const lockBodyMat = new THREE.MeshStandardMaterial({ color: 0xFFD700, metalness: 1.0, roughness: 0.1 }); 
        const lockBody = new THREE.Mesh(new THREE.BoxGeometry(1.2, 0.8, 0.4), lockBodyMat);
        padlockGroup.add(lockBody);
        
        const shackleMat = new THREE.MeshStandardMaterial({ color: 0xFFD700, metalness: 1.0, roughness: 0.1 }); 
        const shackle = new THREE.Mesh(new THREE.TorusGeometry(0.4, 0.1, 16, 32, Math.PI), shackleMat);
        shackle.position.y = 0.4;
        padlockGroup.add(shackle);

        // 두 문이 만나는 중앙 위치에 자물쇠 배치
        padlockGroup.position.set(0, 2, state.waypoints[4] - 1.7);
        scene.add(padlockGroup);

        // 카메라를 항상 캐릭터의 등 뒤에서 바라보도록 업데이트하는 핵심 함수
        function updateCamera() {
            camera.position.x = character.position.x;
            camera.position.y = character.position.y + 4; // 머리 위쪽
            camera.position.z = character.position.z + 6; // 등 뒤 (z축 양의 방향)
            camera.lookAt(character.position.x, character.position.y + 1, character.position.z - 5); // 앞을 봄
        }
        
        updateCamera(); // 초기 위치 설정

        let clock = new THREE.Clock();
        let walkTime = 0;
        let unlockAnimStage = 0; // 0: 잠김, 1: 고리 열림, 2: 전체 떨어짐
        let doorOpenAnim = false;

        function animate() {
            requestAnimationFrame(animate);
            let delta = clock.getDelta();

            // 빨간 장애물 낙하 연출
            for (let i = fallingObstacles.length - 1; i >= 0; i--) {
                let obs = fallingObstacles[i];
                obs.position.y -= 5 * delta;
                obs.rotation.x += 2 * delta;
                obs.rotation.z += 1.5 * delta;
                if (obs.position.y < -30) {
                    scene.remove(obs);
                    fallingObstacles.splice(i, 1);
                }
            }

            // 황금 자물쇠 열림 -> 낙하 연출
            if (unlockAnimStage === 1) {
                shackle.position.y += 2 * delta; // 고리가 철컥 위로 열림
                if (shackle.position.y > 0.9) {
                    unlockAnimStage = 2; // 다 열리면 떨어지기 시작
                }
            } else if (unlockAnimStage === 2) {
                padlockGroup.position.y -= 4 * delta; // 자물쇠 전체 낙하
                padlockGroup.rotation.x += 3 * delta;
                if (padlockGroup.position.y < -15) {
                    scene.remove(padlockGroup);
                    unlockAnimStage = 0;
                    doorOpenAnim = true; // 자물쇠 떨어지면 문 열림 시작
                }
            }

            // 철문 열림 연출
            if (doorOpenAnim) {
                if (doorLeft.position.x > -6) {
                    doorLeft.position.x -= 2 * delta;
                    doorRight.position.x += 2 * delta;
                } else {
                    doorOpenAnim = false;
                    setTimeout(showCertificate, 1000);
                }
            }

            if (state.isWalking) {
                // 초당 이동 속도를 조절하여 정확히 목표 거리에 도달하도록 세팅 (속도 = 거리15 / 시간2초 = 7.5)
                const moveSpeed = 7.5;
                character.position.z -= moveSpeed * delta;
                updateCamera(); // 카메라 추적
                    
                // 걷기 애니메이션 및 발소리
                walkTime += delta * 15;
                leftArm.rotation.x = Math.sin(walkTime) * 0.5;
                rightArm.rotation.x = -Math.sin(walkTime) * 0.5;
                leftLeg.rotation.x = -Math.sin(walkTime) * 0.5;
                rightLeg.rotation.x = Math.sin(walkTime) * 0.5;
                if (Math.sin(walkTime) > 0.9 && Math.sin(walkTime - 0.1) <= 0.9) AudioSystem.playStep();
                if (-Math.sin(walkTime) > 0.9 && -Math.sin(walkTime - 0.1) <= 0.9) AudioSystem.playStep();

                // 현재 단계의 장애물/도착 지점 체크
                let targetZ = state.waypoints[state.currentStage];
                // 1~3단계는 장애물 앞(-1.5)이 아니라 도착 웨이포인트(targetZ)에서 멈추도록 조정됨 
                // 위에서 박스 위치를 targetZ - 1.5로 옮겼으므로, 캐릭터가 targetZ까지 오면 박스 바로 앞에 멈춤
                
                if (character.position.z <= targetZ) {
                    character.position.z = targetZ; // 정확한 위치 보정
                    stopWalking();
                    
                    if (state.currentStage >= 1 && state.currentStage <= 3) {
                        triggerStage(state.currentStage);
                    } else if (state.currentStage === 4) {
                        initFinalStage();
                    }
                }
            }
            renderer.render(scene, camera);
        }
        animate();

        function stopWalking() {
            state.isWalking = false;
            updateCamera(); // 멈출 때도 카메라 위치 정렬
            leftArm.rotation.x = 0; rightArm.rotation.x = 0;
            leftLeg.rotation.x = 0; rightLeg.rotation.x = 0;
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        function startGame() {
            GAME_DATA.passwords = [
                Math.floor(Math.random() * 10).toString(),
                Math.floor(Math.random() * 10).toString(),
                Math.floor(Math.random() * 10).toString()
            ];
            document.getElementById('start-screen').classList.remove('active');
            document.getElementById('hud').style.display = 'block';
            Tone.start();
            AudioSystem.init();
            walkToNext();
        }

        function walkToNext() {
            state.currentStage++;
            state.isWalking = true;
        }

        function showScreen(id) {
            document.querySelectorAll('.overlay').forEach(el => el.classList.remove('active'));
            if (id) document.getElementById(id).classList.add('active');
        }

        function showMessage(title, text, callback) {
            document.getElementById('message-title').innerText = title;
            document.getElementById('message-text').innerText = text;
            onMessageClose = callback;
            showScreen('message-screen');
        }

        function closeMessage() {
            showScreen(null);
            if (onMessageClose) {
                onMessageClose();
                onMessageClose = null;
            }
        }

        function updateHUD() {
            for(let i=0; i<state.collectedPasswords.length; i++) {
                document.getElementById(`pw-${i+1}`).innerText = state.collectedPasswords[i];
            }
        }

        function completeStage(stageNum) {
            AudioSystem.playCorrect();
            showScreen(null);
            const pw = GAME_DATA.passwords[stageNum - 1];
            state.collectedPasswords.push(pw);
            
            showMessage(`🎉 ${stageNum}단계 성공!`, `자물쇠 비밀번호 [ ${pw} ] 를 획득했습니다!`, () => {
                updateHUD();
                
                // 장애물 낙하 연출 시작
                const obs = obstacles[stageNum - 1];
                if (obs) {
                    fallingObstacles.push(obs);
                    AudioSystem.playFall();
                }
                
                // 장애물이 떨어질 시간을 1초 주고 다시 전진
                setTimeout(walkToNext, 1000);
            });
        }

        function triggerStage(stageNum) {
            if (stageNum === 1) initStage1();
            else if (stageNum === 2) initStage2();
            else if (stageNum === 3) initStage3();
        }

        let drawLine = null;
        let startDot = null;
        
        function initStage1() {
            showScreen('stage1-screen');
            const leftCol = document.getElementById('s1-left');
            const rightCol = document.getElementById('s1-right');
            const svg = document.getElementById('lines-svg');
            leftCol.innerHTML = ''; rightCol.innerHTML = ''; svg.innerHTML = '';
            state.s1_lines = [];

            let words = shuffle([...GAME_DATA.s1.words]);
            let meanings = shuffle([...GAME_DATA.s1.meanings]);

            words.forEach(w => {
                const el = document.createElement('div');
                el.className = 'match-item left'; el.innerText = w; el.dataset.val = w;
                const dot = document.createElement('div'); dot.className = 'dot'; dot.dataset.side = 'left';
                el.appendChild(dot); leftCol.appendChild(el);
            });

            meanings.forEach(m => {
                const el = document.createElement('div');
                el.className = 'match-item right'; el.innerText = m;
                const origIdx = GAME_DATA.s1.meanings.indexOf(m);
                el.dataset.val = GAME_DATA.s1.words[origIdx]; 
                const dot = document.createElement('div'); dot.className = 'dot'; dot.dataset.side = 'right';
                el.appendChild(dot); rightCol.appendChild(el);
            });

            document.querySelectorAll('.dot').forEach(dot => {
                dot.addEventListener('pointerdown', (e) => {
                    e.preventDefault();
                    startDot = dot;
                    startDot.classList.add('active');
                    const rect = startDot.getBoundingClientRect();
                    const svgRect = svg.getBoundingClientRect();
                    
                    drawLine = document.createElementNS('http://www.w3.org/2000/svg', 'line');
                    const x = rect.left + rect.width/2 - svgRect.left;
                    const y = rect.top + rect.height/2 - svgRect.top;
                    drawLine.setAttribute('x1', x); drawLine.setAttribute('y1', y);
                    drawLine.setAttribute('x2', x); drawLine.setAttribute('y2', y);
                    svg.appendChild(drawLine);
                    dot.setPointerCapture(e.pointerId);
                });
                
                dot.addEventListener('pointermove', (e) => {
                    if(!drawLine) return;
                    const svgRect = svg.getBoundingClientRect();
                    drawLine.setAttribute('x2', e.clientX - svgRect.left);
                    drawLine.setAttribute('y2', e.clientY - svgRect.top);
                });

                dot.addEventListener('pointerup', (e) => {
                    if(!drawLine) return;
                    startDot.classList.remove('active');
                    dot.releasePointerCapture(e.pointerId);
                    
                    const elemBelow = document.elementFromPoint(e.clientX, e.clientY);
                    const endDot = elemBelow ? (elemBelow.classList.contains('dot') ? elemBelow : null) : null;

                    if (endDot && endDot !== startDot && endDot.dataset.side !== startDot.dataset.side) {
                        const val1 = startDot.parentElement.dataset.val;
                        const val2 = endDot.parentElement.dataset.val;
                        
                        if (val1 === val2) {
                            AudioSystem.playCorrect();
                            const rect = endDot.getBoundingClientRect();
                            const svgRect = svg.getBoundingClientRect();
                            drawLine.setAttribute('x2', rect.left + rect.width/2 - svgRect.left);
                            drawLine.setAttribute('y2', rect.top + rect.height/2 - svgRect.top);
                            drawLine.classList.add('correct');
                            
                            startDot.style.pointerEvents = 'none';
                            endDot.style.pointerEvents = 'none';
                            state.s1_lines.push(val1);
                            
                            if (state.s1_lines.length === 5) {
                                setTimeout(() => completeStage(1), 1000);
                            }
                        } else {
                            AudioSystem.playWrong();
                            drawLine.remove();
                        }
                    } else {
                        drawLine.remove();
                    }
                    drawLine = null;
                    startDot = null;
                });
            });
        }

        function initStage2() {
            state.s2_currentIndex = 0;
            showScreen('stage2-screen');
            loadS2Question();
        }

        function loadS2Question() {
            const qData = GAME_DATA.s2[state.s2_currentIndex];
            document.getElementById('s2-progress').innerText = `${state.s2_currentIndex + 1}/3`;
            document.getElementById('s2-question').innerText = qData.q;
            
            const optsContainer = document.getElementById('s2-options');
            optsContainer.innerHTML = '';
            
            let options = shuffle([...qData.options]);
            options.forEach(opt => {
                const btn = document.createElement('button');
                btn.className = 'w-full bg-white border-4 border-blue-200 hover:bg-blue-50 text-xl md:text-2xl font-bold py-3 md:py-5 rounded-xl transition shadow-sm';
                btn.innerText = opt;
                btn.onclick = () => checkS2Answer(opt, qData.a, btn);
                optsContainer.appendChild(btn);
            });
        }

        function checkS2Answer(selected, correct, btn) {
            if (selected === correct) {
                AudioSystem.playCorrect();
                btn.classList.add('bg-green-200', 'border-green-500');
                state.s2_currentIndex++;
                setTimeout(() => {
                    if (state.s2_currentIndex >= 3) completeStage(2);
                    else loadS2Question();
                }, 500);
            } else {
                AudioSystem.playWrong();
                btn.classList.add('wrong-shake');
                setTimeout(() => btn.classList.remove('wrong-shake'), 500);
            }
        }

        let activeWord = null;
        let dragOffset = {x: 0, y: 0};
        
        function initStage3() {
            state.s3_solvedCount = 0;
            showScreen('stage3-screen');
            
            const bank = document.getElementById('s3-word-bank');
            const sents = document.getElementById('s3-sentences');
            bank.innerHTML = ''; sents.innerHTML = '';

            let words = shuffle([...GAME_DATA.s3.words]);
            words.forEach(w => {
                const el = document.createElement('div');
                el.className = 'draggable-word';
                el.innerText = w;
                el.dataset.word = w;
                el.addEventListener('pointerdown', startWordDrag);
                bank.appendChild(el);
            });

            GAME_DATA.s3.sentences.forEach((s, idx) => {
                const box = document.createElement('div');
                box.className = 'sentence-box';
                box.innerHTML = `${s.t1} <span class="drop-zone" data-ans="${s.a}" id="dz-${idx}"></span> ${s.t2}`;
                sents.appendChild(box);
            });
        }

        function startWordDrag(e) {
            e.preventDefault();
            activeWord = e.target;
            const rect = activeWord.getBoundingClientRect();
            dragOffset.x = e.clientX - rect.left;
            dragOffset.y = e.clientY - rect.top;
            
            activeWord.style.position = 'fixed';
            activeWord.style.zIndex = '1000';
            activeWord.style.left = (e.clientX - dragOffset.x) + 'px';
            activeWord.style.top = (e.clientY - dragOffset.y) + 'px';
            document.body.appendChild(activeWord);

            document.addEventListener('pointermove', moveWordDrag);
            document.addEventListener('pointerup', endWordDrag);
        }

        function moveWordDrag(e) {
            if(!activeWord) return;
            activeWord.style.left = (e.clientX - dragOffset.x) + 'px';
            activeWord.style.top = (e.clientY - dragOffset.y) + 'px';
            
            document.querySelectorAll('.drop-zone:not(.filled)').forEach(dz => dz.classList.remove('hover'));
            activeWord.style.visibility = 'hidden';
            const elemBelow = document.elementFromPoint(e.clientX, e.clientY);
            activeWord.style.visibility = 'visible';
            
            if(elemBelow && elemBelow.classList.contains('drop-zone') && !elemBelow.classList.contains('filled')) {
                elemBelow.classList.add('hover');
            }
        }

        function endWordDrag(e) {
            if(!activeWord) return;
            document.removeEventListener('pointermove', moveWordDrag);
            document.removeEventListener('pointerup', endWordDrag);
            
            activeWord.style.visibility = 'hidden';
            const elemBelow = document.elementFromPoint(e.clientX, e.clientY);
            activeWord.style.visibility = 'visible';
            
            let droppedOnZone = (elemBelow && elemBelow.classList.contains('drop-zone') && !elemBelow.classList.contains('filled')) ? elemBelow : null;

            if (droppedOnZone) {
                if (droppedOnZone.dataset.ans === activeWord.dataset.word) {
                    AudioSystem.playCorrect();
                    droppedOnZone.innerText = activeWord.dataset.word;
                    droppedOnZone.classList.add('filled');
                    droppedOnZone.classList.remove('hover');
                    activeWord.remove();
                    state.s3_solvedCount++;
                    if (state.s3_solvedCount === 3) {
                        setTimeout(() => completeStage(3), 1000);
                    }
                } else {
                    AudioSystem.playWrong();
                    droppedOnZone.classList.remove('hover');
                    returnWordToBank(activeWord, true);
                }
            } else {
                returnWordToBank(activeWord, false);
            }
            activeWord = null;
        }

        function returnWordToBank(wordEl, playShakeAnim) {
            wordEl.style.position = 'static';
            wordEl.style.zIndex = 'auto';
            document.getElementById('s3-word-bank').appendChild(wordEl);
            if(playShakeAnim) {
                wordEl.classList.add('wrong-shake');
                setTimeout(() => wordEl.classList.remove('wrong-shake'), 500);
            }
        }

        function initFinalStage() {
            showScreen('final-screen');
        }

        function checkFinalPassword() {
            const p1 = document.getElementById('final-pw1').value;
            const p2 = document.getElementById('final-pw2').value;
            const p3 = document.getElementById('final-pw3').value;
            
            if (p1 === GAME_DATA.passwords[0] && p2 === GAME_DATA.passwords[1] && p3 === GAME_DATA.passwords[2]) {
                showScreen(null);
                document.getElementById('hud').style.display = 'none';
                AudioSystem.playUnlock(); // 철컥 소리
                unlockAnimStage = 1; // 자물쇠 열림 애니메이션 시작 (이후 문 열림 트리거)
            } else {
                AudioSystem.playWrong();
                const inputs = [document.getElementById('final-pw1'), document.getElementById('final-pw2'), document.getElementById('final-pw3')];
                inputs.forEach(i => {
                    i.classList.add('wrong-shake');
                    setTimeout(() => i.classList.remove('wrong-shake'), 500);
                });
            }
        }

        function showCertificate() {
            showScreen('certificate-screen');
            AudioSystem.playTriumph();
            
            var duration = 4000;
            var end = Date.now() + duration;

            (function frame() {
                confetti({
                    particleCount: 7,
                    angle: 60,
                    spread: 60,
                    origin: { x: 0 },
                    colors: ['#26ccff', '#a25afd', '#ff5e7e', '#88ff5a', '#fcff42', '#ffa62d', '#ff36ff']
                });
                confetti({
                    particleCount: 7,
                    angle: 120,
                    spread: 60,
                    origin: { x: 1 },
                    colors: ['#26ccff', '#a25afd', '#ff5e7e', '#88ff5a', '#fcff42', '#ffa62d', '#ff36ff']
                });

                if (Date.now() < end) {
                    requestAnimationFrame(frame);
                }
            }());
        }

    </script>
</body>
</html>
