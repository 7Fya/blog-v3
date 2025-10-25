<script setup lang="ts">
const appConfig = useAppConfig()
const timeEstablished = appConfig.timeEstablished

const blogLog = [
	{ label: '2025-08-26', value: '重构到 Nuxt 4 + Nuxt Content v3' },
	// { label: '2024-08-11', value: '重构到 Nuxt 3 + Nuxt Content v2' },
	{ label: '2025-08-24', value: '更换框架为 Hexo，升级博客' },
	// { label: '2020-08-24', value: '使用 zhilu.cyou 域名' },
	{ label: timeEstablished, value: '发布第一篇文章' },
]
</script>

<template>
<ZWidget card title="更新日志">
	<ZDlGroup size="large" :items="blogLog" />
</ZWidget>
</template>
