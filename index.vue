<template>
  <view class="sw-player-wrap">
    <view
      :class="{
        'agora-player': true,
        'vertical-player-wrap': liveInfo.isVertical == VERTICAL,
        'horizontal-player-wrap': liveInfo.isVertical == HORIZONTAL,
      }"
    >
      <view class="sheng-wang" id="video-agora"></view>
      <!-- 加载提示Icon -->
      <view class="sw-loading" v-if="isShowLoadingIcon"></view>

      <!-- 在线人数 -->
      <view
        v-if="visitorCount > 0 && liveInfo.iscloseOnlineNumber == 1"
        class="online-number"
      >
        <text>{{ visitorCount }}在看</text>
      </view>

      <!-- 会员ID -->
      <view class="member-id" v-if="!isFullScreen && memberIdSwitch">
        <text class="id">{{ member.memberId }}</text>
      </view>

      <!-- 遮罩层、错误提示 -->
      <view
        class="mask-toast-info"
        v-watermark="{ text: watermarkText, isFullScreen: isFullScreen }"
      >
        <view v-if="iShowToastInfo" class="toast-info">
          <view class="error-text">
            <text>{{ errorPlayText }}</text>
          </view>
        </view>
      </view>
      <!-- 播放按钮 -->
      <view v-if="showBtn" class="play-btn" @click="startPlay">
        <image class="play-btn-img" src="@/static/icon/play1.png" />
      </view>
    </view>
  </view>
