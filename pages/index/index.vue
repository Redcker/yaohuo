<template>
	<view v-if="!loading">
		<uni-transition mode-class="fade" :duration="300" :show="true">
			<view class="logo">
				<image src="https://yaohuo.me/tupian/yaohuo.png"></image>
			</view>
			<uni-search-bar bgColor="#fff" class="uni-mt-10" radius="8" placeholder="善用搜索" clearButton="auto"
				@confirm="search" @cancel="cancelSearch" />
			<view class="content">
				<view class="grid">
					<uni-grid :column="5" :showBorder="false" :square="false" :highlight="true" @change="gridChange">
						<uni-grid-item v-for="(item,index) in idArr" :index="parseInt(item.id)" :key="item.id">
							<view class="grid-item-box">
								<uni-icons :type="item.icon" :size="30" color="#777" />
								<text class="text">{{item.name}}</text>
							</view>
						</uni-grid-item>
					</uni-grid>
				</view>
				<view class="grid mt-20">
					<uni-grid :column="2" :showBorder="false" :square="false" :highlight="true" @change="goToWebview">
						<uni-grid-item v-for="(item,index) in actionArr" :index="index" :key="index">
							<view class="grid-item-box">
								<uni-icons :type="item.icon" :size="30" color="#777" />
								<text class="text">{{item.name}}</text>
							</view>
						</uni-grid-item>
					</uni-grid>
				</view>
				<view class="mt-20 br-8 f-15" style="background-color: #fff;">
					<uni-section titleFontSize="16px" title="最新回复" type="line"></uni-section>
					<uni-row>
						<uni-col :span="12" v-for="(item,index) in newArr" :key="index">
							<view class="new-post-item" @click="goToDetail(item.id)">
								{{index+1}}.{{item.title}}
							</view>
						</uni-col>
					</uni-row>
				</view>
				<view class="mt-20 br-8 recomment-card f-15">
					<uni-section titleFontSize="16px" title="老C推荐" type="line"></uni-section>
					<uni-row>
						<uni-col :span="12" v-for="(item,index) in recommedArr" :key="index">
							<view class="new-post-item" @click="goToRecommend(item)">
								<image class="img" :src="item.img"></image>
								{{item.title}}
							</view>
						</uni-col>
					</uni-row>
					<view style="position: absolute;top: 20rpx;right: 20rpx;" class="" @click="goToRecommend(extraObj)">
						<image class="img" :src="extraObj.img"></image>
						{{extraObj.title}}
					</view>
				</view>
			</view>

			<uni-fab ref="fab" :pattern="pattern" :content="content" :horizontal="horizontal" :vertical="vertical"
				:direction="direction" @trigger="trigger" />
			<post-list style="margin-top: 10rpx;" ref="postList"></post-list>
		</uni-transition>

	</view>
</template>

