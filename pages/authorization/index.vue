<template>
  <view class="container">
    <!-- #ifdef MP-WEIXIN -->
    <view
      v-if="!token"
      class="force-login-wrap"
    >
      <!-- <image class="logo-bg" :src="`${$VUE_APP_RESOURCES_URL}/images/logo_bg.png`" mode="aspectFill"></image> -->
      <view class="force-login__content y-f">
        <open-data
          class="user-avatar"
          type="userAvatarUrl"
        ></open-data>
        <open-data
          class="user-name"
          type="userNickName"
        ></open-data>
        <view class="login-notice">为了提供更优质的服务，请先登录</view>
        <button
          class="cu-btn author-btn"
          open-type="getPhoneNumber"
          @getphonenumber="phoneLogin"
        >手机号快捷登录</button>
        <!-- <button class="cu-btn author-btn" v-if="canIUseGetUserProfile" @tap="getUserInfoProfile">授权并查看</button>
		<button class="cu-btn author-btn" v-else @getuserinfo="getUserInfoBtn" open-type="getUserInfo">授权并查看</button> -->
        <button
          class="cu-btn close-btn"
          @tap="back"
        >暂不登录</button>
      </view>
    </view>
    <!-- #endif -->
    <!-- #ifndef MP-WEIXIN -->
    <view class="force-login-wrap">
      <!-- <image class="logo-bg" :src="`${$VUE_APP_RESOURCES_URL}/images/logo_bg.png`" mode="aspectFill"></image> -->
      <view class="force-login__content y-f">
        <view class="login-notice">为了提供更优质的服务，请先登录</view>
        <button
          class="cu-btn author-btn"
          @tap="toLogin"
        >去登录</button>
      </view>
    </view>
    <!-- #endif -->
  </view>
</template>

<script>
import { mapState, mapMutations, mapActions } from 'vuex'
import { wxappAuth, getUser, getUserInfo } from '@/api/user'
import { login, authorize, getProvider } from '@/utils'
import dayjs from 'dayjs'
import cookie from '@/utils/store/cookie'
import store from '@/store'

// 公众号登录
import { auth } from '@/libs/wechat.js'

