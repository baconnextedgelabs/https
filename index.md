<video id="selfView" width="320" height="240" autoplay muted></video>

<button id="call">Call</button>

<video id="remoteView" width="320" height="240" autoplay muted></video>

<script>
    let address = 'ws://192.168.43.79:9999/';
    let origin = 'client' + +new Date();
    let ws = new WebSocket(address);

    ws.addEventListener('open', function () {
        console.log('Connected');
    });

    ws.addEventListener('close', function () {
        console.log('Connection lost');
    });

    ws.addEventListener('message', function (e) {
        let data = e.data;
        let [name, type, msg] = data.split('|');
        msg = JSON.parse(msg);
        if (!pc)
            start(false);
        if (name !== origin) {
            if (type === 'sdp') {
                console.log('get', 'setRemoteDescription');
                pc.setRemoteDescription(new RTCSessionDescription(msg));
            } else if (type === 'candidate') {
                console.log('get', 'addIceCandidate', name, origin);
                pc.addIceCandidate(new RTCIceCandidate(msg));
            }
        }
    });

    let send = function () {
        let [origin, type, msg] = Array.prototype.slice.call(arguments);
        ws.send(origin + '|' + type + '|' + JSON.stringify(msg));
        console.log('sent', origin, type);
    }

    // ws.send('123123');


    // RTCPeerConnection

    var configuration = {
        iceServers: [{
            urls: "stun:23.21.150.121"
        }, {
            urls: "stun:stun.l.google.com:19302"
        }]
    };
    var pc;

    function start(isCaller) {
        pc = new RTCPeerConnection(configuration);
        // send any ice candidates to the other peer
        pc.onicecandidate = function (evt) {
            send(origin, 'candidate', evt.candidate);
        };
        // once remote stream arrives, show it in the remote video element
        pc.ontrack = function (evt) {
            // remoteView.src = URL.createObjectURL(evt.streams[0]);
            remoteView.srcObject = evt.streams[0];
        };
        // get the local stream, show it in the local video element and send it
        navigator.mediaDevices.getUserMedia({
            "audio": true,
            "video": true
        }).then((stream) => {
            // selfView.src = URL.createObjectURL(stream);
            selfView.srcObject = stream;
            pc.addStream(stream);
            if (isCaller)
                pc.createOffer().then((desc) => gotDescription(desc));
            else
                pc.createAnswer().then((desc) => gotDescription(desc));

            function gotDescription(desc) {
                console.log('setLocalDescription');
                pc.setLocalDescription(desc);
                send(origin, 'sdp', desc);
            }
        });
    }
    call.addEventListener('click', () => {
        start(true);
    });
</script>
