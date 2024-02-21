<template>
  <div class="container">
    <div class="row">
      <div class="col-sm-6">
        <div>
          <div id="local-video"></div>
        </div>
        <div>
          <a href="#" @click="loginRoom()" class="btn btn-success">Login Room</a>
          <a href="#" @click="logoutRoom()" class="btn btn-danger">Logout Room</a>
          <a href="#" @click="startPlaying()" class="btn btn-info">Start</a>
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
      user_id: 'USER_001',
      userName: 'USER_001',
      token: "",
      room_id: "0007",
      streamID: "0007",
      isChecked: false,
      isLogin: false,
      publishVideo: false,
      localStream: null,
      remoteStream: null,
      published: false,
      played: false,
      mesaageList: [],
      userMsg: null,
      microphoneDevicesVal: null,
      cameraDevicesVal: '',
      cameraCheckStatus: true,
      microphoneCheckStatus: true,
      publishStreamStatus: false,
      connectStatus: 'DISCONNECTED',
      audioDeviceList: [],
      videoDeviceList: [],
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
    async enumDevices() {
      const deviceInfo = await this.zg.enumDevices();
      this.audioDeviceList = deviceInfo &&
        deviceInfo.microphones.map((item, index) => {
          if (!item.deviceName) {
            item.deviceName = 'microphone' + index;
          }
          console.log('microphone: ' + item.deviceName);
          return item;
        });
      this.audioDeviceList.push({ deviceID: 0, deviceName: '禁止' });
      this.microphoneDevicesVal = this.audioDeviceList[0].deviceID;
      this.videoDeviceList = deviceInfo &&
        deviceInfo.cameras.map((item, index) => {
          if (!item.deviceName) {
            item.deviceName = 'camera' + index;
          }
          console.log('camera: ' + item.deviceName);
          return item;
        });
      this.videoDeviceList.push({ deviceID: 0, deviceName: '禁止' });
      this.cameraDevicesVal = this.videoDeviceList[0].deviceID;
    },
    async checkSystemRequirements() {

      console.log('sdk version is', this.zg.getVersion());
      try {
        const result = await this.zg.checkSystemRequirements();
        console.warn('checkSystemRequirements ', result);
        if (!result.webRTC) {
          console.error('browser is not support webrtc!!');
          return false;
        } else if (!result.videoCodec.H264 && !result.videoCodec.VP8) {
          console.error('browser is not support H264 and VP8');
          return false;
        } else if (!result.camera && !result.microphone) {
          console.error('camera and microphones not allowed to use');
          return false;
        } else if (result.videoCodec.VP8) {
          if (!result.screenSharing) console.warn('browser is not support screenSharing');
        } else {
          console.log('不支持VP8，请前往混流转码测试');
        }
        return true;
      } catch (err) {
        console.error('checkSystemRequirements', err);
        return false;
      }
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
      const config = {
        camera: {
          video: this.cameraCheckStatus ? {
            input: this.cameraDevicesVal
          } : false,
          audio: this.microphoneCheckStatus ? {
            input: this.microphoneDevicesVal
          } : false,
        }
      };
      const flag = this.startPlayingStream(this.streamID, config);
      this.publishVideo = true;

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
      //  Start Publishing Stream
      try {
        this.localStream = await this.zg.createZegoStream(options);
        console.log(options);
        // Play preview of the Stream.
        this.zg.startPublishingStream(stream_Id, this.localStream);
        this.localStream.playVideo(document.querySelector("#local-video"));
        this.isload = true;
        return true;
      } catch (err) {
        return false;
      }
    },
    async stopPlayingStream(stream_Id) {
      this.zg.stopPublishingStream(stream_Id);
      this.zg.destroyStream(this.localStream);
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
    this.checkSystemRequirements()
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
