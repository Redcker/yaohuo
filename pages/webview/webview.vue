<template>
	<view>
		<web-view :src="url"></web-view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				url: ''
			}
		},
		methods: {

		},
		onLoad(option) {
			this.url = option.url
		}
	}
</script>

<style>

</style>
