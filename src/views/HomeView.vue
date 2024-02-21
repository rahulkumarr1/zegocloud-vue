<template>
  <div class="container">
    <div class="row">
      <div class="col-sm-6">
        <div>
          <video ref="remoteVideo" autoplay playsinline :muted="true" style="width: 300px;"></video>
        </div>
        <div>
          <!-- <a href="#" @click="loginRoom()" class="btn btn-success">Login Room</a> -->
          <a href="#" @click="logoutRoom()" class="btn btn-danger">Logout Room</a> 
          <!-- <a href="#" @click="startPlaying()" class="btn btn-info">Start</a> -->
          <a href="#" @click="stopPlaying()" class="btn btn-warning">Stop</a>
        </div>

      </div>
      <div class="col-sm-6">
        <h3>Message</h3>
        <ul class="list-group">
          <li class="list-group-item" v-for="msg in mesaageList">{{ msg.username }} : {{ msg.message }}</li>
        </ul>
        <div class="input-group mb-3">
          <input type="text" class="form-control" v-model="userMsg" placeholder="Your Message">
          <div class="input-group-append" @click="sendUserMessage()">
            <span class="input-group-text btn btn-info">Send</span>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ZegoExpressEngine } from 'zego-express-engine-webrtc'
import axios from 'axios'

export default {
  components: {},
  data() {
    return {
      appID: 142874099,
      server: "wss://webliveroom142874099-api.coolzcloud.com/ws",
      user_id: 'USER_002',
      userName: 'USER_002',
      token: "",
      room_id: "0007",
      streamID: "0007",
      isChecked: false,
      isLogin: false,
      localStream: null,
      remoteStream: null,
      published: false,
      played: false,
      mesaageList: [],
      userMsg: null,
      videoCodec: localStorage.getItem('VideoCodec') === 'H.264' ? 'H264' : 'VP8',
      zg: null
    }
  },
  methods: {
    createZegoExpressEngine() {
      this.zg = new ZegoExpressEngine(this.appID, this.server);
      this.genrateToken();
    },
    genrateToken() {
      axios.get("https://dhwaniastro.democlicks.com/api/zego_access_token", {
        params: {
          roomId: this.room_id,
          userID: this.user_id,
        }
      }).then((result) => {
        console.log(result);
        if (result.data.status == true) {
          this.token = result.data._token;
        } else {
          console.log("token not genrate");
        }
      });
    },
    initEvent() {
      // Callback for updates on the current user's room connection status.
      this.zg.on('roomStateUpdate', (roomID, state, errorCode, extendedData) => {
        if (state == 'DISCONNECTED') {
          // Disconnected from the room.
        }

        if (state == 'CONNECTING') {
          // Connecting to the room.
        }

        if (state == 'CONNECTED') {
          // Connected to the room.
        }
      });

      // Callback for updates on the status of ther users in the room.
      this.zg.on('roomUserUpdate', (roomID, updateType, userList) => {
        console.warn(`roomUserUpdate: room ${roomID}, user ${updateType === 'ADD' ? 'added' : 'left'} `, JSON.stringify(userList));
      });

      // Callback for updates on the status of the streams in the room.
      this.zg.on('roomStreamUpdate', async (roomID, updateType, streamList, extendedData) => {
        if (updateType == 'ADD') {
          // New stream added, start playing the stream.
          this.streamID = streamList[0].streamID;
          this.startPlaying();
        } else if (updateType == 'DELETE') {
          // Stream deleted, stop playing the stream.
          this.stopPlaying();
        }

        console.log('roomStreamUpdate roomID ', roomID, streamList);
      });

      this.zg.on('IMRecvBroadcastMessage', (roomID, chatData) => {
        for (let i = 0; i < chatData.length; i++) {
          console.log({ username: chatData[i].fromUser.userName, message: chatData[i].message });
          this.mesaageList.push({ username: chatData[i].fromUser.userName, message: chatData[i].message });
        }
      });
    },
    loginRoom() {
      if (!this.isLogin) {
        try {
          this.isLogin = true;
          return this.zg.loginRoom(this.room_id, this.token, {
            userID: this.user_id,
            userName: this.userName
          });
        } catch (err) {
          throw err;
        }
      }
    },
    startPlaying() {
      const flag = this.startPlayingStream(this.streamID);
    },
    stopPlaying() {
      const flag = this.stopPlayingStream(this.streamID);
    },
    logoutRoom() {
      this.zg.logoutRoom(this.room_id);
    },
    sendUserMessage() {
      const message = this.userMsg;
      this.sendBroadcastMessage(this.room_id, message);
      this.mesaageList.push({ username: this.userName, message: message });
      this.userMsg = null;
    },
    async startPlayingStream(stream_Id, options = {}) {
      const remoteStream = await this.zg.startPlayingStream(stream_Id);
      // The remoteVideo object is the <video> or <audio> element on your webpage.
      this.$refs['remoteVideo'].srcObject = remoteStream;

    },
    async stopPlayingStream(stream_Id) {
      this.zg.stopPlayingStream(stream_Id)
    },

    async sendBroadcastMessage(roomId, message) {
      try {
        const msgData = await this.zg.sendBroadcastMessage(roomId, message)
        return msgData
      } catch (err) {
        return { errorCode: 1, extendedData: JSON.stringify(err) }
      }
    }
  },
  watch: {
    token() {
      this.loginRoom();
    },
    isLogin() {
      if (this.isLogin == true) {
        this.startPlaying();
      }
    }
  },
  mounted() {
    this.createZegoExpressEngine();
    this.initEvent();

  },
};
</script>

<style>
#app {
  height: 100vh;
  width: 100vw;
}
</style>
