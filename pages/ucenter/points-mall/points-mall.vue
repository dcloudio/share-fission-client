<template>
	<view class="points-mall">
		<!-- 顶部积分余额 -->
		<view class="header-balance">
			<view class="balance-info">
				<text class="balance-label">我的积分</text>
				<text class="balance-number">{{ userPoints }}</text>
			</view>
			<view class="order-link" @click="toExchangeOrder">
				<text class="order-text">兑换记录</text>
				<uni-icons type="right" size="14" color="#FFFFFF"></uni-icons>
			</view>
		</view>

		<!-- 分类筛选 -->
		<view class="category-filter">
			<scroll-view scroll-x class="category-scroll">
				<view class="category-list">
					<view
						v-for="(category, index) in categories"
						:key="index"
						class="category-item"
						:class="{ active: currentCategory === category.value }"
						@click="selectCategory(category.value)"
					>
						<text class="category-text">{{ category.label }}</text>
					</view>
				</view>
			</scroll-view>
		</view>

		<!-- 排序 -->
		<view class="sort-bar">
			<view class="sort-left">
				<text class="result-count">共 {{ filteredProducts.length }} 件商品</text>
			</view>
			<view class="sort-right">
				<view
					class="sort-item"
					:class="{ active: sortType === 'asc' }"
					@click="changeSortType('asc')"
				>
					<text class="sort-text">积分</text>
					<uni-icons type="up" size="12" :color="sortType === 'asc' ? '#007AFF' : '#999999'"></uni-icons>
				</view>
				<view
					class="sort-item"
					:class="{ active: sortType === 'desc' }"
					@click="changeSortType('desc')"
				>
					<text class="sort-text">积分</text>
					<uni-icons type="down" size="12" :color="sortType === 'desc' ? '#007AFF' : '#999999'"></uni-icons>
				</view>
			</view>
		</view>

		<!-- 商品列表 -->
    <scroll-view class="product-list" scroll-y>
      <view class="scroll-content">
        <view
        	v-for="(product, index) in sortedProducts"
        	:key="product.id"
        	class="product-item"
        	@click="toProductDetail(product.id)"
        >
        	<view class="product-image-wrapper">
        		<image class="product-image" :src="product.image" mode="aspectFill"></image>
        		<view v-if="product.stock <= 10" class="stock-tag">
        			<text class="stock-text">仅剩{{ product.stock }}件</text>
        		</view>
        	</view>
        	<view class="product-info">
        		<text class="product-name">{{ product.name }}</text>
        		<text class="product-desc">{{ product.desc }}</text>
        		<view class="product-footer">
        			<view class="product-points">
        				<text class="points-number">{{ product.points }}</text>
        				<text class="points-unit">积分</text>
        			</view>
        			<text class="exchange-count">已兑{{ product.soldCount }}</text>
        		</view>
        	</view>
        </view>
      </view>
    </scroll-view>

		<!-- 空状态 -->
		<view v-if="filteredProducts.length === 0" class="empty-state">
			<uni-icons type="shop" size="60" color="#cccccc"></uni-icons>
			<text class="empty-text">暂无商品</text>
		</view>
	</view>
</template>

<script>
const db = uniCloud.database();

