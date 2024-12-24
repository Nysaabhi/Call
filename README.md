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

        #phoneNumber {
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

        .key span {
            font-size: 12px;
            color: #666;
            display: block;
            text-align: center;
            line-height: 1;
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

        .call-btn:hover {
            background: #43A047;
        }

        .hangup-btn:hover {
            background: #E53935;
        }

        .status {
            text-align: center;
            margin-top: 10px;
            color: #666;
            font-size: 14px;
        }

        body > h1:first-of-type:not(.heading) {
    writing-mode: vertical-rl; /* Rotates text vertically */
    text-orientation: upright; /* Keeps characters upright */
    display: block !important; /* Ensure it stays visible for styling */
}

.markdown-body h1:first-child {
    writing-mode: vertical-rl; /* Rotates text vertically */
    text-orientation: upright; /* Keeps characters upright */
    display: block !important; /* Ensure visibility */
}

.position-relative h1:first-child {
    writing-mode: vertical-rl; /* Rotates text vertically */
    text-orientation: upright; /* Keeps characters upright */
    display: block !important; /* Ensure visibility */
}

    </style>
</head>
<body>
    <div class="dialer">
        <div class="screen">
            <input type="tel" id="phoneNumber" placeholder="Enter number" readonly>
        </div>
        <div class="keypad">
            <div class="key" onclick="addNumber('1')">1</div>
            <div class="key" onclick="addNumber('2')">2<span>ABC</span></div>
            <div class="key" onclick="addNumber('3')">3<span>DEF</span></div>
            <div class="key" onclick="addNumber('4')">4<span>GHI</span></div>
            <div class="key" onclick="addNumber('5')">5<span>JKL</span></div>
            <div class="key" onclick="addNumber('6')">6<span>MNO</span></div>
            <div class="key" onclick="addNumber('7')">7<span>PQRS</span></div>
            <div class="key" onclick="addNumber('8')">8<span>TUV</span></div>
            <div class="key" onclick="addNumber('9')">9<span>WXYZ</span></div>
            <div class="key" onclick="addNumber('*')">*</div>
            <div class="key" onclick="addNumber('0')">0<span>+</span></div>
            <div class="key" onclick="addNumber('#')">#</div>
        </div>
        <div class="actions">
            <button class="call-btn" onclick="startCall()">Call</button>
            <button class="hangup-btn" onclick="endCall()">End</button>
        </div>
        <div class="status" id="callStatus"></div>
    </div>

    <script>
        let pc = null; // RTCPeerConnection
        let localStream = null;
        
        // Add number to display
        function addNumber(num) {
            const phoneNumber = document.getElementById('phoneNumber');
            phoneNumber.value += num;
        }

        // Initialize WebRTC connection
        async function initializeWebRTC() {
            try {
                // Get user media (microphone)
                localStream = await navigator.mediaDevices.getUserMedia({ audio: true });
                
                // Create RTCPeerConnection
                const configuration = {
                    iceServers: [
                        { urls: 'stun:stun.l.google.com:19302' }
                    ]
                };
                pc = new RTCPeerConnection(configuration);
                
                // Add local stream to connection
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
            const phoneNumber = document.getElementById('phoneNumber').value;
            if (!phoneNumber) {
                updateStatus('Please enter a phone number');
                return;
            }

            if (!await initializeWebRTC()) return;

            try {
                // Create and set local description
                const offer = await pc.createOffer();
                await pc.setLocalDescription(offer);

                // Show calling status
                updateStatus('Calling...');
                document.querySelector('.call-btn').style.display = 'none';
                document.querySelector('.hangup-btn').style.display = 'block';

                // In a real implementation, you would:
                // 1. Send the offer to a signaling server
                // 2. Receive answer from remote peer
                // 3. Set remote description
                // 4. Exchange ICE candidates

            } catch (error) {
                console.error('Error starting call:', error);
                updateStatus('Error: Could not establish call');
                endCall();
            }
        }

        // End call
        function endCall() {
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
            updateStatus('Call ended');
        }

        // Update status message
        function updateStatus(message) {
            document.getElementById('callStatus').textContent = message;
        }

        // Handle backspace with keyboard
        document.addEventListener('keydown', (e) => {
            if (e.key === 'Backspace') {
                const phoneNumber = document.getElementById('phoneNumber');
                phoneNumber.value = phoneNumber.value.slice(0, -1);
            }
        });
    </script>
</body>
</html>
