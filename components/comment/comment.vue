<template>
	<view class="content">
		<view class="uni-padding-wrap">
			<!-- 评论区 start -->
			<view class="f-16">全部评论（共{{postInfo.replyCount}}条）</view>
			<view class="uni-comment">
				<view class="uni-comment-list" v-for="(comment,index) in comments" :key="index">
					<view class="uni-comment-face">
						<view class="floor">
							{{comment.floor}}
						</view>
					</view>
					<view class="uni-comment-body">
						<view class="uni-comment-top">
							<text @click="goToUserArea(index)">{{comment.user}}</text>
						</view>
						<view class="uni-comment-date">
							<text>{{comment.time}}</text>
							<view class="uni-comment-reply-btn" v-if="!postInfo.isEnd" @click="replyToFloor(index)">回复
							</view>
						</view>
						<mp-html :content="comment.text" selectable lazy-load domain="https://yaohuo.me"
							containerStyle="line-height:40rpx;word-break: break-all;font-size:30rpx"></mp-html>
					</view>
				</view>
			</view>
		</view>
		<view style="position: fixed;bottom: 0;left: 0;right: 0;background-color: #f7f7f7;">
			<view class="" style="height: 80rpx;padding: 10rpx 20rpx;">
				<uni-row>
					<uni-col :span="18">
						<view style="display: inline-block;margin-right: 20rpx;">
							<picker @change="bindPickerChange" :value="faceIndex" :range="array" range-key="name">
								<image src="../../static/smile.png" style="width: 70rpx;height: 70rpx;"></image>
							</picker>
						</view>
						<view style="display: inline-block;margin-right: 25rpx;">
							<image src="../../static/picture.png" style="width: 70rpx;height: 70rpx;"></image>
						</view>
						<view style="display: inline-block;margin-right: 30rpx;">
							<image src="../../static/video.png" style="width: 60rpx;height: 60rpx;margin-bottom: 5rpx;">
							</image>
						</view>
						<view style="display: inline-block;margin-right: 20rpx;">
							<image src="../../static/music.png" style="width: 60rpx;height: 60rpx;margin-bottom: 5rpx;">
							</image>
						</view>
					</uni-col>
					<uni-col :span="6">
						<view class="text-right" v-show="replyData.content">
							<button :loading="loading" :disabled="loading" class="submit-btn" type="primary" size="mini"
								@click="reply">发表</button>
						</view>
					</uni-col>
				</uni-row>
			</view>
			<view class="" style="height: 120rpx;padding: 15rpx;">
				<view style="background-color: #fff;">
					<input style="height: 90rpx;padding: 0 20rpx;border-radius: 32rpx;" cursor-spacing="20rpx"
						:adjust-position='true' type="text" :focus="isReplyFloor" @blur="cancelReply"
						v-model="replyData.content" :placeholder="replyTips" />
				</view>
				</uni-row>
			</view>
		</view>
	</view>
</template>

