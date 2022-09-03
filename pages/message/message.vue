<template>
	<view class="content">
		<view v-for="(item,index) in messages" :key="index" class="message" @click="goToDetail(index)"
			@longpress="deleteMessage(index)">
			<view class="f-16">
				<image v-if="item.unRead" src="https://yaohuo.me/NetImages/new.gif" style="width: 50rpx;height: 25rpx;">
				</image>{{item.content}}
			</view>
			<view class="info">
				<uni-row>
					<uni-col :span="12">
						<view class="f-13">
							{{item.from}}
						</view>
					</uni-col>
					<uni-col :span="12">
						<view class="text-right f-13">
							{{item.time}}
						</view>
					</uni-col>
				</uni-row>
			</view>
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
				messages: [],
				page: 1,
				totalPage: 0,
				canFresh:true
			}
		},
		onLoad() {
			this.fetchMessage()
		},
		onReachBottom() {
			if (this.page < this.totalPage) {
				this.page++
				this.fetchMessage()
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
			this.canFresh = false
			this.page = 1
			setTimeout(() => {
				this.canFresh = true
			}, 1000 * 20)
			this.fetchMessage()
		},
		methods: {
			deleteMessage(index) {
				uni.showModal({
					title: '操作提醒',
					content: '删除后无法恢复，确认删除吗？',
					success: (res) => {
						if (res.confirm) {
							uni.request({
								url: `https://yaohuo.me/bbs/messagelist_del.aspx?action=godel&id=${this.messages[index].id}`,
								header: {
									cookie: uni.getStorageSync('cookie')
								},
								success: (res) => {
									this.messages.splice(index, 1)
									uni.showToast({
										title: '删除成功',
										icon: 'success'
									})
								}
							})
						}
					}
				})
			},
			goToDetail(index) {
				uni.navigateTo({
					url: `/pages/chat/chat?id=${this.messages[index].id}&user=${this.messages[index].from}`
				})
			},
			fetchMessage() {
				uni.showLoading({
					title: '加载中'
				})
				uni.request({
					url: `https://yaohuo.me/bbs/messagelist.aspx?page=${this.page}`,
					header: {
						cookie: uni.getStorageSync('cookie')
					},
					success: (res) => {
						let $ = cheerio.load(res.data)
						let pageMatch = $('.showpage')[0].children[0].data.match(/\d\/(\d{0,10}) 页/)
						this.totalPage = pageMatch ? pageMatch[1] : 1
						let messages = $('.listmms')
						let messageArr = []
						messages.each(index => {
							let message = messages[index]
							let msgObj = {}
							msgObj.content = ''
							msgObj.unRead = false
							message.children.forEach((child, index) => {
								if (child.data && index !== 0 && child.data !== ']') {
									if (child.data.indexOf('来自') > -1) {
										msgObj.from = child.data.replace('来自', '')
									} else if (child.data[child.data.length - 1] === '[') {
										msgObj.time = child.data.replace('[', '')
									} else {
										msgObj.content += child.data
									}
								}
								if (child.type && child.name === 'img') {
									if (child.attribs.src.indexOf('new.gif') > -1) {
										msgObj.unRead = true
									}
								}
								if (child.type && child.name === 'a') {
									let id = child.attribs.href.match(/&id=(.*?)&/)
									if (child.next.data !== ']') {
										msgObj.content += child.children[0].data
									}
									msgObj.id = id[1]
									// if (child.attribs.href.indexOf('messagelist_del.aspx') > -
									// 	1) {
									// 	// content +=
									// 	// 	`<a href="https://yaohuo.me/bbs/messagelist_del.aspx?action=del&id=${id[1]}">${child.children[0].data}</a>`
									// } else {
									// 	// content +=
									// 	// 	`<a href="/pages/chat/chat?id=${id[1]}">${child.children[0].data}</a>`
									// }
								}
							})
							messageArr.push(msgObj)
						})
						if (this.page === 1) {
							this.messages = messageArr
						} else {
							this.messages = this.messages.concat(messageArr)
						}
						uni.hideLoading()
						uni.stopPullDownRefresh()
					}
				})
			}
		}
	}
</script>
<style>
	page {
		background-color: #fff;
	}
</style>
<style scoped>
	/* 	.content {
		padding: 10rpx 20rpx;
	} */

	.message {
		border-bottom: 1px solid #F3F3F3;
		padding: 30rpx;
	}

	.info {
		margin-top: 20rpx;
		color: rgba(0, 0, 0, .3);
	}
</style>