</template>
<script>
import { HORIZONTAL, VERTICAL } from "@/config/constants";
import { getRandomRange } from "@/config/utils";
import aegisLiveView from "@/mixins/aegisLiveView";
import {
  LivePlayer,
  PlayerEvent,
  setRTCParameter,
  enableLogUpload,
  setLogLevel,
  RtcEvent,
} from "agora-fls-sdk";
export default {
  mixins: [aegisLiveView],
  props: {
    liveInfo: {
      type: Object,
      default: {},
    },
    merchantId: {
      type: [String, Number],
      default: null,
    },
    liveId: {
      type: [String, Number],
      default: null,
    },
    visitorCount: {
      type: [String, Number],
      default: 0,
    },
    member: {
      type: Object,
      default: {},
    },
    adaptionPlan: {
      type: [String, Number],
      default: 1,
    },
    coverShow: {
      type: Boolean,
      default: true,
    },
    watermarkText: {
      type: String,
      default: "",
    },
    isFullScreen: {
      type: Boolean,
      default: false,
    },
    memberIdSwitch: {
      type: [String, Number],
      default: 0,
    },
  },
  data() {
    return {
      HORIZONTAL, // 横屏--常量
      VERTICAL, // 竖屏--常量
      isShowLoadingIcon: true, // 是否显示加载Icon
      agoraVideo: null, // 声网播放器
      iShowToastInfo: false, // 是否显示提示文案
      errorPlayText: "主播离开一会，请稍后~", // 错误文案信息
      timer: null,
      showBtn: false, // 是否显示播放按钮
    };
  },
  methods: {
    // 创建声网播放器
    createAgoraPlayer() {
      // console.log(isRtcSupported(), isHlsSupported(), "是否支持")
      // 开启日志上报
      enableLogUpload();
      // 设置日志的输出级别
      setLogLevel(2);
      // 解决触发播放事件，但海报封面依然显示1s左右延迟问题
      setRTCParameter("ENABLE_INSTANT_VIDEO", true);
      setRTCParameter("ENABLE_PRE_SUB", true);
      // 1. 用dom创建video标签  2. uniapp会把原生video标签编译为uni-video
      const video = document.createElement("video");
      video.setAttribute("id", `remote-video`);
      video.setAttribute("playsinline", true);
      video.setAttribute("webkit-playsinline", true);
      // video.setAttribute("poster", this.liveInfo.coverUrl);
      video.setAttribute(
        "style",
        `background-image: url(${this.liveInfo.coverUrl})`
      );
      const videoAgora = document.getElementById("video-agora");
      videoAgora.appendChild(video);
      // 播放器参数配置
      const url = this.liveInfo.pullUrl.replace("&uid=", "");
      console.log(url, "agora-url参数调整");
      const playerOptions = {
        url,
        el: "remote-video",
        width: uni.getSystemInfoSync().windowWidth,
        height: videoAgora.offsetHeight,
        aspectRatio: "16/9",
        autoSwitchHLS: true,
        autoplay: false,
        mirror: false,
        objectFit: `${this.adaptionPlan == 2 ? "cover" : "fill"}`,
        hlsConfig: {
          maxBufferLength: 40,
          maxBufferSize: 8 * 1000 * 1000,
          maxMaxBufferLength: 700,
        },
        timeout: 30000,
      };
      this.agoraVideo = new LivePlayer(playerOptions);
      this.aegisInfoAll(
        "声网播放器-初始化成功",
        {
          ["url信息"]: url,
        },
        this.member.memberId
      );
      console.log(this.agoraVideo, "声网播放器");
      // 网络状态
      this.agoraVideo.on(PlayerEvent.NETWORK_QUALITY, (e) => {
        // console.log(e, "声网网络状态");
      });
      // 分配的 uid 发生变化
      this.agoraVideo.on(PlayerEvent.USER_ASSIGNED, (e) => {
        console.log(e, "声网播放器-分配的uid");
        this.aegisInfoAll(
          "声网播放器-分配的uid",
          {
            ["声网uid"]: e,
          },
          this.member.memberId
        );
      });

      // 播放器状态
      this.agoraVideo.on(PlayerEvent.PLAY_STATE_CHANGED, (e) => {
        this.isShowLoadingIcon = e == "playing" ? false : true;
        console.log(e, "声网播放器状态");
        if (e == "playing") {
          this.$emit("envelopeCountDonPlay", "声网视频播放");
          this.showBtn = false; // 隐藏播放按钮
        }
      });
      // 频道内远端用户状态发生改变
      this.agoraVideo.on(PlayerEvent.RTC_USER_STATE_CHANGED, (e) => {
        console.log(e, "频道内远端用户状态发生改变");
        if (e.state == "unpublished") {
          console.log("主播已离开");
          // 判断如果停止播放，则检查直播状态 采用随机4-6s调用
          const random = getRandomRange(4000, 6000); // 随机数
          clearTimeout(this.timer);
          this.timer = setTimeout(() => {
            this.$emit("reloadObsLive", "videoError");
          }, random);
        }
      });

      // 处理rtc一些异常事件状态
      this.agoraVideo.on(PlayerEvent.RTC_EVENTS, (e) => {
        console.log(e, "声网处理rtc一些异常事件状态", RtcEvent);
      });

      // rtc频道内当前播放的媒体源发生变化
      this.agoraVideo.on(PlayerEvent.RTC_MEDIA_CHANGED, (e) => {
        // 媒体源发生变化
        this.$emit("mediaChanged", e.state);
        if (e.state == "playing") {
          this.iShowToastInfo = false;
          this.showBtn = false; // 隐藏播放按钮
        }
        console.log(e, "声网rtc频道内当前播放的媒体源发生变化");
      });

      // rtc视频源状态发生变化
      this.agoraVideo.on(PlayerEvent.RTC_SOURCE_STATE_CHANGED, (e) => {
        console.log(e, "声网rtc视频源状态发生变化");
      });

      // 播放器切换视频源（如从 RTC 切换到 HLS）成功后，触发该事件
      this.agoraVideo.on(PlayerEvent.MEDIA_SOURCE_CHANGED, (e) => {
        console.log(
          e,
          "声网播放器切换视频源（如从 RTC 切换到 HLS）成功后，触发该事件"
        );
        this.aegisInfoAll(
          "声网视频--播放器切换视频源（如从 RTC 切换到 HLS）成功",
          {
            ["错误信息"]: e,
          },
          this.member.memberId
        );
      });

      // 视频源出现不可恢复的错误，一旦触发需要切换视频源。如从 RTC 切换到 HLS
      this.agoraVideo.on(PlayerEvent.REQUEST_SWITCH_MEDIA_SOURCE, (e) => {
        console.log(
          e,
          "视频源出现不可恢复的错误，一旦触发需要切换视频源。如从 RTC 切换到 HLS"
        );
        this.aegisInfoAll(
          "声网视频--视频源出现不可恢复的错误",
          {
            ["错误信息"]: e,
          },
          this.member.memberId
        );
      });

      // 播放器错误
      this.agoraVideo.on(PlayerEvent.ERROR, (e) => {
        // 拉流超时提醒： 视频加载超时: 10000ms，播放器将继续尝试加载
        console.log(e, "声网播放器错误");
        this.iShowToastInfo = true;
        this.errorPlayText = "主播离开一会，请稍后~";
        if (e.error.code === "TIMEOUT") {
          // TODO TIMEOUT 30s传递自定义事件 需要排查断流哪一刻是否会触发对应事件
          this.$emit("reloadObsLive", "videoError");
        }
        this.aegisInfoAll(
          "声网视频--播放器错误",
          {
            ["错误信息"]: e,
          },
          this.member.memberId
        );
      });
      // 自动播放失败 手动触发不会触发
      this.agoraVideo.on(PlayerEvent.AUTOPLAY_PREVENTED, (e) => {
        console.log(e, "视频自动播放失败");
        this.agoraVideo.pause().then(() => {
          console.log("我被暂停了------------");
          this.showBtn = true;
          this.isShowLoadingIcon = false;
        });
      });
    },
    startPlay() {
      if (!this.agoraVideo) return;
      console.log(this.agoraVideo, "触发声网播放事件");
      this.agoraVideo.play();
      this.showBtn = false; // 隐藏播放按钮
    },
    handlePause() {
      this.agoraVideo.pause();
    },
    async handleDestroy() {
      await this.agoraVideo.destroy();
      this.agoraVideo = null;
    },
  },
};
</script>