export default {
  data() {
    return {
      authorize: false,
      canIUseGetUserProfile: false,
      code: ''
    }
  },
  computed: {
    ...mapState(['isAuthorization', '$deviceType', 'token']),
  },
  onLoad() {
    if (wx.getUserProfile) {
      this.canIUseGetUserProfile = true
    }
    this.getLoginCode()

    // // 先校验用户是否授权，如果没有授权，显示授权按钮
  },
  mounted() {
    this.getLoginCode()
  },
  onHide() {
    this.updateAuthorizationPage(false)
    this.changeAuthorization(false)
  },
  onUnload() {
    this.updateAuthorizationPage(false)
    this.changeAuthorization(false)
  },
  methods: {
    ...mapActions(['changeAuthorization', 'setUserInfo']),
    ...mapMutations(['updateAuthorizationPage']),
    toLogin() {
      this.$yrouter.push({
        path: '/pages/user/Login/index',
        query: {},
      })
    },
    back() {
      this.$yrouter.switchTab({
        path: '/pages/home/index',
        query: {},
      })
    },
    getUserInfoBtn(data) {
      console.log(data)
      console.log('getUserInfoBtn')
      if (data.detail.errMsg == 'getUserInfo:fail auth deny') {
        uni.showToast({
          title: '取消授权',
          icon: 'none',
          duration: 2000,
        })
        return
      }
      uni.showLoading({
        title: '登录中',
      })
      login()
        .then(e => {
          uni.hideLoading()
          console.log('重定向')
          this.$yrouter.reLaunch({
            path: cookie.get('redirect').replace(/\ /g, ''),
          })
        })
        .catch(error => {
          uni.hideLoading()
          console.log(error)
          uni.showToast({
            title: error,
            icon: 'none',
            duration: 2000,
          })
        })
    },
    // 申请获取用户信息
    getUserInfoProfile(data) {
      console.log('getUserInfoProfile')
      wx.getUserProfile({
        lang: 'zh_CN',
        desc: '需要获取您的信息用来展示',
        success: res => {
          console.log('用户信息', res)
          uni.showLoading({
            title: '登录中',
          })
          login(res)
            .then(e => {
              console.log('重定向')
              this.$yrouter.reLaunch({
                path: cookie.get('redirect').replace(/\ /g, ''),
              })
            })
            .catch(error => {
              console.log(error)
              uni.showToast({
                title: error,
                icon: 'none',
                duration: 2000,
              })
            })
        },
      })
    },
    getLoginCode() {
      getProvider()
        .then(provider => {
          // uni登录
          uni.login({
            provider: provider,
            success: async loginRes => {
              console.log("?? ~ file: index.vue:171 ~ getLoginCode ~ loginRes:", loginRes)
              this.code = loginRes.code
            }
          })
        })
    },
    // 微信用户手机号登录
    phoneLogin(e) {
      console.log('用户微信手机号登录')
      if (e.mp.detail.errMsg == 'getPhoneNumber:ok') {
        console.log('wxLoginCode', this.code)
        cookie.set('wxLoginCode', this.code)
        wxappAuth({
          encryptedData: e.mp.detail.encryptedData,
          iv: e.mp.detail.iv,
          code: this.code,
          spread: cookie.get('spread'),
        })
          .then(res => {
            console.log('登录成功,开始处理登录信息保存，并获取用户详情')
            uni.hideLoading()
            store.commit('login', res.data.token, dayjs(res.data.expires_time))
            store.dispatch('userInfo', true)
            getUserInfo()
              .then(user => {
                console.log('获取用户信息成功')
                uni.setStorageSync('uid', user.data.uid)
                store.dispatch('setUserInfo', user.data)
                this.$yrouter.reLaunch({
                  path: cookie.get('redirect').replace(/\ /g, ''),
                })
              })
              .catch(error => {
                console.log('获取用户信息失败')
                this.getLoginCode()
              })
          })
          .catch(error => {
            console.log(error)
            console.log('登录接口调用失败')
            this.getLoginCode()
          })
      } else {
        uni.showToast({
          title: '已拒绝授权',
          icon: 'none',
          duration: 2000,
        })
      }
    },
  },
  mounted() {
    if (wx.getUserProfile) {
      this.canIUseGetUserProfile = true
      console.log(this.canIUseGetUserProfile)
    }
  },
}
</script>

<style lang="less">
.container {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  position: relative;
}

.force-login-wrap {
  width: 100%;
  height: 100%;
  overflow: hidden;
  z-index: 11111;
  top: 0;

  .logo-bg {
    width: 640rpx;
    height: 300rpx;
  }

  .force-login__content {
    position: absolute;
    left: 50%;
    top: 50%;
    transform: translate(-50%, -50%);

    .user-avatar {
      width: 160rpx;
      height: 160rpx;
      border-radius: 50%;
      overflow: hidden;
      margin-bottom: 40rpx;
    }

    .user-name {
      font-size: 35rpx;
      font-family: PingFang SC;
      font-weight: bold;
      color: #000;
      margin-bottom: 30rpx;
    }

    .login-notice {
      font-size: 28rpx;
      font-family: PingFang SC;
      font-weight: 400;
      color: #000;
      line-height: 44rpx;
      width: 500rpx;
      text-align: center;
      margin-bottom: 80rpx;
    }

    .author-btn {
      width: 630rpx;
      height: 80rpx;
      background: linear-gradient(to right, #f35447 0%, #ff8e3c 100%);
      background: -moz-linear-gradient(to right, #f35447 0%, #ff8e3c 100%);
      // box-shadow: 0px 7rpx 6rpx 0px rgba(229, 138, 0, 0.22);
      border-radius: 40rpx;
      font-size: 30rpx;
      font-family: PingFang SC;
      font-weight: 500;
      color: rgba(255, 255, 255, 1);
    }

    .close-btn {
      width: 630rpx;
      height: 80rpx;
      margin-top: 30rpx;
      border-radius: 40rpx;
      border: 2rpx solid #eb3729;
      background: none;
      font-size: 30rpx;
      font-family: PingFang SC;
      font-weight: 500;
      color: #eb3729;
    }
  }
}
</style>
