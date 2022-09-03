<template>
	<view>
		<view class="tips color_fff size_12 align_c" :class="{ 'show':ajax.loading }">
			{{ajax.loadText}}
		</view>
		<view class="box-1" id="list-box">
			<view class="talk-list">
				<view v-for="(item,index) in talkList" :key="index" :id="`msg-${item.id}`">
					<view class="item flex_col" :class=" item.type == 1 ? 'push':'pull' ">
						<view class="content">
							<mp-html :content="item.content" selectable lazy-load domain="https://yaohuo.me"
								containerStyle="line-height:40rpx;word-break: break-all;font-size:30rpx"></mp-html>
						</view>
					</view>
				</view>
				<view id="bottom"></view>
			</view>
		</view>
		<view class="box-2">
			<view class="flex_col">
				<view class="flex_grow">
					<input type="text" class="content" v-model="replyData.content" placeholder="请输入聊天内容"
						placeholder-style="color:#DDD;" :cursor-spacing="6">
				</view>
				<button class="send" @tap="send">发送</button>
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
				talkList: [],
				ajax: {
					loading: true, // 加载中
					loadText: '正在获取消息'
				},
				replyData: {
					content: '',
					action: 'gomod',
					classid: 0,
					siteid: 1000,
					types: 0,
					issystem: '',
					toid: '',
					backurl: 'wapindex.aspx?siteid=1000&classid=0',
					title: '',
					touseridlist: '',
					sid: uni.getStorageSync('cookie').split('=')[1],
					g: ' 发 送 '
				}
			}
		},
		onLoad(option) {
			this.replyData.toid = option.id
			this.ajax.loading = true;
			this.ajax.loadText = '正在获取消息';
			uni.request({
				url: `https://yaohuo.me/bbs/messagelist_view.aspx?id=${option.id}`,
				header: {
					'cookie': uni.getStorageSync('cookie')
				},
				success: (res) => {
					let $ = cheerio.load(res.data)
					let inputs = $('input')
					inputs.each(i => {
						if (inputs[i].attribs.name === 'touseridlist') {
							this.replyData.touseridlist = inputs[i].attribs.value
							uni.setNavigationBarTitle({
								title: `${option.user}(${inputs[i].attribs.value})`
							})
						}
					})
					let messages = $('.listmms')
					let messageArr = []
					messages.each(index => {
						let message = messages[index]
						let type = 0
						if (message.attribs.class.indexOf('the_me') > -1) {
							type = 1
						}
						let first = message.children[0]
						let content = ''

						function getText(children) {
							if (!children || !children.children) {
								return
							}
							children.children.forEach(child => {
								if (child.type && child.type == 'tag' && child.name === 'br') {
									content += '<br>'
								}
								if (child.data) {
									if (content.indexOf(child.data) < 0) {
										content += child.data
									}
								}
								if (child.type && child.name === 'img') {
									content += `<img src="${child.attribs.src}"></img>`
								}
								// console.log(child);
								if (child.type && child.name === 'a') {
									if (child.attribs.href.indexOf('bbs/book_view.aspx') > -
										1) {
										let id = child.attribs.href.match(/\Wid=(\d{0,20})/)[1]
										content +=
											`<a href="/pages/detail/detail?id=${id}">${child.children[0].data}</a>`
									} else {
										content +=
											`<a href="${child.attribs.href}">${child.children[0].data}</a>`
									}

								}
								getText(child)
							})
						}
						getText(first)
						messageArr.unshift({
							id: index,
							type,
							content
						})
					})
					this.talkList = messageArr
					this.ajax.loadText = '消息获取成功';
					setTimeout(() => {
						this.ajax.loading = false;
						this.$nextTick(() => {
							this.setPageScrollTo('#bottom')
						})
					}, 500);
				}
			})
		},
		methods: {
			setPageScrollTo(selector) {
				let view = uni.createSelectorQuery().in(this).select(selector);
				view.boundingClientRect((res) => {
					uni.pageScrollTo({
						scrollTop: res.top,
						duration: 300
					});
				}).exec();
			},
			// 发送信息
			send() {
				if (!this.replyData.content) {
					return uni.showToast({
						title: '请输入有效的内容',
						icon: 'none'
					})
				}
				uni.showLoading({
					title: '正在发送'
				})
				uni.request({
					method: 'POST',
					url: 'https://yaohuo.me/bbs/messagelist_add.aspx',
					header: {
						'Content-Type': 'application/x-www-form-urlencoded',
						cookie: uni.getStorageSync('cookie')
					},
					data: this.replyData,
					success: (res) => {
						uni.hideLoading();
						let data = {
							"id": new Date().getTime(),
							"content": this.replyData.content,
							"type": 1
						}
						this.talkList.push(data);
						this.$nextTick(() => {
							this.replyData.content = '';
							uni.pageScrollTo({
								scrollTop: 999999,
								duration: 300
							});
						})
					}
				})
			}
		}
	}