export default {
	data() {
		return {
			// 用户积分
			userPoints: 5000,

			// 分类
			categories: [
				{ label: '全部', value: 'all' },
				{ label: '会员卡', value: 'vip' },
				{ label: '游戏币', value: 'game' },
				{ label: '优惠券', value: 'coupon' },
				{ label: '话费充值', value: 'phone' },
				{ label: '视频会员', value: 'video' }
			],
			currentCategory: 'all',

			// 排序
			sortType: 'asc', // asc: 积分从低到高, desc: 积分从高到低

			// 商品列表（虚拟数据）
			products: [
				{
					id: 'P001',
					name: '爱奇艺会员月卡',
					desc: '30天畅享海量影视',
					category: 'video',
					points: 1500,
					stock: 50,
					soldCount: 1230,
					image: 'https://via.placeholder.com/300x300/667eea/ffffff?text=爱奇艺',
					images: [
						'https://via.placeholder.com/750x750/667eea/ffffff?text=爱奇艺1',
						'https://via.placeholder.com/750x750/764ba2/ffffff?text=爱奇艺2'
					],
					detail: '爱奇艺黄金VIP会员月卡，30天有效期，畅享海量影视资源、蓝光画质、广告特权等。兑换成功后，卡密将在订单详情中显示。'
				},
				{
					id: 'P002',
					name: '腾讯视频会员月卡',
					desc: '好剧抢先看',
					category: 'video',
					points: 1500,
					stock: 100,
					soldCount: 980,
					image: 'https://via.placeholder.com/300x300/ff6b6b/ffffff?text=腾讯视频',
					images: [
						'https://via.placeholder.com/750x750/ff6b6b/ffffff?text=腾讯1',
						'https://via.placeholder.com/750x750/ee5a6f/ffffff?text=腾讯2'
					],
					detail: '腾讯视频VIP会员月卡，30天有效期，热播剧集抢先看，海量内容随心享。'
				},
				{
					id: 'P003',
					name: '王者荣耀点券',
					desc: '1000点券',
					category: 'game',
					points: 1000,
					stock: 8,
					soldCount: 2340,
					image: 'https://via.placeholder.com/300x300/feca57/333333?text=王者荣耀',
					images: [
						'https://via.placeholder.com/750x750/feca57/333333?text=王者1',
						'https://via.placeholder.com/750x750/ff9ff3/333333?text=王者2'
					],
					detail: '王者荣耀1000点券充值卡，可用于购买皮肤、英雄等游戏道具。兑换后请在游戏内使用卡密充值。'
				},
				{
					id: 'P004',
					name: '网易云音乐黑胶VIP',
					desc: '月度会员',
					category: 'vip',
					points: 800,
					stock: 200,
					soldCount: 567,
					image: 'https://via.placeholder.com/300x300/ee5a6f/ffffff?text=网易云',
					images: [
						'https://via.placeholder.com/750x750/ee5a6f/ffffff?text=网易云1',
						'https://via.placeholder.com/750x750/c23616/ffffff?text=网易云2'
					],
					detail: '网易云音乐黑胶VIP月卡，畅享无损音质、下载歌曲、个性化推荐等特权。'
				},
				{
					id: 'P005',
					name: '10元话费充值',
					desc: '全国通用',
					category: 'phone',
					points: 1000,
					stock: 500,
					soldCount: 3450,
					image: 'https://via.placeholder.com/300x300/48dbfb/ffffff?text=话费',
					images: [
						'https://via.placeholder.com/750x750/48dbfb/ffffff?text=话费1',
						'https://via.placeholder.com/750x750/0abde3/ffffff?text=话费2'
					],
					detail: '10元话费充值，支持移动、联通、电信三网，全国通用。充值成功后5分钟内到账。'
				},
				{
					id: 'P006',
					name: '50元优惠券',
					desc: '满100可用',
					category: 'coupon',
					points: 500,
					stock: 1000,
					soldCount: 890,
					image: 'https://via.placeholder.com/300x300/5f27cd/ffffff?text=优惠券',
					images: [
						'https://via.placeholder.com/750x750/5f27cd/ffffff?text=券1',
						'https://via.placeholder.com/750x750/341f97/ffffff?text=券2'
					],
					detail: '50元通用优惠券，满100元可用，适用于全平台商品，有效期30天。'
				},
				{
					id: 'P007',
					name: '喜马拉雅会员月卡',
					desc: '听书神器',
					category: 'vip',
					points: 600,
					stock: 150,
					soldCount: 432,
					image: 'https://via.placeholder.com/300x300/ff9ff3/333333?text=喜马拉雅',
					images: [
						'https://via.placeholder.com/750x750/ff9ff3/333333?text=喜马1',
						'https://via.placeholder.com/750x750/f368e0/333333?text=喜马2'
					],
					detail: '喜马拉雅VIP会员月卡，畅听海量有声书、精品课程。'
				},
				{
					id: 'P008',
					name: '和平精英UC',
					desc: '600UC',
					category: 'game',
					points: 600,
					stock: 300,
					soldCount: 1876,
					image: 'https://via.placeholder.com/300x300/00d2d3/ffffff?text=和平精英',
					images: [
						'https://via.placeholder.com/750x750/00d2d3/ffffff?text=精英1',
						'https://via.placeholder.com/750x750/01a3a4/ffffff?text=精英2'
					],
					detail: '和平精英600UC充值，可用于购买服饰、道具等。兑换后凭卡密在游戏内充值。'
				}
			]
		}
	},
	computed: {
		// 过滤后的商品
		filteredProducts() {
			if (this.currentCategory === 'all') {
				return this.products
			}
			return this.products.filter(product => product.category === this.currentCategory)
		},

		// 排序后的商品
		sortedProducts() {
			const products = [...this.filteredProducts]
			if (this.sortType === 'asc') {
				return products.sort((a, b) => a.points - b.points)
			} else {
				return products.sort((a, b) => b.points - a.points)
			}
		}
	},
	onLoad() {
		this.loadUserPoints()
		// 保存商品数据到本地存储供详情页使用
		uni.setStorageSync('mall_products', this.products)
	},
	methods: {
		/**
		 * 加载用户积分
		 */
		async loadUserPoints() {
			try {
				const res = await db.collection("uni-id-scores")
					.where('"user_id" == $env.uid')
					.field('score,balance')
					.orderBy("create_date", "desc")
					.limit(1)
					.get()

				if (res.result.data && res.result.data.length > 0) {
					this.userPoints = res.result.data[0].balance || 0
				}
			} catch (error) {
				console.error('获取积分失败:', error)
				this.userPoints = 5000
			}
		},

		/**
		 * 选择分类
		 */
		selectCategory(category) {
			this.currentCategory = category
		},

		/**
		 * 切换排序
		 */
		changeSortType(type) {
			this.sortType = type
		},

		/**
		 * 跳转到商品详情
		 */
		toProductDetail(productId) {
			uni.navigateTo({
				url: `/pages/ucenter/product-detail/product-detail?id=${productId}`
			})
		},

		/**
		 * 跳转到兑换记录
		 */
		toExchangeOrder() {
			uni.navigateTo({
				url: '/pages/ucenter/exchange-order/exchange-order'
			})
		}
	}
}
</script>