<style lang="scss" scoped>
@-webkit-keyframes loadingagora {
  0% {
    -webkit-transform: rotate(0);
    transform: rotate(0);
  }
  100% {
    -webkit-transform: rotate(360deg);
    transform: rotate(360deg);
  }
}
@keyframes loadingagora {
  0% {
    -webkit-transform: rotate(0);
    transform: rotate(0);
  }
  100% {
    -webkit-transform: rotate(360deg);
    transform: rotate(360deg);
  }
}

.sw-player-wrap {
  /deep/ .agora-player {
    position: relative;

    &.horizontal-player-wrap {
      .sheng-wang,
      .mask-toast-info {
        height: 440rpx;
      }
      .sheng-wang #remote-video {
        height: 440rpx;
        background-position: center;
        background-repeat: no-repeat;
        background-size: cover;
      }
    }

    &.vertical-player-wrap {
      .sheng-wang,
      .mask-toast-info {
        height: 100vh;
      }
      .sheng-wang #remote-video {
        height: 100vh;
        background-position: center;
        background-repeat: no-repeat;
        background-size: cover;
      }
    }

    .sw-loading {
      box-sizing: border-box;
      background-clip: padding-box;
      width: 100rpx;
      height: 100rpx;
      position: absolute;
      top: 50%;
      left: 50%;
      z-index: 10;
      margin: -50rpx 0 0 -50rpx;
      text-indent: -9999em;
      border-radius: 50%;
      border: 6rpx solid rgba(255, 255, 255, 0);
      border-left-color: #fff;
      border-right-color: #fff;
      -webkit-transform: translateZ(0);
      -ms-transform: translateZ(0);
      transform: translateZ(0);
      -webkit-animation: loadingagora 1.1s infinite linear;
      animation: loadingagora 1.1s infinite linear;
    }

    .online-number {
      position: absolute;
      left: 8rpx;
      top: 8rpx;
      padding: 8rpx 16rpx;
      border-radius: 8rpx;
      color: #fff;
      font-size: 24rpx;
      background: rgba(0, 0, 0, 0.5);
      z-index: 11;
    }

    .member-id {
      position: absolute;
      top: 10rpx;
      right: 10rpx;
      font-size: 24rpx;
      font-weight: 500;
      color: #fff;
      border-radius: 8rpx;
      padding: 8rpx 16rpx;
      background: rgba(0, 0, 0, 0.5);
    }

    .mask-toast-info {
      position: absolute;
      top: 0;
      bottom: 0;
      left: 0;
      right: 0;
      z-index: 10;
      .toast-info {
        padding: 0 20rpx;
        height: 100%;
        background-image: url("@/static/live/live_bg_row.png");
        background-repeat: no-repeat;
        background-size: 100% 100%;
        display: flex;
        align-items: center;
        justify-content: center;
        .error-text {
          font-size: 32rpx;
          color: #fff;
        }
      }
    }
    .play-btn {
      position: absolute;
      left: 0;
      top: 0;
      width: 100%;
      height: 100%;
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 12;

      .play-btn-img {
        width: 80rpx;
        height: 80rpx;
      }
    }
  }
}

// 横屏模式下的全屏
.fullScreen-layout {
  .sw-player-wrap {
    width: 100vw;
    height: 100vh;
    /deep/ .agora-player {
      position: relative;

      .sheng-wang {
        height: 100vh;
        #remote-video {
          position: absolute;
          top: 0;
          left: 100vw;
          width: 100vh !important;
          height: 100vw !important;
          transform: translate(0, 0) rotate(90deg) !important;
          transform-origin: 0 0;
          background-position: center;
          background-repeat: no-repeat;
          background-size: cover;
        }
      }

      .online-number {
        top: 20rpx;
        left: calc(100vw - 36rpx);
        transform: translate(0, 0) rotate(90deg);
        transform-origin: 0 0;
        width: max-content;
      }

      .mask-toast-info {
        width: 100vw;
        height: 100vh !important;
        .toast-info {
          .error-text {
            transform: rotate(90deg);
          }
        }
      }
      .play-btn {
        .play-btn-img {
          transform: rotate(90deg);
        }
      }
    }
  }
}
</style>
