<template>
	<view class="content">
		<uni-forms :modelValue="formData" label-position="top">
			<uni-forms-item required label="用户名" name="username">
				<uni-easyinput type="text" v-model="formData.username" placeholder="请输入用户名" />
			</uni-forms-item>
			<uni-forms-item required name="password" label="密码">
				<uni-easyinput type="password" v-model="formData.password" placeholder="请输入密码" />
			</uni-forms-item>
		</uni-forms>
		<button @click="login" type="default" class="btn" :loading="loading" :disabled="loading">登录</button>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				formData: {
					username: '',
					password: ''
				},
				loading: false
			}
		},
		methods: {
			login() {
				this.loading = true
				uni.request({
					url: 'https://yaohuo.me/waplogin.aspx',
					method:'POST',
					header: {
						'Content-Type': 'application/x-www-form-urlencoded',
					},
					data: {
						logname: this.formData.username,
						logpass: this.formData.password,
						action: 'login',
						classid: 0,
						siteid: 1000,
						sid: '',
						backurl: 'wapindex.aspx?sid=-2',
						referer: '',
						remember: 1
					},
					success: (res) => {
						let cookie = res.header['Set-Cookie'].match(/sidyaohuo=(.*?);/)
						if (cookie && cookie.length) {
							uni.setStorageSync('cookie', cookie[0])
							uni.showToast({
								title: '登陆成功',
								icon: 'success'
							})
							setTimeout(() => {
								uni.redirectTo({
									url:'/pages/index/index'
								})
							}, 500)
						} else {
							uni.showToast({
								title: '账号或密码错误',
								icon: 'error'
							})
						}

					},
					fail: (err) => {
						uni.showToast({
							title: '连接服务器失败',
							icon: 'error'
						})
					},
					complete: () => {
						this.loading = false
					}
				})
			}
		}
	}
</script>

<style lang="scss" scoped>
	.content {
		padding: 0 20rpx;

		.btn {
			color: #fff;
			background-color: rgb(0, 122, 255);
		}
	}
</style>