</script>

<style lang="scss">
	@import "../../lib/global.scss";

	page {
		background-color: #F3F3F3;
		font-size: 28rpx;
	}

	/* 加载数据提示 */
	.tips {
		position: fixed;
		left: 0;
		top: var(--window-top);
		width: 100%;
		z-index: 9;
		background-color: rgba(0, 0, 0, 0.15);
		height: 72rpx;
		line-height: 72rpx;
		transform: translateY(-80rpx);
		transition: transform 0.3s ease-in-out 0s;

		&.show {
			transform: translateY(0);
		}
	}

	.box-1 {
		width: 100%;
		height: auto;
		padding-bottom: 100rpx;
		box-sizing: content-box;

		/* 兼容iPhoneX */
		margin-bottom: 0;
		margin-bottom: constant(safe-area-inset-bottom);
		margin-bottom: env(safe-area-inset-bottom);
	}

	.box-2 {
		position: fixed;
		left: 0;
		width: 100%;
		bottom: 0;
		height: auto;
		z-index: 2;
		border-top: #e5e5e5 solid 1px;
		box-sizing: content-box;
		background-color: #F3F3F3;

		/* 兼容iPhoneX */
		padding-bottom: 0;
		padding-bottom: constant(safe-area-inset-bottom);
		padding-bottom: env(safe-area-inset-bottom);

		>view {
			padding: 0 20rpx;
			height: 100rpx;
		}

		.content {
			background-color: #fff;
			height: 64rpx;
			padding: 0 20rpx;
			border-radius: 32rpx;
			font-size: 28rpx;
		}

		.send {
			background-color: #42b983;
			color: #fff;
			height: 64rpx;
			margin-left: 20rpx;
			border-radius: 32rpx;
			padding: 0;
			width: 120rpx;
			line-height: 62rpx;

			&:active {
				background-color: #5fc496;
			}
		}
	}

	.talk-list {
		padding-bottom: 20rpx;

		/* 消息项，基础类 */
		.item {
			padding: 20rpx 20rpx 0 20rpx;
			align-items: flex-start;
			align-content: flex-start;
			color: #333;

			.pic {
				width: 92rpx;
				height: 92rpx;
				border-radius: 50%;
				border: #fff solid 1px;
			}

			.content {
				padding: 20rpx;
				border-radius: 4px;
				max-width: 500rpx;
				word-break: break-all;
				line-height: 52rpx;
				position: relative;
			}

			/* 收到的消息 */
			&.pull {
				.content {
					margin-left: 32rpx;
					background-color: #fff;

					&::after {
						content: '';
						display: block;
						width: 0;
						height: 0;
						border-top: 16rpx solid transparent;
						border-bottom: 16rpx solid transparent;
						border-right: 20rpx solid #fff;
						position: absolute;
						top: 30rpx;
						left: -18rpx;
					}
				}
			}

			/* 发出的消息 */
			&.push {
				/* 主轴为水平方向，起点在右端。使不修改DOM结构，也能改变元素排列顺序 */
				flex-direction: row-reverse;

				.content {
					margin-right: 32rpx;
					background-color: #a0e959;

					&::after {
						content: '';
						display: block;
						width: 0;
						height: 0;
						border-top: 16rpx solid transparent;
						border-bottom: 16rpx solid transparent;
						border-left: 20rpx solid #a0e959;
						position: absolute;
						top: 30rpx;
						right: -18rpx;
					}
				}
			}
		}
	}
</style>
