<template>
	<view>
		<uni-search-bar bgColor="#fff" class="uni-mt-10" radius="8" placeholder="善用搜索" clearButton="auto"
			@confirm="search" @cancel="cancelSearch" />
		<view class="content">

			<view v-for="(post,index) in posts" :key="index" class="post-card" @click="goToDetail(post.url)">
				<view class="title f-16">
					{{index+1}}.{{post.title}}
				</view>
				<view class="info">
					<uni-row>
						<uni-col :span="12">
							<view class="f-13">
								{{post.author}}
							</view>
						</uni-col>
						<uni-col :span="12">
							<view class="f-13 text-right">
								<uni-icons type="eye" class="icon"></uni-icons>{{post.readCount}}
								<uni-icons type="chatboxes" class="icon"></uni-icons>{{post.replyCount}}
							</view>
						</uni-col>
					</uni-row>
				</view>
				<view class="tags">
					<uni-tag class="tag" v-for="(tag,tagIndex) in post.tags" :key="tagIndex" :circle="true"
						:type="typeObj[tag]" :text="tag"></uni-tag><strong></strong>
				</view>
			</view>
			<uni-load-more style="height: 50rpx;" v-show="page!==1" :status="status"></uni-load-more>
			<uni-fab ref="fab" :pattern="pattern" :content="content" :horizontal="horizontal" :vertical="vertical"
				:direction="direction" @trigger="trigger" />
		</view>
	</view>
</template>

<script>
	import {
		cheerio
	} from '@/utils/cheerio.js'

	export default {
		data() {
			return {
				posts: [],
				typeObj: {
					'附': 'success',
					'赏': 'warning',
					'肉': 'warning',
					'结': 'default'
				},
				isSearch: false,
				searchContent: '',
				page: 1,
				totalPage: 135,
				status: 'more',
				canFresh: true,
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
					}
				]
			}
		},
		onReachBottom() {
			if (this.page < this.totalPage) {
				this.page++
				this.status = 'loading'
				if (!this.isSearch) {
					this.fetchData()
				} else {
					this.searchNext()
				}
			}
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
		onPullDownRefresh() {
			if (!this.canFresh) {
				uni.showToast({
					title: '请勿频繁刷新',
					icon: 'error'
				})
				uni.stopPullDownRefresh()
				return
			}
			this.isSearch = false
			this.canFresh = false
			this.page = 1
			setTimeout(() => {
				this.canFresh = true
			}, 1000 * 20)
			this.fetchData()
		},
		methods: {
			searchNext() {
				this.fetchSearchData()
			},
			search(e) {
				this.page = 1
				this.searchContent = e.value
				this.fetchSearchData()
			},
			fetchSearchData() {
				uni.showLoading({
					title: '搜索中'
				})
				uni.request({
					url: `https://yaohuo.me/bbs/book_list.aspx?action=search&type=title&key=${this.searchContent}&page=${this.page}`,
					header: {
						cookie: uni.getStorageSync('cookie')
					},
					success: (res) => {
						this.handleData(res.data)
						this.isSearch = true
					}
				})
			},
			cancelSearch() {
				this.isSearch = false
				this.searchContent = ''
				this.page = 1
				this.fetchData()
			},
			trigger(e) {
				if (e.index) {
					uni.navigateTo({
						url: '/pages/message/message'
					})
				} else {
					uni.navigateTo({
						url: '/pages/webview/webview?url=https://yaohuo.me/myfile.aspx'
					})
				}
			},
			fetchData() {
				if (this.canFresh && this.status != 'loading') {
					uni.showLoading({
						title: '拉取数据中'
					})
				}
				uni.request({
					url: `https://yaohuo.me/bbs/book_list.aspx?action=new&siteid=1000&classid=0&getTotal=2022&page=${this.page}`,
					header: {
						cookie: uni.getStorageSync('cookie')
					},
					success: (res) => {
						let messageCountMatch = res.data.match(/收到(.*?)封飞鸽传书/)
						if (messageCountMatch) {
							uni.setNavigationBarTitle({
								title: `妖火网（${messageCountMatch[1]}条新消息）`
							})
						}else{
							uni.setNavigationBarTitle({
								title: '妖火网'
							})
						}
						let tip = res.data.match(/<div class=\"tip\">(.*?)<\/div>/)
						if (tip) {
							if (tip[1].indexOf('失效') > -1) {
								setTimeout(() => {
									uni.redirectTo({
										url: '/pages/login/login'
									})
								}, 500)
								return uni.showToast({
									title: '请重新登录',
									icon: 'error'
								})
							}
						}
						this.handleData(res.data)
					}
				})
			},
			handleData(resData) {
				const $ = cheerio.load(resData)
				let data = $('.listdata')
				let posts = []
				for (let index in data) {
					let post = {}
					let tags = []
					if (data[index].type === 'tag') {
						let children = data[index].children
						let child = children[1]
						while (child && child.name === 'img') {
							tags.push(child.attribs.alt === '礼' ? '肉' : child.attribs.alt)
							if (child.next.name === 'img') {
								child = child.next
							} else {
								break
							}
						}
						post.tags = tags
						while (child) {
							if (child.children && child.children.length === 2) {
								break
							}
							if (child.name && child.name == 'a' && isNaN(child.children[0].data)) {
								post.title = child.children[0].data
								post.url = child.attribs.href
							}
							if (child.name && child.name == 'a' && !isNaN(child.children[0].data)) {
								post.replyCount = child.children[0].data
								post.readCount = child.next.data.replace(/[^\d.]/g, '')
							}
							if (child.type === 'text' && child.next.name && child.next.name === 'a') {
								if (child.data === '/') {
									post.author = child.prev.children[0].data
								} else {
									post.author = child.data.replace('/', '')
								}
							}
							child = child.next
						}
						posts.push(post)
					}
				}
				if (this.page === 1) {
					this.posts = posts
				} else {
					this.posts = this.posts.concat(posts)
				}
				if (this.page < this.totalPage) {
					this.status = 'more'
				} else {
					this.status = 'noMore'
				}
				uni.hideLoading()
				uni.stopPullDownRefresh()
			},
			goToDetail(url) {
				if (uni.getStorageSync('cookie')) {
					let id = url.split('-')[1].split('.')[0]
					uni.navigateTo({
						url: `/pages/detail/detail?id=${id}`
					})
				} else {
					uni.showModal({
						content: '请先登录',
						success: (res) => {
							if (res.confirm) {
								uni.navigateTo({
									url: '/pages/login/login'
								})
							}
						}
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
	.content {
		padding: 10rpx 20rpx 0;

		.post-card {
			padding: 20rpx 40rpx;
			box-shadow: rgba(0, 0, 0, .2) 0 1px 5px 0px;
			margin-bottom: 20rpx;
			border-radius: 8px;
			position: relative;
			background: #fff;

			.tags {
				position: absolute;
				top: -10rpx;
				right: -10rpx;

				.tag {
					margin-left: 10rpx;
				}
			}

			.title {}

			.info {
				margin-top: 20rpx;

				.icon {
					margin: 0 10rpx;
					vertical-align: -1px;
				}
			}


		}
	}
</style>