<script>
	import md5 from 'js-md5'
	import faces from '@/utils/faces.js'
	export default {
		name: 'comment',
		props: {
			comments: {
				type: Array,
				default: () => {
					return []
				}
			},
			postInfo: {
				type: Object,
				default: () => {
					return {}
				}
			}
		},
		data() {
			return {
				replyTips: '请勿乱打字回复,以免被加黑。',
				array: faces,
				faceIndex: 0,
				md5: md5,
				loading: false,
				isReplyFloor: false,
				replyModalShow: false,
				replyData: {
					face: '',
					sendmsg: 0,
					content: '',
					action: 'add',
					id: this.postInfo.postId,
					siteid: 1000,
					lpage: 1,
					classid: this.postInfo.classId,
					sid: uni.getStorageSync('cookie').split('=')[1],
					g: '快速回复'
				},
				originReplyData: {}
			}
		},
		mounted() {
			this.originReplyData = JSON.parse(JSON.stringify(this.replyData))
		},
		methods: {
			bindPickerChange: function(e) {
				this.faceIndex = e.detail.value;
				if (this.faceIndex) {
					this.replyData.face = this.array[this.faceIndex].face
				}
			},
			cancelReply() {
				if (!this.loading) {
					this.isReplyFloor = false
					delete this.replyData.reply
					delete this.replyData.touserid
					this.replyData.g = '快速回复'
					this.replyTips = '请勿乱打字回复,以免被加黑。'
				}
			},
			replyToFloor(index) {
				let floor = this.comments[index]
				this.replyData.g = '发表回复'
				this.replyData.reply = floor.floor
				this.replyData.touserid = floor.user.match(/\((.*?)\)/)[1]
				this.isReplyFloor = true
				this.replyTips = `回复${floor.floor}楼：`
			},
			goToUserArea(index){
				let user = this.comments[index].user
				let id = user.match(/\((\d{0,10})\)/)
				uni.navigateTo({
					url:`/pages/webview/webview?url=https://yaohuo.me/bbs/userinfo.aspx?touserid=${id[1]}`
				})
			},
			reply() {
				if (!this.replyData.content) {
					return uni.showToast({
						title: '评论不得为空',
						icon: 'error'
					})
				}
				this.loading = true
				uni.request({
					url: 'https://yaohuo.me/bbs/book_re.aspx',
					method: 'POST',
					header: {
						'Content-Type': 'application/x-www-form-urlencoded',
						cookie: uni.getStorageSync('cookie')
					},
					data: this.replyData,
					success: (res) => {
						uni.showToast({
							title: '评论成功',
							icon: 'success'
						})
						this.cancelReply()
						// this.$emit('fetchReply', 1)
					},
					fail: (err) => {
						uni.showToast({
							title: '评论失败',
							icon: 'error'
						})
					},
					complete: () => {
						this.loading = false
						this.replyData.content = ''
					}
				})
			}
		}
	}
</script>

<style scoped>
	/* comment */
	page {
		background-color: #fff;
	}

	.uni-padding-wrap {
		padding: 0 20rpx 200rpx;
	}


	.uni-comment {
		padding: 5rpx 0;
		display: flex;
		flex-grow: 1;
		flex-direction: column;
	}

	.uni-comment-list {
		flex-wrap: nowrap;
		padding: 10rpx 0;
		margin: 10rpx 0;
		width: 100%;
		display: flex;
	}

	.uni-comment-face {
		width: 70rpx;
		height: 70rpx;
		border-radius: 50%;
		margin-right: 15rpx;
		background-color: #1E90FF;
	}

	.uni-comment-face image {
		width: 100%;
		border-radius: 100%;
	}

	.uni-comment-body {
		width: 100%;
	}

	.uni-comment-top {
		line-height: 1.5em;
		justify-content: space-between;
	}

	.uni-comment-top text {
		color: #0A98D5;
		font-size: 30upx;
	}

	.uni-comment-date {
		line-height: 38upx;
		flex-direction: row;
		justify-content: space-between;
		display: flex !important;
		flex-grow: 1;
		color: rgba(0, 0, 0, .3);
		font-size: 24rpx;
	}

	.uni-comment-date view {
		color: #666666;
		font-size: 24upx;
		line-height: 38upx;
	}

	.uni-comment-content {
		line-height: 1.6em;
		font-size: 28upx;
		padding: 8rpx 0;
	}

	.uni-comment-reply-btn {
		background: rgba(247, 247, 247);
		font-size: 24upx;
		line-height: 28upx;
		padding: 5rpx 20upx;
		border-radius: 30upx;
		color: #333 !important;
		margin: 0 10upx;
	}

	.submit-btn {
		color: #fff;
		height: 70rpx;
		line-height: 70rpx;
		background-color: #07c160;
	}

	.floor {
		width: 70rpx;
		height: 70rpx;
		line-height: 70rpx;
		text-align: center;
		font-size: 22rpx;
		color: #ffffff;
	}
</style>
