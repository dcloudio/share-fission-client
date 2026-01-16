<template>
	<view class="exchange-order">
		<!-- 状态筛选 -->
		<view class="filter-tabs">
			<view
				v-for="(tab, index) in statusTabs"
				:key="index"
				class="tab-item"
				:class="{ active: currentStatus === tab.value }"
				@click="changeStatus(tab.value)"
			>
				<text class="tab-text">{{ tab.label }}</text>
			</view>
		</view>

		<!-- 订单列表 -->
		<view class="order-list">
			<view v-if="filteredOrders.length === 0" class="empty-state">
				<uni-icons type="list" size="60" color="#cccccc"></uni-icons>
				<text class="empty-text">暂无兑换记录</text>
			</view>

			<view
				v-for="(order, index) in filteredOrders"
				:key="order.id"
				class="order-item"
			>
				<!-- 订单头部 -->
				<view class="order-header">
					<text class="order-id">订单号：{{ order.id }}</text>
					<view class="status-tag" :class="getStatusClass(order.status)">
						<text class="status-text">{{ getStatusText(order.status) }}</text>
					</view>
				</view>

				<!-- 商品信息 -->
				<view class="order-product">
					<image class="product-image" :src="order.productImage" mode="aspectFill"></image>
					<view class="product-info">
						<text class="product-name">{{ order.productName }}</text>
						<view class="product-points">
							<text class="points-number">{{ order.points }}</text>
							<text class="points-unit">积分</text>
						</view>
					</view>
				</view>

				<!-- 卡密信息 -->
				<view v-if="order.status === 'success'" class="card-info">
					<view class="info-row">
						<text class="info-label">兑换卡密</text>
						<view class="card-number-wrapper">
							<text class="card-number">{{ order.cardNumber }}</text>
							<view class="copy-btn" @click="copyCardNumber(order.cardNumber)">
								<uni-icons type="copy" size="14" color="#007AFF"></uni-icons>
								<text class="copy-text">复制</text>
							</view>
						</view>
					</view>
					<view class="info-row">
						<text class="info-label">兑换时间</text>
						<text class="info-value">{{ formatTime(order.createTime) }}</text>
					</view>
					<view class="usage-tip">
						<uni-icons type="info" size="14" color="#FF9500"></uni-icons>
						<text class="tip-text">请妥善保管卡密，使用后不可退换</text>
					</view>
				</view>

				<!-- 处理中/失败提示 -->
				<view v-if="order.status === 'pending'" class="order-tip pending">
					<uni-icons type="spinner-cycle" size="16" color="#FF9500"></uni-icons>
					<text class="tip-text">订单处理中，请稍后查看</text>
				</view>

				<view v-if="order.status === 'failed'" class="order-tip failed">
					<uni-icons type="closeempty" size="16" color="#FF3B30"></uni-icons>
					<text class="tip-text">兑换失败，积分已退回</text>
				</view>

				<!-- 订单时间 -->
				<view class="order-footer">
					<text class="footer-time">{{ formatTime(order.createTime) }}</text>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			// 状态筛选
			statusTabs: [
				{ label: '全部', value: 'all' },
				{ label: '兑换成功', value: 'success' },
				{ label: '处理中', value: 'pending' },
				{ label: '兑换失败', value: 'failed' }
			],
			currentStatus: 'all',

			// 订单列表
			orders: []
		}
	},
	computed: {
		// 过滤后的订单
		filteredOrders() {
			if (this.currentStatus === 'all') {
				return this.orders
			}
			return this.orders.filter(order => order.status === this.currentStatus)
		}
	},
	onLoad() {
		this.loadOrders()
	},
	onShow() {
		// 每次显示页面时重新加载数据
		this.loadOrders()
	},
	methods: {
		/**
		 * 加载订单列表
		 */
		loadOrders() {
			try {
				const orders = uni.getStorageSync('exchange_orders') || []
				this.orders = orders

				// 如果没有订单，添加虚拟数据供演示
				if (this.orders.length === 0) {
					this.orders = this.generateMockOrders()
				}
			} catch (error) {
				console.error('加载订单失败:', error)
				this.orders = this.generateMockOrders()
			}
		},

		/**
		 * 生成虚拟订单数据
		 */
		generateMockOrders() {
			const now = Date.now()
			return [
				{
					id: 'EO' + (now - 86400000 * 5),
					productId: 'P001',
					productName: '爱奇艺会员月卡',
					productImage: 'https://via.placeholder.com/300x300/667eea/ffffff?text=爱奇艺',
					points: 1500,
					cardNumber: 'AQYK-2H8F-9NMT-K7PL',
					status: 'success',
					createTime: now - 86400000 * 5
				},
				{
					id: 'EO' + (now - 86400000 * 2),
					productId: 'P003',
					productName: '王者荣耀点券',
					productImage: 'https://via.placeholder.com/300x300/feca57/333333?text=王者荣耀',
					points: 1000,
					cardNumber: 'WZRY-3K9L-6HJM-N4RT',
					status: 'success',
					createTime: now - 86400000 * 2
				},
				{
					id: 'EO' + (now - 3600000),
					productId: 'P005',
					productName: '10元话费充值',
					productImage: 'https://via.placeholder.com/300x300/48dbfb/ffffff?text=话费',
					points: 1000,
					status: 'pending',
					createTime: now - 3600000
				}
			]
		},

		/**
		 * 切换状态
		 */
		changeStatus(status) {
			this.currentStatus = status
		},

		/**
		 * 获取状态样式类
		 */
		getStatusClass(status) {
			return status
		},

		/**
		 * 获取状态文本
		 */
		getStatusText(status) {
			const statusMap = {
				success: '兑换成功',
				pending: '处理中',
				failed: '兑换失败'
			}
			return statusMap[status] || '未知'
		},

		/**
		 * 复制卡密
		 */
		copyCardNumber(cardNumber) {
			uni.setClipboardData({
				data: cardNumber,
				success: () => {
					uni.showToast({
						title: '卡密已复制',
						icon: 'success'
					})
				},
				fail: () => {
					uni.showToast({
						title: '复制失败',
						icon: 'none'
					})
				}
			})
		},

		/**
		 * 格式化时间
		 */
		formatTime(timestamp) {
			const date = new Date(timestamp)
			const year = date.getFullYear()
			const month = String(date.getMonth() + 1).padStart(2, '0')
			const day = String(date.getDate()).padStart(2, '0')
			const hour = String(date.getHours()).padStart(2, '0')
			const minute = String(date.getMinutes()).padStart(2, '0')

			return `${year}-${month}-${day} ${hour}:${minute}`
		}
	}
}
</script>

