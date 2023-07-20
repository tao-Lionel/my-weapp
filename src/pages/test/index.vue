<template>
  <view class="record-main">
    <!-- 语音识别结果 -->
    <view class="record-result">
      <text style="word-break: break-all">{{ recorderResult }}</text>
    </view>

    <!-- 底部操作栏 -->
    <view class="record-footer">
      <view style="width: 56px; text-align: center">
        <van-icon name="cross" />
      </view>
      <van-button round block color="linear-gradient(to right, #FC7527, #FF2C00)" style="width: 50%" @touchend="onTouchend" @longpress="onStStart" @touchmove="onTouchMove">
        <view>
          <text style="font-size: 36rpx">{{ !recording ? "按住后请说话" : "松开完成" }}</text>
        </view>
      </van-button>

      <!-- <view>
        <van-field :value="value" placeholder="请输入" :border="false" @change="onChange" />
      </view> -->

      <view style="width: 56px; text-align: center"><van-icon name="edit" /></view>
    </view>

    <!-- 遮罩栏 -->
    <view v-if="showCancelMask" class="record-footer-mask">
      <van-icon name="close" color="grey" size="50px" />
      <text>松开 取消发送</text>
    </view>
  </view>
</template>

<script>
// import system from "@/api/system.js";
const getToken = require("./utils/token").getToken;
const SpeechRecognition = require("./utils/sr");
const sleep = require("./utils/util").sleep;

const configUrl = `wss://nls-gateway.cn-shanghai.aliyuncs.com/ws/v1`;

const DEFAULT = "请说话，我在听...";

export default {
  components: {},
  data() {
    return {
      srStart: false,
      stResult: {}, // 语音识别的结果
      recorderResult: "", // 语音识别结束后回显的句子
      recordConfig: {
        duration: 600000,
        numberOfChannels: 1,
        sampleRate: 16000,
        format: "PCM",
        frameSize: 4,
      },
      sr: null,
      recording: false, // 是否录制中
      showCancelMask: false, // 是否显示取消按钮
      timeout: null,
      screenHeight: wx.getSystemInfoSync().screenHeight,
    };
  },

  async onLoad() {
    this.init();
  },

  /**
   * 生命周期函数--监听页面卸载
   */
  onUnload() {
    console.log("sr onUnload");
    this.srStart = false;
    wx.getRecorderManager().stop();
    if (this.sr) {
      this.sr.shutdown();
    } else {
      console.log("sr is null");
    }
  },

  methods: {
    async init() {
      // 监听已经录制完指定帧大小的回调
      wx.getRecorderManager().onFrameRecorded(res => {
        //当前帧是否正常录音结束前的最后一帧
        if (res.isLastFrame) {
          console.log("录音完成");
        }
        if (this.sr && this.srStart) {
          console.log("监听 录制 send " + res.frameBuffer.byteLength);
          this.sr.sendAudio(res.frameBuffer);
        }
      });

      // 监听录音开始事件
      wx.getRecorderManager().onStart(() => {
        console.log("开始录音...");
      });

      // 监听录音结束事件
      wx.getRecorderManager().onStop(res => {
        console.log("结束录音...");
        // 录音文件的临时路径 (本地路径)
        console.log("录音文件的临时路径", res.tempFilePath);
        if (res.tempFilePath) {
          wx.getFileSystemManager().removeSavedFile({
            filePath: res.tempFilePath,
          });
        }
        this.recording = false;
      });

      //监听录音错误事件
      wx.getRecorderManager().onError(res => {
        console.log("录音出错:" + JSON.stringify(res));
      });

      // 获取语音识别token等参数
      // const { data } = await system.getRecordArguments();

      // try {
      //   this.data.token = await getToken();
      // } catch (e) {
      //   console.log("error on get token:", JSON.stringify(e));
      //   return;
      // }

      let sr = new SpeechRecognition({
        url: configUrl,
        appkey: "QO3JEdOaoCqX1jXu",
        token: "7397a84c4f7b4821acdb7815c93ba20f",
      });

      console.log("sr==========", sr);

      sr.on("started", msg => {
        console.log("语音识别开始", msg);
        this.stResult = msg;
      });

      sr.on("changed", msg => {
        console.log("语音识别中间结果:", msg);
        this.stResult = msg;
        // 回显到页面上的句子
        this.recorderResult = JSON.parse(msg).payload.result;
      });

      sr.on("completed", msg => {
        console.log("语音识别完成:", msg);
        this.stResult = msg;
        // 回显到页面上的句子
        this.recorderResult = JSON.parse(msg).payload.result;
      });

      sr.on("closed", () => {
        console.log("连接关闭");
      });

      sr.on("failed", msg => {
        console.log("错误:", msg);
        this.stResult = msg;
      });

      this.sr = sr;
    },

    // 按住按钮开始语音识别
    async onStStart(e) {
      console.log("按住按钮开始语音识别");
      // 录制中
      this.recording = true;
      if (!this.sr) {
        console.log("sr is null");
        return;
      }

      if (this.srStart) {
        console.log("sr is started!");
        return;
      }
      let sr = this.sr;
      try {
        await sr.start({ ...sr.defaultStartParams(), enable_intermediate_result: true });
        this.srStart = true;
      } catch (e) {
        console.log("启动失败" + e);
        return;
      }

      wx.getRecorderManager().start(this.recordConfig);
      this.recorderResult = DEFAULT;
    },

    // 松开按钮
    async onTouchend(e) {
      console.log("松开按钮结束语音识别", e);
      setTimeout(() => {
        this.showCancelMask = false;
      }, 150);

      if (!this.recording) {
        uni.showToast({
          title: "说话时间太短",
          icon: "none",
        });
        return;
      }

      this.cancelRecorder();
    },

    // 移动取消
    async onTouchMove(e) {
      if (this.timeout) clearTimeout(this.timeout);
      this.timeout = setTimeout(() => {
        if (e.touches[0].clientY < this.screenHeight - 200) {
          // 取消
          console.log("移动取消", e);
          this.showCancelMask = true;
        } else {
          // 不取消
          console.log("不取消");
          this.showCancelMask = false;
        }
      }, 100);
    },

    // 取消录音和语音识别
    async cancelRecorder() {
      wx.getRecorderManager().stop();
      await sleep(500);
      if (this.srStart && this.sr) {
        try {
          console.log("prepare close sr");
          await this.sr.close();
          this.srStart = false;
        } catch (e) {
          console.log("close sr failed:" + JSON.stringify(e));
        }
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.record-main {
  position: relative;
  width: 100vw;
  height: 100vh;
  background-image: linear-gradient(to bottom, rgb(35, 70, 159), rgb(4, 4, 53));
  color: white;
}
.record-result {
  width: 100%;
  display: flex;
  justify-content: center;
  font-size: 40rpx;
  font-weight: bold;
  position: fixed;
  bottom: 300px;
  text {
    word-break: break-all;
  }
}

.record-footer {
  width: 100%;
  position: fixed;
  bottom: 20px;
  display: flex;
  justify-content: space-evenly;
  align-items: center;
  font-size: 30rpx;
}

.record-footer-mask {
  position: fixed;
  bottom: 160px;
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text {
    margin-top: 5px;
  }
}
</style>
