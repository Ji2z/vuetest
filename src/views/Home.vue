<template>
  <div class="home">
    <img alt="Vue logo" src="../assets/logo.png" />
    <div id="container">
		<div id="wrapper">
			<div id="join" class="animate join">
				<h1>Join a Room</h1>
				<form onsubmit="register(); return false;" accept-charset="UTF-8">
					<p>
						<input type="text" name="name" value="" id="name"
							placeholder="Username" required>
					</p>
					<p>
						<input type="text" name="room" value="" id="roomName"
							placeholder="Room" required>
					</p>
					<p class="submit">
						<input type="submit" name="commit" value="Join!">
					</p>
				</form>
			</div>
			<div id="room" style="display: none;">
				<h2 id="room-header"></h2>
				<div id="participants"></div>
				<input type="button" id="button-leave" onmouseup="leaveRoom();"
					value="Leave room">
			</div>
		</div>
	</div>
    <HelloWorld msg="Welcome to Your Vue.js App" />
  </div>
</template>

<script>
// @ is an alias to /src
import HelloWorld from "@/components/HelloWorld.vue";

export default {
  name: "Home",
  components: {
    HelloWorld,
  },
  data() {
    return {
      PARTICIPANT_MAIN_CLASS : 'participant main',
      PARTICIPANT_CLASS : 'participant',
      ws : null,
      participants : {},
      name: '',
    }
  },
  created() {
       this.connect()
  },
  methods: {
    connect() {
      console.log(location.host)
      const serverURL = 'wss://localhost:8080/groupcall';
      this.ws = new WebSocket(serverURL);
      console.log("kurento webSocket : ", this.ws)     
    },

    Participant(name) {
      this.name = name;
      var container = this.document.createElement('div');
      container.className = isPresentMainParticipant() ? this.PARTICIPANT_CLASS : this.PARTICIPANT_MAIN_CLASS;
      container.id = name;
      var span = this.document.createElement('span');
      var video = this.document.createElement('video');
      var rtcPeer;

      container.appendChild(video);
      container.appendChild(span);
      container.onclick = switchContainerClass;
      this.document.getElementById('participants').appendChild(container);

      span.appendChild(document.createTextNode(name));

      video.id = 'video-' + name;
      video.autoplay = true;
      video.controls = false;


      this.getElement = function() {
        return container;
      }

      this.getVideoElement = function() {
        return video;
      }

      function switchContainerClass() {
        if (container.className === this.PARTICIPANT_CLASS) {
          var elements = Array.prototype.slice.call(document.getElementsByClassName(this.PARTICIPANT_MAIN_CLASS));
          elements.forEach(function(item) {
              item.className = this.PARTICIPANT_CLASS;
            });

            container.className = this.PARTICIPANT_MAIN_CLASS;
          } else {
          container.className = this.PARTICIPANT_CLASS;
        }
      }

      function isPresentMainParticipant() {
        return ((document.getElementsByClassName(this.PARTICIPANT_MAIN_CLASS)).length != 0);
      }

      this.offerToReceiveVideo = function(error, offerSdp, wp){
        if (error) return console.error ("sdp offer error")
        console.log('Invoking SDP offer callback function', wp);
        var msg =  { id : "receiveVideoFrom",
            sender : name,
            sdpOffer : offerSdp
          };
        this.sendMessage(msg);
      }


      this.onIceCandidate = function (candidate, wp) {
          console.log("Local candidate" + JSON.stringify(candidate),wp);

          var message = {
            id: 'onIceCandidate',
            candidate: candidate,
            name: name
          };
          this.sendMessage(message);
      }

      Object.defineProperty(this, 'rtcPeer', { writable: true});

      this.dispose = function() {
        console.log('Disposing participant ' + this.name);
        rtcPeer.dispose();
        container.parentNode.removeChild(container);
      };
    },

    register() {
      this.name = document.getElementById('name').value;
      var room = document.getElementById('roomName').value;

      document.getElementById('room-header').innerText = 'ROOM ' + room;
      document.getElementById('join').style.display = 'none';
      document.getElementById('room').style.display = 'block';

      var message = {
        id : 'joinRoom',
        name : this.name,
        room : room,
      }
      this.sendMessage(message);
      console.log(message)
    },

    onNewParticipant(request) {
      this.receiveVideo(request.name);
    },

    receiveVideoResponse(result) {
      this.participants[result.name].rtcPeer.processAnswer (result.sdpAnswer, function (error) {
        if (error) return console.error (error);
      });
    },

    callResponse(message) {
      if (message.response != 'accepted') {
        console.info('Call not accepted by peer. Closing call');
        stop();
      } else {
        this.webRtcPeer.processAnswer(message.sdpAnswer, function (error) {
          if (error) return console.error (error);
        });
      }
    },

    onExistingParticipants(msg) {
        var constraints = {
          audio : true,
          video : {
            mandatory : {
              maxWidth : 320,
              maxFrameRate : 15,
              minFrameRate : 15
            }
          }
        };
        console.log(this.name + " registered in room " + this.room);
        var participant = new this.Participant(this.name);
        this.participants[name] = participant;
        var video = participant.getVideoElement();

        var options = {
              localVideo: video,
              mediaConstraints: constraints,
              onicecandidate: participant.onIceCandidate.bind(participant)
            }
        participant.rtcPeer = new this.kurentoUtils.WebRtcPeer.WebRtcPeerSendonly(options,
          function (error) {
            if(error) {
              return console.error(error);
            }
            this.generateOffer (participant.offerToReceiveVideo.bind(participant));
        });

        msg.data.forEach(this.receiveVideo);
    },

    leaveRoom() {
      this.sendMessage({
        id : 'leaveRoom'
      });

      for ( var key in this.participants) {
        this.participants[key].dispose();
      }

      document.getElementById('join').style.display = 'block';
      document.getElementById('room').style.display = 'none';

      this.ws.close();
    },

    receiveVideo(sender) {
      var participant = new this.Participant(sender);
      this.participants[sender] = participant;
      var video = participant.getVideoElement();

      var options = {
          remoteVideo: video,
          onicecandidate: participant.onIceCandidate.bind(participant)
        }

      participant.rtcPeer = new this.kurentoUtils.WebRtcPeer.WebRtcPeerRecvonly(options,
          function (error) {
            if(error) {
              return console.error(error);
            }
            this.generateOffer (participant.offerToReceiveVideo.bind(participant));
      });;
    },

    onParticipantLeft(request) {
      console.log('Participant ' + request.name + ' left');
      var participant = this.participants[request.name];
      participant.dispose();
      delete this.participants[request.name];
    },

    sendMessage(message) {
      var jsonMessage = JSON.stringify(message);
      console.log('Sending message: ' + jsonMessage);
      this.ws.send(jsonMessage);
    },
  },
};
</script>
