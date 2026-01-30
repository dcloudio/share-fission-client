<template>
	<view class="points-mall">
		<!-- 顶部积分余额 -->
		<view class="header-balance">
      <!-- #ifndef H5 -->
      <statusBar></statusBar>
      <!-- #endif -->
      <view class="header-balance-content">
        <view class="balance-info">
        	<text class="balance-label">我的积分</text>
        	<text class="balance-number">{{ userPoints }}</text>
        </view>
        <view class="order-link" @click="toExchangeOrder">
        	<text class="order-text">兑换记录</text>
        	<uni-icons type="right" size="14" color="#FFFFFF"></uni-icons>
        </view>
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
import statusBar from "@/uni_modules/uni-nav-bar/components/uni-nav-bar/uni-status-bar";
const db = uniCloud.database();
const sfCo = uniCloud.importObject('share-fission-co', { customUI: true });

export default {
	components: {
		statusBar
	},
	data() {
		return {
			// 用户积分
			userPoints: 5000,

			// 分类（后端动态加载，这里只保留“全部”兜底）
			categories: [
				{ label: '全部', value: 'all' }
			],
			currentCategory: 'all',

			// 排序
			sortType: 'asc', // asc: 积分从低到高, desc: 积分从高到低

			// 商品列表（由后端加载）
			products: [],
			// 后端分类ID映射（用于筛选）
			categoryMap: {
				all: ''
			},
			// 分页
			pageIndex: 1,
			pageSize: 200,
			loading: false,
			finished: false,
			// 缓存当前列表，用于详情页读取
			mallProductsCache: []
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
		this.loadCategoriesAndProducts()
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
				this.userPoints = 0
			}
		},

		/**
		 * 加载分类+商品
		 */
		async loadCategoriesAndProducts() {
			await this.loadCategories()
			await this.loadProducts(true)
		},

		/**
		 * 加载分类（client/goodsCategories/getList）
		 */
		async loadCategories() {
			try {
				const res = await sfCo.action({
					name: 'client/goodsCategories/getList',
					data: {
						pageIndex: 1,
						pageSize: 200,
						sortField: 'sort',
						sortOrder: 'asc'
					}
				})
				const list = (res && (res.list || (res.data && res.data.list))) || []

				// 组装分类：全部 + 后端分类
				const newCategories = [{ label: '全部', value: 'all' }]
				const map = { all: '' }

				list.forEach((c) => {
					if (c && c._id && c.name) {
						const val = String(c._id)
						newCategories.push({ label: String(c.name), value: val })
						map[val] = val
					}
				})

				this.categories = newCategories
				this.categoryMap = map
				// 默认选中全部
				this.currentCategory = 'all'
			} catch (e) {
				console.error('加载分类失败:', e)
				this.categories = [{ label: '全部', value: 'all' }]
				this.categoryMap = { all: '' }
				this.currentCategory = 'all'
			}
		},

		/**
		 * 加载商品（client/goods/getList）
		 */
		async loadProducts(reset) {
			if (this.loading || this.finished) return
			this.loading = true
			try {
				if (reset) {
					this.products = []
					this.pageIndex = 1
					this.finished = false
				}

				const categoryId = this.currentCategory === 'all' ? '' : this.currentCategory
				const res = await sfCo.action({
					name: 'client/goods/getList',
					data: {
						pageIndex: this.pageIndex,
						pageSize: this.pageSize,
						category_id: categoryId,
						sortField: 'sort_order',
						sortOrder: 'desc'
					}
				})
				const list = (res && (res.list || (res.data && res.data.list))) || []

				const mapped = list.map((g) => {
					return {
						id: String(g._id || ''),
						name: String(g.name || ''),
						desc: String(g.description || ''),
						category: String(g.category_id || ''),
						points: Number(g.score_cost || 0),
						stock: Number(g.stock || 0),
						soldCount: Number(g.sales_count || 0),
						image: (g.images && g.images.length > 0) ? String(g.images[0]) : '',
						images: (g.images && g.images.length > 0) ? g.images : [],
						detail: String(g.detail || '')
					}
				})

				this.products = this.products.concat(mapped)
				this.mallProductsCache = this.products
				uni.setStorageSync('mall_products', this.mallProductsCache)

				if (mapped.length < this.pageSize) {
					this.finished = true
				} else {
					this.pageIndex = this.pageIndex + 1
				}
			} catch (e) {
				console.error('加载商品失败:', e)
				if (reset) {
					this.products = []
					uni.setStorageSync('mall_products', [])
				}
			} finally {
				this.loading = false
			}
		},

		/**
		 * 选择分类
		 */
		selectCategory(category) {
			this.currentCategory = category
			// 重置分页并重新加载
			this.pageIndex = 1
			this.finished = false
			this.loadProducts(true)
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
    .header-balance-content {
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
