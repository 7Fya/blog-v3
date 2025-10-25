<script setup lang="ts">
// import type { CSSProperties } from 'vue'
// 1. 引入农历转换库（需先安装：npm install lunar-javascript）
import { Lunar } from 'lunar-javascript'
import { computed, onMounted, onUnmounted, ref } from 'vue'

const appConfig = useAppConfig()

// 2. 重构节日数据：区分「公历固定日期」和「农历固定日期」
const festivals = ref([
	// 公历节日（固定月日，无需转换）
	{ name: '元旦', type: 'solar', month: 1, day: 1 },
	{ name: '妇女节', type: 'solar', month: 3, day: 8 },
	{ name: '清明节', type: 'solar', month: 4, day: 5 },
	{ name: '劳动节', type: 'solar', month: 5, day: 1 },
	{ name: '青年节', type: 'solar', month: 5, day: 4 },
	{ name: '儿童节', type: 'solar', month: 6, day: 1 },
	{ name: '建党节', type: 'solar', month: 7, day: 1 },
	{ name: '建军节', type: 'solar', month: 8, day: 1 },
	{ name: '教师节', type: 'solar', month: 9, day: 10 },
	{ name: '国庆节', type: 'solar', month: 10, day: 1 },

	// 农历节日（固定农历月日，需转换为公历）
	{ name: '春节', type: 'lunar', month: 1, day: 1 }, // 农历正月初一
	{ name: '端午节', type: 'lunar', month: 5, day: 5 }, // 农历五月初五
	{ name: '中秋节', type: 'lunar', month: 8, day: 15 }, // 农历八月十五
])

const contentStyle = computed<CSSProperties>(() => ({
	'--seasonal-bg': appConfig.seasonal?.widgetBackground
		? `url(${appConfig.seasonal.widgetBackground})`
		: undefined,
	'--seasonal-emoji': appConfig.seasonal?.emoji || '🎯',
}))

const now = ref(new Date())

// 3. 新增：农历转公历的工具函数
function getSolarDate(festival: { type: string, month: number, day: number }, year: number) {
	if (festival.type === 'solar') {
		// 公历节日：直接生成日期（month-1 因JS月份从0开始）
		return new Date(year, festival.month - 1, festival.day)
	} else {
		// 农历节日：用库转换为公历
		const lunarDate = Lunar.fromYmd(year, festival.month, festival.day) // 农历日期
		const solarDate = lunarDate.getSolar() // 转为公历
		// 返回JS Date对象（注意库返回的月份是1-12，需减1）
		return new Date(solarDate.getYear(), solarDate.getMonth() - 1, solarDate.getDay())
	}
}

// 4. 修改：计算下一个节日（兼容农历转换）
const nextFestival = computed(() => {
	const current = new Date(now.value)
	const currentYear = current.getFullYear()

	// 生成「今年+明年」的所有节日日期（处理跨年）
	const festivalDates = festivals.value.flatMap(festival => {
		// 今年的节日日期（公历/农历转换后）
		const thisYearDate = getSolarDate(festival, currentYear)
		// 明年的节日日期（防止年底时显示明年初的节日）
		const nextYearDate = getSolarDate(festival, currentYear + 1)

		return [
			{ ...festival, dateObj: thisYearDate, year: currentYear },
			{ ...festival, dateObj: nextYearDate, year: currentYear + 1 }
		]
	})

	// 筛选未来的节日 + 按时间排序
	const futureFestivals = festivalDates
		.filter(f => f.dateObj.getTime() >= current.getTime())
		.sort((a, b) => a.dateObj.getTime() - b.dateObj.getTime())

	// 取最近的节日
	if (futureFestivals.length > 0) {
		const next = futureFestivals[0]
		const daysLeft = Math.ceil(
			(next.dateObj.getTime() - current.getTime()) / (1000 * 60 * 60 * 24),
		)

		// 格式化显示日期（补0，如1月→01）
		const month = String(next.dateObj.getMonth() + 1).padStart(2, '0')
		const day = String(next.dateObj.getDate()).padStart(2, '0')

		return {
			...next,
			daysLeft,
			displayDate: `${next.year}-${month}-${day}`, // 如2025-01-29（2025年春节）
			typeTag: next.type === 'lunar' ? '(农历)' : '' // 标注农历节日
		}
	}

	return null
})

// 进度百分比计算（不变）
function calcPercent(start: Date, end: Date, cur: Date) {
	const total = end.getTime() - start.getTime()
	const passed = cur.getTime() - start.getTime()
	return Math.min(100, Math.max(0, (passed / total) * 100))
}

