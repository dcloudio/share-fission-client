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
	onShow() {
		if (getApp().globalData.pointsNeedRefresh) {
			getApp().globalData.pointsNeedRefresh = false
			this.loadUserPoints()
		}
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
	background: #FAFAFA;
}

.points-mall {
	flex: 1;
  height: 100%;
	background: #FAFAFA;
	overflow: hidden;
  flex-direction: column;

	/* 顶部积分余额 */
	.header-balance {
		background: linear-gradient(135deg, #5B8FF9 0%, #3D6FD8 100%);
		padding: 24px 20px;
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
      		margin-right: 12px;
      	}

      	.balance-number {
      		font-size: 24px;
      		font-weight: 600;
      		color: #FFFFFF;
      	}
      }
    }

		.order-link {
			flex-direction: row;
			align-items: center;
			padding: 6px 16px;
			background: rgba(255, 255, 255, 0.15);
			border-radius: 999px;
			border: 1px solid rgba(255, 255, 255, 0.25);

			.order-text {
				font-size: 13px;
				color: #FFFFFF;
				margin-right: 4px;
			}
		}
	}

	/* 分类筛选 */
	.category-filter {
		background: #FFFFFF;
		padding: 12px 0;
		margin-bottom: 12px;

		.category-scroll {
			white-space: nowrap;

			.category-list {
				flex-direction: row;
				padding: 0 16px;

				.category-item {
					padding: 6px 20px;
					margin-right: 12px;
					background: #F5F5F5;
					border-radius: 999px;
					white-space: nowrap;

					.category-text {
						font-size: 14px;
						color: #595959;
						white-space: nowrap;
					}

					&.active {
						background: #5B8FF9;

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
		padding: 12px 20px;
		margin-bottom: 12px;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;

		.sort-left {
			.result-count {
				font-size: 13px;
				color: #8C8C8C;
			}
		}

		.sort-right {
			flex-direction: row;
			align-items: center;

			.sort-item {
				flex-direction: row;
				align-items: center;
				margin-left: 16px;
				padding: 4px 12px;
				border-radius: 4px;
				.sort-text {
					font-size: 13px;
					color: #595959;
					margin-right: 4px;
				}
				&.active {
					background: rgba(91, 143, 249, 0.1);
					.sort-text {
						color: #5B8FF9;
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
			padding: 0 16px;
			justify-content: space-between;

			.product-item {
				width: calc(50% - 8px);
				background: #FFFFFF;
				border-radius: 12px;
				margin-bottom: 16px;
				overflow: hidden;
				box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);

				.product-image-wrapper {
					width: 100%;
					padding-bottom: 100%;
					position: relative;

					.product-image {
						position: absolute;
						top: 0;
						left: 0;
						width: 100%;
						height: 100%;
					}

					.stock-tag {
						position: absolute;
						top: 12px;
						right: 0;
						background: rgba(245, 34, 45, 0.9);
						padding: 4px 12px;
						border-radius: 999px 0 0 999px;

						.stock-text {
							font-size: 12px;
							color: #FFFFFF;
						}
					}
				}

				.product-info {
					padding: 12px;

					.product-name {
						font-size: 15px;
						color: #262626;
						font-weight: 500;
						margin-bottom: 6px;
						overflow: hidden;
						text-overflow: ellipsis;
						white-space: nowrap;
					}

					.product-desc {
						font-size: 13px;
						color: #8C8C8C;
						margin-bottom: 12px;
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
								font-weight: 600;
								color: #F5222D;
								margin-right: 4px;
							}

							.points-unit {
								font-size: 13px;
								color: #F5222D;
							}
						}

						.exchange-count {
							font-size: 13px;
							color: #8C8C8C;
						}
					}
				}
			}
		}
	}

	/* 空状态 */
	.empty-state {
		background: #FFFFFF;
		border-radius: 12px;
		padding: 60px 0;
		margin: 0 16px;
		align-items: center;
		justify-content: center;

		.empty-text {
			font-size: 14px;
			color: #8C8C8C;
			margin-top: 16px;
		}
	}
}
</style>
