<template>
	<view v-if="nodes.length">
		<uni-transition mode-class="fade" :duration="300" :show="true">
			<view class="content">
				<view class="f-18 title">
					{{info.title}}
				</view>
				<view v-if="info.extra" class="f-14" style="margin-top: 20rpx;">
					{{info.extra}}
				</view>
				<view style="margin-top: 20rpx;">
					<uni-row>
						<uni-col :span="18">
							<view class="f-14">
								{{info.author}}
							</view>
						</uni-col>
						<uni-col :span="6">
							<view class="f-14 text-right">
								{{idObj[info.classId]}}
							</view>
						</uni-col>
					</uni-row>
				</view>
				<view style="border-bottom:  1px dashed #dcdcdc;margin-bottom: 20rpx;margin-top: 15rpx;">

				</view>
				<mp-html :content="nodes" selectable lazy-load domain="https://yaohuo.me"
					containerStyle="line-height:60rpx;word-break: break-all;font-size:32rpx" @linktap="linkTap">
				</mp-html>
				<view style="margin: 20rpx 0;">
					<uni-row>
						<uni-col :span="12">
							<view class="f-13 info">
								浏览{{info.readCount}}次
							</view>
						</uni-col>
						<uni-col :span="12">
							<view class="f-13 text-right info">
								{{info.time}}
							</view>
						</uni-col>
					</uni-row>
				</view>
			</view>
			<view class="tip" v-if="info.status">
				<text class="tip-text">{{info.status}}</text>
			</view>
			<comment id="comment" :comments="comments" :postInfo="info" @fetchReply="fetchReply" :postId="info.postId">
			</comment>
		</uni-transition>
	</view>
</template>