<style lang="scss" scoped>
page, view {
	display: flex;
	box-sizing: border-box;
	flex-direction: column;
}

page {
	height: 100%;
	background: #f5f5f5;
}

.points-mall {
	flex: 1;
  height: 100%;
	background: #f5f5f5;
	overflow: hidden;
  flex-direction: column;

	/* 顶部积分余额 */
	.header-balance {
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		padding: 40rpx 30rpx;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;

		.balance-info {
			flex-direction: row;
			align-items: baseline;

			.balance-label {
				font-size: 14px;
				color: rgba(255, 255, 255, 0.9);
				margin-right: 16rpx;
			}

			.balance-number {
				font-size: 24px;
				font-weight: bold;
				color: #FFFFFF;
			}
		}

		.order-link {
			flex-direction: row;
			align-items: center;
			padding: 12rpx 24rpx;
			background: rgba(255, 255, 255, 0.2);
			border-radius: 30rpx;

			.order-text {
				font-size: 13px;
				color: #FFFFFF;
				margin-right: 4rpx;
			}
		}
	}

	/* 分类筛选 */
	.category-filter {
		background: #FFFFFF;
		padding: 20rpx 0;
		margin-bottom: 20rpx;

		.category-scroll {
			white-space: nowrap;

			.category-list {
				flex-direction: row;
				padding: 0 20rpx;

				.category-item {
					padding: 12rpx 32rpx;
					margin-right: 20rpx;
					background: #f8f8f8;
					border-radius: 30rpx;
					white-space: nowrap;

					.category-text {
						font-size: 14px;
						color: #333333;
						white-space: nowrap;
					}

					&.active {
						background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);

						.category-text {
							color: #FFFFFF;
							font-weight: 500;
						}
					}
				}
			}
		}
	}

	/* 排序栏 */
	.sort-bar {
		background: #FFFFFF;
		padding: 20rpx 30rpx;
		margin-bottom: 20rpx;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;

		.sort-left {
			.result-count {
				font-size: 13px;
				color: #999999;
			}
		}

		.sort-right {
			flex-direction: row;
			align-items: center;

			.sort-item {
				flex-direction: row;
				align-items: center;
				margin-left: 20rpx;
				padding: 8rpx 16rpx;
				border-radius: 20rpx;
				.sort-text {
					font-size: 13px;
					color: #666666;
					margin-right: 4rpx;
				}
				&.active {
					background: rgba(0, 122, 255, 0.1);
					.sort-text {
						color: #007AFF;
						font-weight: 500;
					}
				}
			}
		}
	}

	/* 商品列表 */
	.product-list {
		height: 0;
		flex: 1;

		.scroll-content {
			flex-direction: row;
			flex-wrap: wrap;
			.product-item {
				width: 345rpx;
        margin-left: 20rpx;
				background: #FFFFFF;
				border-radius: 16rpx;
				margin-bottom: 20rpx;
				overflow: hidden;
				.product-image-wrapper {
					width: 345rpx;
					height: 345rpx;
					position: relative;

					.product-image {
						width: 100%;
						height: 100%;
					}

					.stock-tag {
						position: absolute;
						top: 16rpx;
						right: 0;
						background: rgba(255, 107, 107, 0.9);
						padding: 8rpx 16rpx;
						border-radius: 30rpx 0 0 30rpx;

						.stock-text {
							font-size: 11px;
							color: #FFFFFF;
						}
					}
				}

				.product-info {
					padding: 20rpx;

					.product-name {
						font-size: 15px;
						color: #333333;
						font-weight: 500;
						margin-bottom: 8rpx;
						overflow: hidden;
						text-overflow: ellipsis;
						white-space: nowrap;
					}

					.product-desc {
						font-size: 12px;
						color: #999999;
						margin-bottom: 16rpx;
						overflow: hidden;
						text-overflow: ellipsis;
						white-space: nowrap;
					}

					.product-footer {
						flex-direction: row;
						justify-content: space-between;
						align-items: flex-end;

						.product-points {
							flex-direction: row;
							align-items: baseline;

							.points-number {
								font-size: 18px;
								font-weight: bold;
								color: #ff6b6b;
								margin-right: 4rpx;
							}

							.points-unit {
								font-size: 12px;
								color: #ff6b6b;
							}
						}

						.exchange-count {
							font-size: 12px;
							color: #999999;
						}
					}
				}
			}
		}
	}

	/* 空状态 */
	.empty-state {
		background: #FFFFFF;
		border-radius: 20rpx;
		padding: 100rpx 0;
		margin: 0 20rpx;
		align-items: center;
		justify-content: center;

		.empty-text {
			font-size: 14px;
			color: #999999;
			margin-top: 20rpx;
		}
	}
}
</style>