<script>
	import UniSection from '@/components/uni-section/components/uni-section/uni-section'
	import {
		cheerio
	} from '@/utils/cheerio.js'
	import {
		idArr
	} from '@/utils/yaohuo.js'
	export default {
		data() {
			return {
				idArr: idArr,
				actionArr: [{
					icon: 'compose',
					name: '发帖',
					url: 'https://yaohuo.me/wapindex.aspx?classid=206'
				}, {
					icon: 'medal',
					name: '游戏',
					url: 'https://yaohuo.me/games/gamesindex.aspx'
				}],
				loading: true,
				isSearch: false,
				searchContent: '',
				newArr: [],
				recommedArr: [],
				extraObj: {},
				directionStr: '垂直',
				horizontal: 'right',
				vertical: 'bottom',
				direction: 'vertical',
				pattern: {
					color: '#7A7E83',
					backgroundColor: '#fff',
					selectedColor: '#07c160',
					buttonColor: '#07c160',
					iconColor: '#fff'
				},
				content: [{
						iconPath: '/static/mine.png',
						selectedIconPath: '/static/mine.png',
						text: '我的'
					},
					{
						iconPath: '/static/message.png',
						selectedIconPath: '/static/message.png',
						text: '消息'
					}, {
						iconPath: '/static/refresh.png',
						selectedIconPath: '/static/refresh.png',
						text: '刷新'
					}
				]
			}
		},
		onReachBottom() {
			this.$refs.postList.loadMore()
		},
		onPullDownRefresh() {
			this.fetchData()
			this.$refs.postList.refreshData()
		},
		onLoad() {
			if (uni.getStorageSync('cookie')) {
				this.fetchData()
			}
		},
		onShow() {
			if (!uni.getStorageSync('cookie')) {
				uni.redirectTo({
					url: '/pages/login/login'
				})
			}
		},
		methods: {
			goToDetail(id) {
				uni.navigateTo({
					url: `/pages/detail/detail?id=${id}`
				})
			},
			goToWebview(e) {
				uni.navigateTo({
					url: `/pages/webview/webview?url=${this.actionArr[e.detail.index].url}`
				})
			},
			goToRecommend(item) {
				if (item.url.indexOf('/bbs/') > -1) {
					let id = item.url.split('-')[1].split('.')[0]
					uni.navigateTo({
						url: `/pages/detail/detail?id=${id}`
					})
				} else {
					uni.navigateTo({
						url: `/pages/webview/webview?url=${item.url}`
					})
				}
			},
			fetchData() {
				uni.request({
					url: 'https://yaohuo.me/',
					header: {
						cookie: uni.getStorageSync('cookie')
					},
					success: (res) => {
						let messageCountMatch = res.data.match(/收到(.*?)封飞鸽传书/)
						if (messageCountMatch) {
							uni.setNavigationBarTitle({
								title: `妖火网（${messageCountMatch[1]}条新消息）`
							})
						} else {
							uni.setNavigationBarTitle({
								title: '妖火网'
							})
						}
						const $ = cheerio.load(res.data)
						const children = $('.list')[0].children
						let newArr = []
						children.forEach((child, index) => {
							if (child.type === 'tag' && child.name === 'a' && newArr.length < 8) {
								newArr.push({
									id: child.attribs.href.split('-')[1].split('.')[0],
									title: child.children[0].data
								})
							}
						})
						this.newArr = newArr
						const recommedC = $('.shouye')
						let recommedArr = []
						let first = recommedC[0]
						recommedC.each(index => {
							let first = recommedC[index].children[0]
							let obj = {}
							while (first) {
								if (first.name === 'a') {
									obj.title = first.children[0].data
									obj.url = first.attribs.href[0] == '/' ?
										`https://yaohuo.me${first.attribs.href}` : first.attribs.href
								}
								if (first.name === 'img') {
									obj.img = first.attribs.src[0] == '/' ?
										`https://yaohuo.me${first.attribs.src}` : first.attribs.src
								}
								first = first.next
							}
							recommedArr.push(obj)
						})
						this.recommedArr = recommedArr
						const recommendExtra = $('.content')
						if (recommendExtra.length) {
							let extraObj = {}
							extraObj.title = ''
							let first = recommendExtra[0]
							first.children.forEach(child => {
								if (child.type == 'tag' && child.name == 'img') {
									extraObj.img = child.attribs.src[0] == '/' ?
										`https://yaohuo.me${child.attribs.src}` : child.attribs.src
								}
								if (child.type == 'tag' && child.name === 'a') {
									extraObj.title += child.children[0].data
									extraObj.url = child.attribs.href[0] == '/' ?
										`https://yaohuo.me${child.attribs.href}` : child.attribs.href
								}
								if (child.type === 'text') {
									extraObj.title += child.data
								}
							})
							this.extraObj = extraObj
						}
						this.loading = false
						uni.stopPullDownRefresh()
					}
				})
			},
			gridChange(e) {
				let url = `https://yaohuo.me/bbs/list.aspx?classid=${e.detail.index}`
				uni.navigateTo({
					url: `/pages/bbsList/bbsList?url=${encodeURIComponent(JSON.stringify({url}))}`
				})
			},

			search(e) {
				this.page = 1
				this.searchContent = e.value
				let url = `https://yaohuo.me/bbs/book_list.aspx?action=search&type=title&key=${this.searchContent}`
				uni.navigateTo({
					url: `/pages/bbsList/bbsList?url=${encodeURIComponent(JSON.stringify({url}))}`
				})
			},

			cancelSearch() {
				this.isSearch = false
				this.searchContent = ''
				this.page = 1
				this.fetchData()
			},
			async trigger(e) {
				if (e.index === 0) {
					uni.navigateTo({
						url: '/pages/message/message'
					})
				} else if (e.index === 1) {
					uni.navigateTo({
						url: '/pages/webview/webview?url=https://yaohuo.me/myfile.aspx'
					})
				} else {
					uni.pageScrollTo({
						scrollTop:0
					})
					uni.startPullDownRefresh({
						
					})
				}
			}
		}
	}
</script>
<style>
	page {
		background-color: #F3F3F3;
	}
</style>
<style lang="scss" scoped>
	.logo {
		text-align: center;
		margin: 20rpx 0 10rpx;

		image {
			width: 180px;
			height: 61px;
		}
	}

	.content {
		padding: 10rpx 20rpx 0;


		.recomment-card {
			background-color: #fff;
			position: relative;

		}

		.new-post-item {
			padding: 0 20rpx 20rpx;
			white-space: nowrap;
			text-overflow: ellipsis;
			overflow: hidden;
		}

		.img {
			width: 30rpx;
			height: 30rpx;
			margin-right: 10rpx;
			vertical-align: -3px;
		}


	}

	::v-deep .uni-section {
		border-radius: 8px;
	}

	.grid {
		background-color: #fff;
		border-radius: 8px;

		.image {
			width: 25px;
			height: 25px;
		}

		.text {
			font-size: 14px;
			margin-top: 5px;
		}

		.example-body {
			/* #ifndef APP-NVUE */
			// display: block;
			/* #endif */
		}

		.grid-dynamic-box {
			margin-bottom: 15px;
		}

		.grid-item-box {
			flex: 1;
			// position: relative;
			/* #ifndef APP-NVUE */
			display: flex;
			/* #endif */
			flex-direction: column;
			align-items: center;
			justify-content: center;
			padding: 15px 0;
		}

		.grid-item-box-row {
			flex: 1;
			// position: relative;
			/* #ifndef APP-NVUE */
			display: flex;
			/* #endif */
			flex-direction: row;
			align-items: center;
			justify-content: center;
			padding: 15px 0;
		}

		.grid-dot {
			position: absolute;
			top: 5px;
			right: 15px;
		}

		.swiper {
			height: 420px;
		}

		/* #ifdef H5 */
		@media screen and (min-width: 768px) and (max-width: 1425px) {
			.swiper {
				height: 630px;
			}
		}

		@media screen and (min-width: 1425px) {
			.swiper {
				height: 830px;
			}
		}

		/* #endif */
	}
</style>