<script>
	import {
		cheerio
	} from '@/utils/cheerio.js'
	import {
		idObj
	} from '@/utils/yaohuo.js'
	import HTMLParser from '@/utils/html-parser.js'
	export default {
		data() {
			return {
				id: '',
				nodes: '',
				comments: [],
				info: {},
				page: 1,
				totalPage: 0,
				status: 'more',
				idObj: idObj
			}
		},
		onLoad(option) {
			this.info.postId = option.id
			this.fetchDetail()
		},
		onPullDownRefresh() {
			this.fetchDetail()
		},
		onReachBottom() {
			if (this.page < this.totalPage) {
				uni.showNavigationBarLoading()
				this.status = 'loading'
				this.page++
				this.fetchReply()
			}
		},
		methods: {
			linkTap(e) {
				if (e.href.indexOf('bbs/picDIY.aspx') > -1) {
					let imgUrl = 'https://yaohuo.me/' + e.href.split('path=')[1]
					uni.previewImage({
						current: imgUrl,
						urls: [imgUrl]
					})
				}
			},
			fetchReply(flag) {
				if (flag) {
					this.comments = []
				}
				uni.request({
					url: `https://yaohuo.me/bbs/book_re.aspx?id=${this.info.postId}&classid=${this.info.classId}&page=${this.page}`,
					header: {
						cookie: uni.getStorageSync('cookie')
					},
					success: (res) => {
						let $ = cheerio.load(res.data)
						let replies = $('.list-reply')
						let reply = replies[0]
						this.getReply(reply)
						if (this.page === this.totalPage) {
							this.status = 'noMore'
						} else {
							this.status = 'more'
						}
					},
					complete: () => {
						uni.hideNavigationBarLoading()
					}
				})
			},
			fetchDetail() {
				uni.showLoading({
					title: '拼命加载中'
				})
				uni.request({
					url: `https://yaohuo.me/bbs-${this.info.postId}.html`,
					header: {
						cookie: uni.getStorageSync('cookie')
					},
					success: (res) => {
						let tip = res.data.match(/<div class=\"tip\">(.*?)<\/div>/)
						this.info.isEnd = false
						if (tip) {
							if (tip[1].indexOf('结束原因') > -1) {
								this.info.status = '已结帖'
								this.info.isEnd = true
							}
							if (tip[1].indexOf('锁定原因') > -1) {
								this.info.status = '已锁定'
							}
							if (tip[1].indexOf('审核') > -1) {
								setTimeout(() => {
									uni.navigateBack({
										delta: 1
									})
								}, 1000)
								return uni.showToast({
									title: '帖子已被删除',
									icon: 'error'
								})
							}
						}
						let replyCountMatch = res.data.match(/更多回帖\((.*?)\)/)
						this.info.replyCount = replyCountMatch ? replyCountMatch[1] : 0
						this.totalPage = Math.ceil(this.info.replyCount / 15)
						let classIdMatch = res.data.match(/&classid=(.*?)"/)
						this.info.classId = classIdMatch ? classIdMatch[1] : ''
						let content = res.data.match(/<!--listS-->([\s\S]*?)<!--listE-->/)
						this.nodes = content ? content[1].replace(
							/height=\"100%px\"/, '').replace(/width=\"100%px\"/, 'width="100%"').replace(
							/poster=\"(.*?)\"/, '').replace(/<a href=\"\/bbs\/picDIY.aspx.*?\""/,
							'<a href=\"\"') : ' '
						let fileList = res.data.match(/<div class=\"line\">([\s\S]*?)<\/div>/g)
						if (fileList) {
							fileList.forEach(r => {
								this.nodes += r.replace(/\/bbs\/download.aspx/,
									'https://yaohuo.me/bbs/download.aspx')
							})
						}
						let $ = cheerio.load(res.data)
						this.getPostInfo($)
						this.fetchReply()
						uni.hideLoading()
						uni.stopPullDownRefresh()
					}
				})
			},
			getReply(reply) {
				if (!reply || reply.data == 'listE') {
					return
				}
				let replyObj = {}
				let first = reply.children[0]
				replyObj.floor = first.data.match(/\[(.*?)\]/)[1]
				if (replyObj.floor == '沙发') {
					replyObj.floor = '1'
				} else if (replyObj.floor == '椅子') {
					replyObj.floor = '2'
				} else if (replyObj.floor == '板凳') {
					replyObj.floor = '3'
				} else {
					replyObj.floor = replyObj.floor.replace('楼', '')
				}
				replyObj.text = ''
				first = first.next
				while (first) {
					if (first.next && !first.next.next) { // 昵称
						let firstChild = first.children[0]
						replyObj.user = ''
						while (firstChild) {
							if (firstChild.type && firstChild.type === 'text') {
								if (firstChild.data !== ' ') {
									replyObj.user += firstChild.data
								}
							}
							if (firstChild.type && first.type === 'tag' && firstChild.name == 'font') {
								replyObj.user += firstChild.children[0].data
							}
							firstChild = firstChild.next
						}
					}
					if (!first.next) { // 时间
						replyObj.time = first.data.trim()
					}
					if (first.type && first.type === 'tag' && first.name === 'b') {
						replyObj.text += `[<b>${first.children[0].data}</b>]`
					}
					if (first.type && first.type === 'text' && first.data.indexOf(']') === 0) {
						replyObj.text += first.data.replace(']', '')
						first = first.next
						while (first) {
							if (first.type && first.type === 'text') {
								replyObj.text += first.data
							}
							if (first.type && first.type === 'tag' && first.name === 'br') {
								replyObj.text += '<br>'
							}
							if (first.type && first.name === 'img') {
								if (first.attribs.src.indexOf('face/') > -1) {
									replyObj.text += `<img src="${first.attribs.src}">`
								} else {
									replyObj.text += `<img style="max-width:80%" src="${first.attribs.src}">`
								}

							}
							if (first.name === 'a' && first.attribs.href.indexOf('userinfo') < 0 && first.attribs.href
								.indexOf('bbs/Book_re') < 0) {
								uni.request({
									url: `https://yaohuo.me${first.attribs.href}`,
									header: {
										cookie: uni.getStorageSync('cookie')
									},
									success: (res) => {
										let imgUrl = res.data.match(/img src=\"(.*?)\"/)
										if (imgUrl) {
											replyObj.text +=
												`<img style="max-width:80%" src="https://yaohuo.me${imgUrl}">`
										} else {
											let fileUrl = res.data.match(/\"(\/bbs\/download.*?)\"/)
											if (fileUrl) {
												replyObj.text +=
													`<a href="https://yaohuo.me${fileUrl[1]}">点击复制附件链接</a>`
											}
										}

									}
								})
							}
							if (first.next.name === 'a' && first.next.attribs.href.indexOf('userinfo') > -1) {
								break
							}
							first = first.next
						}
					}
					if (first.type && first.name === 'img') {
						replyObj.text += `<img style="width:80%" src="${first.attribs.src}">`
					}
					first = first.next
				}
				replyObj.text = replyObj.text.replace('{}', '').replace('[]', '')
				this.comments.push(replyObj)
				reply = reply.next
				this.getReply(reply)
			},
			getPostInfo($) {
				// this.info.postId = this.url.split('-')[1].split('.')[0]
				let data = $('.content')[0]
				let children = data.children
				if (children[0].data.indexOf('标题') > -1) {
					this.info.title = children[0].data.match(/](.*?)\(阅/)[1].trim()
					this.info.readCount = children[0].data.match(/\(阅(.*?)\)/)[1].trim()
					this.info.time = children[2].data.match(/\](.*)/)[1].trim()
				} else if (children[0].data.indexOf('悬赏') > -1) {
					this.info.title = children[2].data.match(/](.*?)\(阅/)[1].trim()
					this.info.readCount = children[2].data.match(/\(阅(.*?)\)/)[1].trim()
					this.info.time = children[4].data.match(/\](.*)/)[1].trim()
					this.info.extra = children[0].data
				} else if (children[0].data.indexOf('礼金') > -1) {
					this.info.title = children[4].data.match(/](.*?)\(阅/)[1].trim()
					this.info.readCount = children[4].data.match(/\(阅(.*?)\)/)[1].trim()
					this.info.time = children[6].data.match(/\](.*)/)[1].trim()
					this.info.extra = children[0].data
				}
				let authorData = $('.subtitle')[1]
				let authorChildren = authorData.children[1].children
				if (authorChildren.length === 1) {
					this.info.author = authorChildren[0].data
				} else {
					let first = authorChildren[0]
					this.info.author = ''
					while (first) {
						if (first.type === 'text' && first.data !== ' ') {
							this.info.author += first.data
						}
						if (first.type === 'tag' && first.name === 'font') {
							this.info.author += first.children[0].data
						}
						first = first.next
					}
				}
			}
		}
	}
</script>
<style>
	page {
		background: #fff;
	}
</style>
<style scoped>
	.content {
		padding: 20rpx 20rpx;
	}

	.title {
		color: 999;
	}

	.info {
		color: rgba(0, 0, 0, .3);
	}

	.tip {
		text-align: center;
		margin-bottom: 20rpx;
	}

	.tip-text {
		background: rgba(247, 247, 247);
		font-size: 28upx;
		line-height: 28upx;
		padding: 12upx 30upx;
		border-radius: 30upx;
		color: #333 !important;
		display: inline-block;
	}
</style>
