<!-- restful  -->
<template>
  <view style="width: 99%">
    <view style="margin-top: 50px">
      <van-button round block plain color="#F0780E" style="width: 48%" @tap="onStStart">
        <view class="center">
          <text>{{ msg }}</text>
        </view>
      </van-button>
    </view>

    <view>
      <!-- <text style="word-break: break-all">{{ stResult }}</text> -->
      <text style="word-break: break-all">{{ recorderResult }}</text>
    </view>

    <view style="margin-top: 50px">
      <van-button round block plain color="#F0780E" style="width: 48%" @tap="onStStop">
        <view>
          <text>停止</text>
        </view>
      </van-button>
    </view>
  </view>
</template>

<script>
import system from "@/api/system.js";
const getToken = require("./utils/token").getToken;
const SpeechTranscription = require("./utils/st");
const sleep = require("./utils/util").sleep;

const configUrl = `wss://nls-gateway.cn-shanghai.aliyuncs.com/ws/v1`;

export default {
  components: {},
  data() {
    return {
      stStart: false,
      stResult: {}, // 语音识别的结果
      recorderResult: "", // 语音识别结束后回显的句子
      recordConfig: {
        duration: 600000,
        numberOfChannels: 1,
        sampleRate: 16000,
        format: "PCM",
        frameSize: 4
      },
      msg: "点击开始",
      st: null
    };
  },

  async onLoad() {
    // 监听已经录制完指定帧大小的回调
    wx.getRecorderManager().onFrameRecorded((res) => {
      //当前帧是否正常录音结束前的最后一帧
      if (res.isLastFrame) {
        console.log("录音完成");
      }
      if (this.st && this.stStart) {
        console.log("监听 录制 send " + res.frameBuffer.byteLength);
        this.st.sendAudio(res.frameBuffer);
      }
    });

    // 监听录音开始事件
    wx.getRecorderManager().onStart(() => {
      console.log("开始录音...");
    });

    // 监听录音结束事件
    wx.getRecorderManager().onStop((res) => {
      console.log("结束录音...");
      // 录音文件的临时路径 (本地路径)
      if (res.tempFilePath) {
        wx.getFileSystemManager().removeSavedFile({
          filePath: res.tempFilePath
        });
      }
    });

    //监听录音错误事件
    wx.getRecorderManager().onError((res) => {
      console.log("录音出错:" + res);
    });

    // 获取语音识别token等参数
    const { data } = await system.getRecordArguments();

    // try {
    //   this.data.token = await getToken();
    // } catch (e) {
    //   console.log("error on get token:", JSON.stringify(e));
    //   return;
    // }

    let st = new SpeechTranscription({
      url: configUrl,
      appkey: "QO3JEdOaoCqX1jXu",
      token: "539a05784ef6400eb9d01a66d35e5f9c"
    });

    console.log("st==========", st);

    st.on("started", (msg) => {
      console.log("实时语音识别开始", msg);
      this.stResult = msg;
    });

    st.on("changed", (msg) => {
      console.log("实时语音识别中间结果:", msg);
      this.stResult = msg;
      // 回显到页面上的句子
      this.recorderResult = JSON.parse(msg).payload.result;
    });

    st.on("completed", (msg) => {
      console.log("实时语音识别完成:", msg);
      this.stResult = msg;
    });

    st.on("begin", (msg) => {
      console.log("句子开始:", msg);
      this.stResult = msg;
    });

    st.on("end", (msg) => {
      console.log("句子结束:", msg);
      this.stResult = msg;
      // 回显到页面上的句子
      this.recorderResult = JSON.parse(msg).payload.result;
    });

    st.on("closed", () => {
      console.log("连接关闭");
    });

    st.on("failed", (msg) => {
      console.log("错误:", msg);
      this.stResult = msg;
    });

    this.st = st;
  },

  /**
   * 生命周期函数--监听页面卸载
   */
  onUnload() {
    console.log("st onUnload");
    this.stStart = false;
    wx.getRecorderManager().stop();
    if (this.st) {
      this.st.shutdown();
    } else {
      console.log("st is null");
    }
  },

  methods: {
    // 按住按钮开始语音识别
    async onStStart(e) {
      console.log("按住按钮开始语音识别");
      if (!this.st) {
        console.log("st 是 null");
        return;
      }

      if (this.stStart) {
        console.log("st is started!");
        return;
      }
      let st = this.st;
      try {
        await st.start({ ...st.defaultStartParams(), enable_intermediate_result: true });
        this.stStart = true;
      } catch (e) {
        console.log("启动失败" + e);
        return;
      }

      wx.getRecorderManager().start(this.recordConfig);

      this.msg = "录音中";
    },

    async onStStop() {
      wx.getRecorderManager().stop();
      await sleep(500);
      if (this.stStart && this.st) {
        try {
          console.log("prepare close st");
          await this.st.close();
          this.stStart = false;
        } catch (e) {
          console.log("close st failed:" + JSON.stringify(e));
        }
      }
      this.msg = "点击开始";
    }
  }
};
</script>

<style lang="scss" scoped>
image {
  width: 108rpx;
  height: 108rpx;
  border-radius: 50%;
}
</style>
