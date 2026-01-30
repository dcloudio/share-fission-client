<template>
	<view class="ucenter">
		<!-- #ifndef H5 -->
		<statusBar></statusBar>
		<!-- #endif -->
		<uni-sign-in ref="signIn"></uni-sign-in>

		<!-- 用户信息卡片 -->
		<view class="user-card" @click.capture="toUserInfo">
			<view class="user-avatar">
				<cloud-image width="120rpx" height="120rpx" v-if="hasLogin && userInfo.avatar_file && userInfo.avatar_file.url" :src="userInfo.avatar_file.url"></cloud-image>
				<view v-else class="default-avatar">
					<uni-icons color="#ffffff" size="40" type="person-filled" />
				</view>
			</view>
			<view class="user-info">
				<text class="user-name">{{ hasLogin ? (userInfo.nickname || userInfo.username || userInfo.mobile) : '未登录' }}</text>
				<text class="user-tip">{{ hasLogin ? '点击查看个人信息' : '点击登录账号' }}</text>
			</view>
			<uni-icons type="right" size="18" color="#cccccc"></uni-icons>
		</view>

		<!-- 功能入口 -->
		<view class="section-card">
			<view class="section-title">
				<text class="title-text">常用功能</text>
			</view>
			<view class="menu-grid">
				<!-- #ifdef APP-PLUS -->
				<view class="menu-item" @click="signInByAd">
					<view class="menu-icon" style="background: linear-gradient(135deg, #FF9500 0%, #FF6B00 100%);">
						<uni-icons type="compose" size="22" color="#FFFFFF"></uni-icons>
					</view>
					<text class="menu-text">看广告签到</text>
				</view>
				<view class="menu-item" @click="gotoMarket">
					<view class="menu-icon" style="background: linear-gradient(135deg, #FFCC00 0%, #FF9500 100%);">
						<uni-icons type="star" size="22" color="#FFFFFF"></uni-icons>
					</view>
					<text class="menu-text">去评分</text>
				</view>
				<!-- #endif -->
				<view class="menu-item" @click="getScore">
					<view class="menu-icon" style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);">
						<uni-icons type="paperplane" size="22" color="#FFFFFF"></uni-icons>
					</view>
					<text class="menu-text">我的积分</text>
				</view>
				<view class="menu-item" @click="navigateTo('/pages/ucenter/withdraw/withdraw')">
					<view class="menu-icon" style="background: linear-gradient(135deg, #11998e 0%, #38ef7d 100%);">
						<uni-icons type="wallet" size="22" color="#FFFFFF"></uni-icons>
					</view>
					<text class="menu-text">积分提现</text>
				</view>
				<view class="menu-item" @click="navigateTo('/pages/ucenter/distribution-invite/distribution-invite')">
					<view class="menu-icon" style="background: linear-gradient(135deg, #ee0979 0%, #ff6a00 100%);">
						<uni-icons type="redo" size="22" color="#FFFFFF"></uni-icons>
					</view>
					<text class="menu-text">分销推荐</text>
				</view>
			</view>
		</view>

		<!-- 其他功能列表 -->
		<view class="section-card">
			<view class="section-title">
				<text class="title-text">更多服务</text>
			</view>
			<view class="menu-list">
				<view class="list-item" @click="navigateTo('/uni_modules/uni-feedback/pages/opendb-feedback/opendb-feedback')">
					<view class="item-left">
						<uni-icons type="help" size="20" color="#007AFF"></uni-icons>
						<text class="item-text">问题与反馈</text>
					</view>
					<uni-icons type="right" size="16" color="#cccccc"></uni-icons>
				</view>
				<view class="list-item" @click="navigateTo('/pages/ucenter/settings/settings')">
					<view class="item-left">
						<uni-icons type="gear" size="20" color="#007AFF"></uni-icons>
						<text class="item-text">设置</text>
					</view>
					<uni-icons type="right" size="16" color="#cccccc"></uni-icons>
				</view>
				<!-- #ifdef APP-PLUS -->
				<view class="list-item" @click="checkVersion">
					<view class="item-left">
						<uni-icons type="loop" size="20" color="#007AFF"></uni-icons>
						<text class="item-text">检查更新</text>
					</view>
					<view class="item-right">
						<text class="item-version">{{ appVersion.version }}-{{ appVersion.versionCode }}</text>
						<view v-if="appVersion.hasNew" class="update-badge"></view>
						<uni-icons type="right" size="16" color="#cccccc"></uni-icons>
					</view>
				</view>
				<view class="list-item" @click="navigateTo('/pages/ucenter/about/about')">
					<view class="item-left">
						<uni-icons type="info" size="20" color="#007AFF"></uni-icons>
						<text class="item-text">关于</text>
					</view>
					<uni-icons type="right" size="16" color="#cccccc"></uni-icons>
				</view>
				<!-- #endif -->
			</view>
		</view>
	</view>