<style lang="scss" scoped>
/* #ifndef APP-NVUE */
view {
	display: flex;
	box-sizing: border-box;
	flex-direction: column;
}
/* #endif */

.exchange-order {
	min-height: 100vh;
	background: #f5f5f5;
	padding-bottom: 40rpx;
}

/* 筛选标签 */
.filter-tabs {
	background: #FFFFFF;
	flex-direction: row;
	padding: 20rpx;
	margin-bottom: 20rpx;
}

.tab-item {
	flex: 1;
	height: 60rpx;
	align-items: center;
	justify-content: center;
	border-radius: 30rpx;
}

.tab-item.active {
	background: rgba(102, 126, 234, 0.1);
}

.tab-text {
	font-size: 28rpx;
	color: #666666;
}

.tab-item.active .tab-text {
	color: #667eea;
	font-weight: 600;
}

/* 订单列表 */
.order-list {
	padding: 0 20rpx;
}

.empty-state {
	background: #FFFFFF;
	border-radius: 20rpx;
	padding: 100rpx 0;
	align-items: center;
	justify-content: center;
}

.empty-text {
	font-size: 28rpx;
	color: #999999;
	margin-top: 20rpx;
}

.order-item {
	background: #FFFFFF;
	border-radius: 20rpx;
	padding: 30rpx;
	margin-bottom: 20rpx;
}