// 剩余时间文案（不变）
function calcRemain(end: Date, cur: Date) {
	const diff = end.getTime() - cur.getTime()
	if (diff <= 0)
		return '已结束'
	const days = Math.floor(diff / (1000 * 60 * 60 * 24))
	const hours = Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60))
	if (days > 0)
		return `还剩 ${days} 天`
	if (hours > 0)
		return `还剩 ${hours} 小时`
	return '不足1小时'
}

// 进度条颜色计算（不变）
function getProgressColor(percent: number): string {
	// eslint-disable-next-line style/max-statements-per-line
	const baseR = 66; const baseG = 90; const baseB = 239
	const depthRatio = percent / 100
	const r = Math.round(baseR * depthRatio + 200 * (1 - depthRatio))
	const g = Math.round(baseG * depthRatio + 200 * (1 - depthRatio))
	const b = Math.round(baseB * depthRatio + 220 * (1 - depthRatio))
	return `rgb(${r}, ${g}, ${b})`
}

// 今日/本周/本月/本年进度（不变）
const progressList = computed(() => {
	const cur = now.value

	// eslint-disable-next-line style/max-statements-per-line
	const todayStart = new Date(cur); todayStart.setHours(0, 0, 0, 0)
	// eslint-disable-next-line style/max-statements-per-line
	const todayEnd = new Date(todayStart); todayEnd.setHours(23, 59, 59, 999)

	// eslint-disable-next-line style/max-statements-per-line, prefer-const
	const weekStart = new Date(cur); let wd = weekStart.getDay() || 7; weekStart.setHours(0,0,0,0); weekStart.setDate(weekStart.getDate() - wd + 1)
	// eslint-disable-next-line style/max-statements-per-line
	const weekEnd = new Date(weekStart); weekEnd.setDate(weekEnd.getDate() + 6); weekEnd.setHours(23,59,59,999)

	const monthStart = new Date(cur.getFullYear(), cur.getMonth(), 1)
	const monthEnd = new Date(cur.getFullYear(), cur.getMonth() + 1, 0, 23, 59, 59, 999)

	const yearStart = new Date(cur.getFullYear(), 0, 1)
	const yearEnd = new Date(cur.getFullYear(), 11, 31, 23, 59, 59, 999)

	return [
		{ label: '今日', start: todayStart, end: todayEnd },
		{ label: '本周', start: weekStart, end: weekEnd },
		{ label: '本月', start: monthStart, end: monthEnd },
		{ label: '本年', start: yearStart, end: yearEnd },
	].map(({ label, start, end }) => {
		const percent = calcPercent(start, end, cur)
		const remain = calcRemain(end, cur)
		const isCovered = percent > 50
		const color = getProgressColor(percent)
		return { label, percent, remain, isCovered, color }
	})
})

// 悬停切换（不变）
const showRemain = ref(false)
// eslint-disable-next-line style/max-statements-per-line
function onEnter() { showRemain.value = true }
// eslint-disable-next-line style/max-statements-per-line
function onLeave() { showRemain.value = false }

// 自动刷新（不变）
onMounted(() => {
	// eslint-disable-next-line style/max-statements-per-line
	const t = setInterval(() => { now.value = new Date() }, 1000 * 60)
	onUnmounted(() => clearInterval(t))
})
</script>

<template>
<ZWidget


	:style="contentStyle"
>
	<!-- content-class="seasonal" -->
	<!-- 倒计时卡片 -->
	<div class="countdown-card" @mouseenter="onEnter" @mouseleave="onLeave">
		<!-- 左侧：节日倒计时（新增农历标注） -->
		<div class="left">
			<div class="festival-header">
				<span class="seasonal-emoji"></span>
				<span>距离</span>
			</div>

			<div v-if="nextFestival" class="festival-info">
				<!-- 显示节日名称 + 农历标注（如“春节 (农历)”） -->
				<div class="festival-name">
					{{ nextFestival.name }}
					<span class="lunar-tag">{{ nextFestival.typeTag }}</span>
				</div>

				<div class="countdown-days">
					<span class="days-number">{{ nextFestival.daysLeft }}</span>
					<span class="days-text">天</span>
				</div>
				<div class="festival-date">{{ nextFestival.displayDate }}</div>
			</div>

			<div v-else class="no-festival">
				暂无节日数据
			</div>
		</div>

		<!-- 右侧：进度条（不变） -->
		<div class="right">
			<div
				class="progress-item-wrap"
				v-for="item in progressList"
				:key="item.label"
			>
				<div class="progress-item">
					<span class="label">{{ item.label }}</span>
					<div class="bar">
						<div
							class="fill"
							:style="{
								width: `${item.percent.toFixed(2)}%`,
								backgroundColor: item.color,
							}"
						/>
						<div class="value-container">
							<span
								class="value-text percent-text"
								:style="{
									color: item.isCovered ? '#ffffff' : '#2c2c2f',
									transform: showRemain ? 'translateX(-30%)' : 'translateX(0)',
									opacity: showRemain ? 0 : 1,
									filter: showRemain ? 'blur(3px)' : 'blur(0)',
								}"
							>
								{{ `${item.percent.toFixed(2)}%` }}
							</span>
							<span
								class="value-text remain-text"
								:style="{
									color: item.isCovered ? '#ffffff' : '#2c2c2f',
									transform: showRemain ? 'translateX(0)' : 'translateX(30%)',
									opacity: showRemain ? 1 : 0,
									filter: showRemain ? 'blur(0)' : 'blur(3px)',
								}"
							>
								{{ item.remain }}
							</span>
						</div>
					</div>
				</div>
			</div>
		</div>
	</div>
