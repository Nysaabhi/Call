<!DOCTYPE html>
<html>
<head>
    <title>Internet Dialer</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }

        body {
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background: #f0f2f5;
        }

        .dialer {
            background: white;
            border-radius: 20px;
            padding: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            width: 300px;
        }

        .screen {
            background: #f8f9fa;
            border-radius: 10px;
            padding: 15px;
            margin-bottom: 20px;
        }

        #userId {
            width: 100%;
            padding: 10px;
            font-size: 18px;
            border: 1px solid #e0e0e0;
            border-radius: 5px;
            margin-bottom: 10px;
        }

        #targetId {
            width: 100%;
            padding: 10px;
            font-size: 24px;
            border: none;
            background: transparent;
            text-align: center;
            letter-spacing: 1px;
        }

        .keypad {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            gap: 10px;
            margin-bottom: 20px;
        }

        .key {
            background: #fff;
            border: 1px solid #e0e0e0;
            border-radius: 50%;
            width: 60px;
            height: 60px;
            display: flex;
            justify-content: center;
            align-items: center;
            cursor: pointer;
            font-size: 24px;
            transition: all 0.2s;
        }

        .key:hover {
            background: #f0f0f0;
        }

        .key:active {
            background: #e0e0e0;
            transform: scale(0.95);
        }

        .actions {
            display: flex;
            justify-content: center;
            gap: 10px;
        }

        .call-btn, .hangup-btn {
            padding: 15px 30px;
            border: none;
            border-radius: 25px;
            cursor: pointer;
            font-size: 16px;
            transition: all 0.2s;
        }

        .call-btn {
            background: #4CAF50;
            color: white;
        }

        .hangup-btn {
            background: #f44336;
            color: white;
            display: none;
        }

        .status {
            text-align: center;
            margin-top: 10px;
            color: #666;
            font-size: 14px;
        }

        #incomingCall {
            display: none;
            text-align: center;
            margin-top: 10px;
        }
    </style>