</template>

<script>
	import statusBar from "@/uni_modules/uni-nav-bar/components/uni-nav-bar/uni-status-bar";
	import checkUpdate from '@/uni_modules/uni-upgrade-center-app/utils/check-update';
	import callCheckVersion from '@/uni_modules/uni-upgrade-center-app/utils/call-check-version';
	// #ifdef APP
	import UniShare from '@/uni_modules/uni-share/js_sdk/uni-share.js';
	const uniShare = new UniShare()
	// #endif
	const db = uniCloud.database();
	import {
		store,
		mutations
	} from '@/uni_modules/uni-id-pages/common/store.js'
	export default {
		components: {
			statusBar
		},
		// #ifdef APP
		onBackPress({from}) {
			if(from=='backbutton'){
				this.$nextTick(function(){
					uniShare.hide()
				})
				return uniShare.isShow;
			}
		},
		// #endif
		data() {
			return {}
		},
		onLoad() {},
		onShow() {},
		computed: {
			userInfo() {
				return store.userInfo
			},
			hasLogin(){
				return store.hasLogin
			},
			// #ifdef APP-PLUS
			appVersion() {
				return getApp().appVersion
			},
			// #endif
			appConfig() {
				return getApp().globalData.config
			}
		},
		methods: {
			navigateTo(url) {
				uni.navigateTo({ url })
			},
			toSettings() {
				uni.navigateTo({
					url: "/pages/ucenter/settings/settings"
				})
			},
			signIn() {
				this.$refs.signIn.open()
			},
			signInByAd(){
				this.$refs.signIn.showRewardedVideoAd()
			},
			async checkVersion() {
				let res = await callCheckVersion()
				if (res.result.code > 0) {
					checkUpdate()
				} else {
					uni.showToast({
						title: res.result.message,
						icon: 'none'
					});
				}
			},
			toUserInfo() {
				uni.navigateTo({
					url: '/uni_modules/uni-id-pages/pages/userinfo/userinfo'
				})
			},
			gotoMarket() {
				// #ifdef APP-PLUS
				if (uni.getSystemInfoSync().platform == "ios") {
					let appstoreid = this.appConfig.marketId.ios;
					plus.runtime.openURL("itms-apps://" + 'itunes.apple.com/cn/app/wechat/' + appstoreid + '?mt=8', err => {});
				}
				if (uni.getSystemInfoSync().platform == "android") {
					var Uri = plus.android.importClass("android.net.Uri");
					var uri = Uri.parse("market://details?id=" + this.appConfig.marketId.android);
					var Intent = plus.android.importClass('android.content.Intent');
					var intent = new Intent(Intent.ACTION_VIEW, uri);
					var main = plus.android.runtimeMainActivity();
					main.startActivity(intent);
				}
				// #endif
			},
			getScore() {
				if (!this.userInfo) return uni.showToast({
					title: "请登录后查看积分",
					icon: 'none'
				});
				uni.showLoading({
					mask: true
				})
				db.collection("uni-id-scores")
					.where('"user_id" == $env.uid')
					.field('score,balance')
					.orderBy("create_date", "desc")
					.limit(1)
					.get()
					.then((res) => {
						const data = res.result.data[0];
						let msg = '';
						msg = data ? ("当前积分为"+ data.balance) : "当前无积分";
						uni.showToast({
							title: msg,
							icon: 'none'
						});
					}).finally(()=>{
						uni.hideLoading()
					})
			},
			async share() {
				let {result} = await db.collection('uni-id-users').where("'_id' == $cloudEnv_uid").field('my_invite_code').get()
				let myInviteCode = result.data[0].my_invite_code
				if(!myInviteCode){
					return uni.showToast({
						title: '请检查uni-config-center中uni-id配置，是否已启用 autoSetInviteCode',
						icon: 'none'
					});
				}
				let {
					appName,
					logo,
					company,
					slogan
				} = this.appConfig.about
				// #ifdef APP
				uniShare.show({
					content: {
						type: 0,
						href: this.appConfig.h5.url +
							`/#/pages/ucenter/invite/invite?code=uniInvitationCode:${myInviteCode}`,
						title: appName,
						summary: slogan,
						imageUrl: logo +
							'?x-oss-process=image/resize,m_fill,h_100,w_100'
					},
					menus: [{
							"img": "/static/app/sharemenu/wechatfriend.png",
							"text": "微信好友",
							"share": {
								"provider": "weixin",
								"scene": "WXSceneSession"
							}
						},
						{
							"img": "/static/app/sharemenu/wechatmoments.png",
							"text": "微信朋友圈",
							"share": {
								"provider": "weixin",
								"scene": "WXSceneTimeline"
							}
						},
						{
							"img": "/static/app/sharemenu/weibo.png",
							"text": "微博",
							"share": {
								"provider": "sinaweibo"
							}
						},
						{
							"img": "/static/app/sharemenu/qq.png",
							"text": "QQ",
							"share": {
								"provider": "qq"
							}
						},
						{
							"img": "/static/app/sharemenu/copyurl.png",
							"text": "复制",
							"share": "copyurl"
						},
						{
							"img": "/static/app/sharemenu/more.png",
							"text": "更多",
							"share": "shareSystem"
						}
					],
					cancelText: "取消分享",
				}, e => {})
				// #endif
			}
		}
	}