/* 订单头部 */
.order-header {
	flex-direction: row;
	justify-content: space-between;
	align-items: center;
	margin-bottom: 20rpx;
	padding-bottom: 20rpx;
	border-bottom: 1px solid #f0f0f0;
}

.order-id {
	font-size: 26rpx;
	color: #999999;
}

.status-tag {
	padding: 8rpx 20rpx;
	border-radius: 20rpx;
}

.status-tag.success {
	background: rgba(52, 199, 89, 0.1);
}

.status-tag.pending {
	background: rgba(255, 149, 0, 0.1);
}

.status-tag.failed {
	background: rgba(255, 59, 48, 0.1);
}

.status-text {
	font-size: 24rpx;
}

.status-tag.success .status-text {
	color: #34C759;
}

.status-tag.pending .status-text {
	color: #FF9500;
}

.status-tag.failed .status-text {
	color: #FF3B30;
}

/* 商品信息 */
.order-product {
	flex-direction: row;
	align-items: center;
	margin-bottom: 20rpx;
}

.product-image {
	width: 120rpx;
	height: 120rpx;
	border-radius: 12rpx;
	margin-right: 20rpx;
}

.product-info {
	flex: 1;
}

.product-name {
	font-size: 30rpx;
	color: #333333;
	font-weight: 500;
	margin-bottom: 12rpx;
}

.product-points {
	flex-direction: row;
	align-items: baseline;
}

.points-number {
	font-size: 32rpx;
	font-weight: bold;
	color: #ff6b6b;
	margin-right: 4rpx;
}

.points-unit {
	font-size: 24rpx;
	color: #ff6b6b;
}

/* 卡密信息 */
.card-info {
	background: #f8f8f8;
	border-radius: 12rpx;
	padding: 20rpx;
	margin-bottom: 16rpx;
}

.info-row {
	flex-direction: row;
	justify-content: space-between;
	align-items: center;
	margin-bottom: 16rpx;
}

.info-row:last-child {
	margin-bottom: 0;
}

.info-label {
	font-size: 26rpx;
	color: #666666;
}

.info-value {
	font-size: 26rpx;
	color: #333333;
}

.card-number-wrapper {
	flex-direction: row;
	align-items: center;
}

.card-number {
	font-size: 26rpx;
	color: #333333;
	font-weight: 500;
	font-family: 'Courier New', monospace;
	margin-right: 12rpx;
}

.copy-btn {
	flex-direction: row;
	align-items: center;
	padding: 6rpx 12rpx;
	background: rgba(0, 122, 255, 0.1);
	border-radius: 20rpx;
}

.copy-text {
	font-size: 24rpx;
	color: #007AFF;
	margin-left: 4rpx;
}

.usage-tip {
	flex-direction: row;
	align-items: center;
	padding-top: 16rpx;
	border-top: 1px dashed #e0e0e0;
}

.tip-text {
	font-size: 24rpx;
	color: #999999;
	margin-left: 8rpx;
}

/* 订单提示 */
.order-tip {
	flex-direction: row;
	align-items: center;
	padding: 16rpx;
	border-radius: 8rpx;
	margin-bottom: 16rpx;
}

.order-tip.pending {
	background: rgba(255, 149, 0, 0.1);
}

.order-tip.pending .tip-text {
	color: #FF9500;
}

.order-tip.failed {
	background: rgba(255, 59, 48, 0.1);
}

.order-tip.failed .tip-text {
	color: #FF3B30;
}

.order-tip .tip-text {
	font-size: 26rpx;
	margin-left: 8rpx;
}

/* 订单底部 */
.order-footer {
	padding-top: 16rpx;
	border-top: 1px solid #f8f8f8;
}

.footer-time {
	font-size: 24rpx;
	color: #999999;
}
</style>