</head>
<body>
    <div class="dialer">
        <div class="screen">
            <input type="text" id="userId" placeholder="Enter your ID">
            <input type="text" id="targetId" placeholder="Enter caller ID" readonly>
        </div>
        <div class="keypad">
            <div class="key" onclick="addNumber('1')">1</div>
            <div class="key" onclick="addNumber('2')">2</div>
            <div class="key" onclick="addNumber('3')">3</div>
            <div class="key" onclick="addNumber('4')">4</div>
            <div class="key" onclick="addNumber('5')">5</div>
            <div class="key" onclick="addNumber('6')">6</div>
            <div class="key" onclick="addNumber('7')">7</div>
            <div class="key" onclick="addNumber('8')">8</div>
            <div class="key" onclick="addNumber('9')">9</div>
            <div class="key" onclick="addNumber('*')">*</div>
            <div class="key" onclick="addNumber('0')">0</div>
            <div class="key" onclick="addNumber('#')">#</div>
        </div>
        <div class="actions">
            <button class="call-btn" onclick="startCall()">Call</button>
            <button class="hangup-btn" onclick="endCall()">End</button>
        </div>
        <div id="incomingCall">
            <p>Incoming call from <span id="callerId"></span></p>
            <button onclick="answerCall()">Answer</button>
            <button onclick="rejectCall()">Reject</button>
        </div>
        <div class="status" id="callStatus"></div>
    </div>

    <script>
        let pc = null;
        let localStream = null;
        let socket = null;
        let userId = null;

        // Connect to signaling server
        function connectSignaling() {
            // Replace with your WebSocket server URL
            socket = new WebSocket('wss://your-signaling-server.com');

            socket.onopen = () => {
                const id = document.getElementById('userId').value;
                socket.send(JSON.stringify({
                    type: 'register',
                    userId: id
                }));
                userId = id;
                updateStatus('Connected');
            };

            socket.onmessage = async (event) => {
                const message = JSON.parse(event.data);
                handleSignalingMessage(message);
            };
        }

        // Handle incoming signaling messages
        async function handleSignalingMessage(message) {
            switch(message.type) {
                case 'offer':
                    showIncomingCall(message.from);
                    await handleOffer(message);
                    break;
                case 'answer':
                    await handleAnswer(message);
                    break;
                case 'candidate':
                    await handleCandidate(message);
                    break;
                case 'hangup':
                    endCall();
                    break;
            }
        }

        // Initialize WebRTC
        async function initializeWebRTC() {
            try {
                localStream = await navigator.mediaDevices.getUserMedia({ audio: true });
                
                const configuration = {
                    iceServers: [
                        { urls: 'stun:stun.l.google.com:19302' },
                        { urls: 'turn:your-turn-server.com', 
                          username: 'username', 
                          credential: 'password' }
                    ]
                };
                
                pc = new RTCPeerConnection(configuration);
                
                pc.onicecandidate = ({candidate}) => {
                    if (candidate) {
                        socket.send(JSON.stringify({
                            type: 'candidate',
                            candidate: candidate,
                            target: document.getElementById('targetId').value
                        }));
                    }
                };

                pc.ontrack = (event) => {
                    const remoteAudio = new Audio();
                    remoteAudio.srcObject = event.streams[0];
                    remoteAudio.play();
                };

                localStream.getTracks().forEach(track => {
                    pc.addTrack(track, localStream);
                });

                return true;
            } catch (error) {
                console.error('Error initializing WebRTC:', error);
                updateStatus('Error: Could not access microphone');
                return false;
            }
        }

        // Start call
        async function startCall() {
            const targetId = document.getElementById('targetId').value;
            if (!targetId) {
                updateStatus('Please enter caller ID');
                return;
            }

            if (!socket) {
                connectSignaling();
            }

            if (!await initializeWebRTC()) return;

            try {
                const offer = await pc.createOffer();
                await pc.setLocalDescription(offer);

                socket.send(JSON.stringify({
                    type: 'offer',
                    offer: offer,
                    target: targetId,
                    from: userId
                }));

                updateStatus('Calling...');
                document.querySelector('.call-btn').style.display = 'none';
                document.querySelector('.hangup-btn').style.display = 'block';
            } catch (error) {
                console.error('Error starting call:', error);
                updateStatus('Error: Could not establish call');
                endCall();
            }
        }

        // Handle incoming call
        function showIncomingCall(callerId) {
            document.getElementById('callerId').textContent = callerId;
            document.getElementById('incomingCall').style.display = 'block';
        }

        // Answer call
        async function answerCall() {
            if (!await initializeWebRTC()) return;

            try {
                const answer = await pc.createAnswer();
                await pc.setLocalDescription(answer);

                socket.send(JSON.stringify({
                    type: 'answer',
                    answer: answer,
                    target: document.getElementById('callerId').textContent
                }));

                document.getElementById('incomingCall').style.display = 'none';
                document.querySelector('.call-btn').style.display = 'none';
                document.querySelector('.hangup-btn').style.display = 'block';
                updateStatus('Connected');
            } catch (error) {
                console.error('Error answering call:', error);
                updateStatus('Error: Could not answer call');
                endCall();
            }
        }

        // Handle offer
        async function handleOffer(message) {
            await pc.setRemoteDescription(new RTCSessionDescription(message.offer));
        }

        // Handle answer
        async function handleAnswer(message) {
            await pc.setRemoteDescription(new RTCSessionDescription(message.answer));
            updateStatus('Connected');
        }

        // Handle ICE candidate
        async function handleCandidate(message) {
            await pc.addIceCandidate(new RTCIceCandidate(message.candidate));
        }

        // End call
        function endCall() {
            if (socket) {
                socket.send(JSON.stringify({
                    type: 'hangup',
                    target: document.getElementById('targetId').value
                }));
            }

            if (pc) {
                pc.close();
                pc = null;
            }
            if (localStream) {
                localStream.getTracks().forEach(track => track.stop());
                localStream = null;
            }

            document.querySelector('.call-btn').style.display = 'block';
            document.querySelector('.hangup-btn').style.display = 'none';
            document.getElementById('incomingCall').style.display = 'none';
            updateStatus('Call ended');
        }

        // Reject call
        function rejectCall() {
            document.getElementById('incomingCall').style.display = 'none';
            endCall();
        }

        // Add number to display
        function addNumber(num) {
            const targetId = document.getElementById('targetId');
            targetId.value += num;
        }

        // Update status
        function updateStatus(message) {
            document.getElementById('callStatus').textContent = message;
        }

        // Handle backspace
        document.addEventListener('keydown', (e) => {
            if (e.key === 'Backspace') {
                const targetId = document.getElementById('targetId');
                targetId.value = targetId.value.slice(0, -1);
            }
        });
    </script>
</body>
</html>