</ZWidget>
</template>

<style lang="scss" scoped>
// :deep(.seasonal) {
// 	position: relative;
// 	z-index: 0;

// 	&::before {
// 		content: "";
// 		position: absolute;
// 		opacity: 0.2;
// 		inset: 0;
// 		border-radius: 0.8rem;
// 		background: center / cover;
// 		background-image: var(--seasonal-bg);
// 		z-index: -1;
// 	}
// }

// .seasonal-emoji::before,
// .seasonal-emoji::after {
// 	content: var(--seasonal-emoji, "\1F595");
// }

.countdown-card {
	// width: 100%;
	display: flex;
	align-items: center;
	gap: 16px;
	// margin:-5px ;
	// margin-bottom: 16px;
	cursor: pointer;
	border: 1px solid #e0e0e0;
	border-radius: 12px;
	padding: 12px;
	background-color: rgba(255, 255, 255, 0.7);
	transition: border-color 0.3s ease;
	&:hover {
		border-color: #425aef;
	}
}

// 左侧节日信息（新增农历标签样式）
.left {
	text-align: center;
	padding: 8px;
	margin-left: -9px;

	.festival-header {
		font-size: 14px;
		color: #666;
		margin-bottom: 8px;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.festival-info {
		margin: 8px 0;

		.festival-name {
			font-size: 18px;
			font-weight: bold;
			color: #333;
			margin-bottom: 4px;

			// 农历标签样式（灰色小字体）
			.lunar-tag {
				font-size: 12px;
				font-weight: normal;
				color: #999;
				margin-left: 4px;
			}
		}

		.festival-date {
			font-size: 13px;
			color: #999;
			margin-bottom: 8px;
		}
	}

	.countdown-days {
		display: flex;
		align-items: center;
		justify-content: center;
		margin: 8px 0;

		.days-number {
			font-size: 32px;
			font-weight: bold;
			color: #5a4df0;
			line-height: 1;
		}

		.days-text {
			font-size: 16px;
			color: #666;
			margin-left: 4px;
		}
	}

	.no-festival {
		color: #999;
		padding: 16px 0;
	}
}

.right { flex: 1; margin-left: -15px;}

.progress-item-wrap {
	margin-bottom: 8px;
}

.progress-item {
	display: flex;
	align-items: center;
}
.label {
	width: 40px;
	font-size: 14px;
	color: #666;
}

.bar {
	position: relative;
	flex: 1;
	height: 20px;
	background: #e3e8f7;
	border-radius: 8px;
	margin-left: 6px;
	overflow: hidden;
}
.fill {
	position: absolute;
	inset: 0 auto 0 0;
	height: 100%;
	border-radius: 8px;
	transition: width 0.9s ease, background-color 0.9s ease;
}

.value-container {
	position: absolute;
	inset: 0;
	display: flex;
	align-items: center;
	padding-left: 6px;
	overflow: visible;
	pointer-events: none;
}

.value-text {
	position: absolute;
	white-space: nowrap;
	transition: all 0.8s cubic-bezier(0.34, 1.56, 0.64, 1);
	font-size: 14px;
	z-index: 2;
}

.percent-text {
	font-size: 11px;
	transform: translateX(0);
	opacity: 1;
	filter: blur(0);
}

.remain-text {
	font-size: 11px;
	transform: translateX(30%);
	opacity: 0;
	filter: blur(3px);
}

p {
	font-size: 12px;
	color: #666; /* 恢复p标签文字颜色，避免透明 */
	margin: 8px 0 0 0;
	line-height: 1.5;
}
</style>
