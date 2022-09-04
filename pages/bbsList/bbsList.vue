<template>
	<view>
		<post-list ref="postList" :url="url"></post-list>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				url: ''
			}
		},
		onLoad(option) {
			this.url = JSON.parse(decodeURIComponent(option.url)).url
		},
		onReachBottom() {
			this.$refs.postList.loadMore()
		},
		onPullDownRefresh() {
			this.$refs.postList.refreshData()
		},
		methods: {

		}
	}
</script>

<style>
	page {
		background-color: #F3F3F3;
	}
</style>