</script>

<style lang="scss" scoped>
view {
	display: flex;
	box-sizing: border-box;
	flex-direction: column;
}

.ucenter {
	min-height: 100vh;
	background: #f5f5f5;
	padding-bottom: 40rpx;
}

/* 用户信息卡片 */
.user-card {
	background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
	margin: 20rpx;
	padding: 40rpx 30rpx;
	border-radius: 20rpx;
	flex-direction: row;
	align-items: center;
	box-shadow: 0 8rpx 30rpx rgba(102, 126, 234, 0.3);
}

.user-avatar {
	margin-right: 24rpx;
}

.default-avatar {
	width: 120rpx;
	height: 120rpx;
	background-color: rgba(255, 255, 255, 0.3);
	border-radius: 50%;
	justify-content: center;
	align-items: center;
}

.user-info {
	flex: 1;
}

.user-name {
	font-size: 18px;
	font-weight: bold;
	color: #FFFFFF;
	margin-bottom: 8rpx;
}

.user-tip {
	font-size: 12px;
	color: rgba(255, 255, 255, 0.8);
}

/* 卡片通用样式 */
.section-card {
	background: #FFFFFF;
	margin: 20rpx;
	border-radius: 20rpx;
	overflow: hidden;
}

.section-title {
	padding: 24rpx 30rpx;
	border-bottom: 1px solid #f0f0f0;
}

.title-text {
	font-size: 15px;
	font-weight: 600;
	color: #333333;
}

/* 功能入口网格 */
.menu-grid {
	flex-direction: row;
	flex-wrap: wrap;
	padding: 20rpx 10rpx;
}

.menu-item {
	width: 25%;
	align-items: center;
	padding: 20rpx 0;
}

.menu-icon {
	width: 90rpx;
	height: 90rpx;
	border-radius: 20rpx;
	justify-content: center;
	align-items: center;
	margin-bottom: 12rpx;
}

.menu-text {
	font-size: 12px;
	color: #666666;
}

/* 列表样式 */
.menu-list {
	padding: 0 30rpx;
}

.list-item {
	flex-direction: row;
	align-items: center;
	justify-content: space-between;
	padding: 30rpx 0;
	border-bottom: 1px solid #f5f5f5;
}

.list-item:last-child {
	border-bottom: none;
}

.item-left {
	flex-direction: row;
	align-items: center;
}

.item-text {
	font-size: 14px;
	color: #333333;
	margin-left: 16rpx;
}

.item-right {
	flex-direction: row;
	align-items: center;
}

.item-version {
	font-size: 12px;
	color: #999999;
	margin-right: 8rpx;
}

.update-badge {
	width: 16rpx;
	height: 16rpx;
	border-radius: 50%;
	background-color: #DD524D;
	margin-right: 8rpx;
}
</style>
