<template>
	<view class="container">
		<view class="dashboard-card">
			<view class="dash-title">全周期服药健康报告</view>
			
			<view class="data-grid">
				<view class="grid-item">
					<text class="g-num">{{ totalPlanDays }}</text>
					<text class="g-lbl">计划总天数</text>
				</view>
				<view class="grid-item">
					<text class="g-num">{{ totalPlanTimes }}</text>
					<text class="g-lbl">计划总次数</text>
				</view>
				<view class="grid-item">
					<text class="g-num blue">{{ takenTimes }}</text>
					<text class="g-lbl">已服次数</text>
				</view>
				<view class="grid-item">
					<text class="g-num red">{{ missedTimes }}</text>
					<text class="g-lbl">漏服次数</text>
				</view>
			</view>

			<view class="remain-box">
				<text class="remain-text">剩余待服：<text class="highlight">{{ remainingTimes }}</text> 次</text>
			</view>

			<view class="adherence-bar">
				<view class="bar-header">
					<text>计划完成率 (依从性)</text>
					<text class="rate-txt" :class="rateColor">{{ adherenceRate }}%</text>
				</view>
				<view class="progress-bg">
					<view class="progress-fill" :class="rateBg" :style="{width: adherenceRate + '%'}"></view>
				</view>
			</view>
		</view>

		<view class="warning-section" v-if="lowStockList.length > 0">
			<view class="warn-title">⚠️ 以下药物余量不足，请及时补充</view>
			<view class="warn-list">
				<view class="warn-item" v-for="w in lowStockList" :key="w.id">
					<text class="w-name">{{ w.name }}</text>
					<text class="w-stock">仅剩 {{ w.stock || 0 }}{{ w.unit || '份' }}</text>
				</view>
			</view>
		</view>

		<view class="history-section">
			<view class="sec-head">
				<text class="sec-title">详细打卡记录</text>
				<text class="sec-sub">时间倒序</text>
			</view>

			<view v-if="logs.length === 0" class="empty-tip">暂无数据</view>

			<view v-for="day in logs" :key="day.date" class="day-group">
				<view class="day-header">{{ formatDateCN(day.date) }}</view>
				
				<view v-for="(log, idx) in day.items" :key="idx" class="log-item" :class="log.status">
					<view class="log-left">
						<text class="log-time">{{ log.time }}</text>
						<view class="log-line"></view>
					</view>
					<view class="log-content">
						<text class="log-name">{{ log.name }}</text>
						<text class="log-status">{{ log.status === 'taken' ? '✅ 已按时服用' : '❌ 未按时服用' }}</text>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			logs: [],
			lowStockList: [],
			totalPlanDays: 0,
			totalPlanTimes: 0,
			takenTimes: 0,
			missedTimes: 0,
			remainingTimes: 0,
			adherenceRate: 100
		}
	},
	computed: {
		rateColor() {
			if(this.adherenceRate >= 90) return 'text-green';
			if(this.adherenceRate >= 60) return 'text-orange';
			return 'text-red';
		},
		rateBg() {
			if(this.adherenceRate >= 90) return 'bg-green';
			if(this.adherenceRate >= 60) return 'bg-orange';
			return 'bg-red';
		}
	},
	onShow() {
		this.calculateStats();
	},
	methods: {
		getLocalDateStr(date) {
			const y = date.getFullYear();
			const m = String(date.getMonth() + 1).padStart(2, '0');
			const d = String(date.getDate()).padStart(2, '0');
			return `${y}-${m}-${d}`;
		},
		formatDateCN(dateStr) {
			const [y, m, d] = dateStr.split('-');
			return `${y}年${parseInt(m)}月${parseInt(d)}日`;
		},
		calculateStats() {
			const list = uni.getStorageSync('medicine_list') || [];
			// 修复UTC导致的日期异常
			const todayStr = this.getLocalDateStr(new Date());
			const nowHourMinute = new Date().getHours() + ':' + String(new Date().getMinutes()).padStart(2,'0');
			
			let allLogs = [];
			let globalPlanDays = 0;
			let globalPlanTimes = 0;
			let globalTaken = 0;
			let globalMissed = 0;
			
			this.lowStockList = [];

			list.forEach(med => {
				if (Number(med.stock) <= Number(med.alertStock)) {
					this.lowStockList.push(med);
				}

				// 兼容旧数据
				let timesArr = (med.times && med.times.length > 0) ? med.times : (med.time ? [med.time] : ['08:00']);
				
				let startD = new Date(med.startDate.replace(/-/g, '/')); // 兼容iOS日期格式
				let endD = new Date(med.endDate.replace(/-/g, '/'));
				
				let timeDiff = endD.getTime() - startD.getTime();
				let daysCount = Math.ceil(timeDiff / (1000 * 3600 * 24)) + 1;
				if(daysCount < 0) daysCount = 0;
				
				globalPlanDays = Math.max(globalPlanDays, daysCount); 
				let medTotalTimes = daysCount * timesArr.length;
				globalPlanTimes += medTotalTimes;

				let loopDate = new Date(startD);
				// 限制最大统计日期为今天或结束日期
				let limitStr = todayStr < med.endDate ? todayStr : med.endDate;
				let limitDate = new Date(limitStr.replace(/-/g, '/'));
				
				while (loopDate <= limitDate) {
					let dStr = this.getLocalDateStr(loopDate);
					let isToday = (dStr === todayStr);

					timesArr.forEach(t => {
						let isTaken = false;
						if (med.history && med.history[dStr] && med.history[dStr][t]) {
							isTaken = true;
						}

						if (isTaken) {
							allLogs.push({ date: dStr, time: t, name: med.name, status: 'taken' });
							globalTaken++;
						} else {
							let isMissed = false;
							if (!isToday) {
								isMissed = true; 
							} else {
								if (t < nowHourMinute) isMissed = true; 
							}

							if (isMissed) {
								allLogs.push({ date: dStr, time: t, name: med.name, status: 'missed' });
								globalMissed++;
							}
						}
					});
					loopDate.setDate(loopDate.getDate() + 1);
				}
			});

			this.totalPlanDays = globalPlanDays;
			this.totalPlanTimes = globalPlanTimes;
			this.takenTimes = globalTaken;
			this.missedTimes = globalMissed;
			this.remainingTimes = Math.max(0, globalPlanTimes - (globalTaken + globalMissed));

			let pastTotal = globalTaken + globalMissed;
			this.adherenceRate = pastTotal === 0 ? 100 : Math.round((globalTaken / pastTotal) * 100);

			allLogs.sort((a, b) => b.date.localeCompare(a.date) || b.time.localeCompare(a.time));

			let groups = [];
			allLogs.forEach(log => {
				let g = groups.find(x => x.date === log.date);
				if (!g) { g = { date: log.date, items: [] }; groups.push(g); }
				g.items.push(log);
			});
			this.logs = groups;
		}
	}
}
</script>

<style>
.container { padding: 20px; background: #F5F7FA; min-height: 100vh; }

/* Dashboard */
.dashboard-card { background: #fff; padding: 25px; border-radius: 20px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); margin-bottom: 20px; }
.dash-title { font-size: 18px; font-weight: bold; color: #333; margin-bottom: 25px; text-align: center; }

.data-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 20px; }
.grid-item { background: #FAFAFA; padding: 15px; border-radius: 12px; text-align: center; }
.g-num { font-size: 28px; font-weight: 800; display: block; margin-bottom: 4px; color: #333;}
.g-num.blue { color: #4A90E2; }
.g-num.red { color: #FF5252; }
.g-lbl { font-size: 13px; color: #888; }

.remain-box { background: #F0F7FF; padding: 12px; border-radius: 12px; text-align: center; margin-bottom: 25px; }
.remain-text { font-size: 15px; color: #555; }
.highlight { font-size: 20px; font-weight: bold; color: #4A90E2; margin: 0 5px; }

.adherence-bar { background: #fff; border: 1px solid #EAEAEA; padding: 15px; border-radius: 12px; }
.bar-header { display: flex; justify-content: space-between; margin-bottom: 8px; font-size: 14px; color: #555; font-weight: bold;}
.rate-txt { font-weight: 800; font-size: 16px; }

.progress-bg { height: 14px; background: #E0E0E0; border-radius: 7px; overflow: hidden; }
.progress-fill { height: 100%; transition: width 0.6s ease; }

.text-green { color: #4CAF50; }
.text-orange { color: #FF9800; }
.text-red { color: #FF5252; }
.bg-green { background: linear-gradient(90deg, #66BB6A, #4CAF50); }
.bg-orange { background: linear-gradient(90deg, #FFB74D, #FF9800); }
.bg-red { background: linear-gradient(90deg, #EF5350, #F44336); }

/* Warning Section */
.warning-section { background: #FFF3E0; border: 1px solid #FFE0B2; border-radius: 16px; padding: 18px; margin-bottom: 25px; }
.warn-title { font-size: 15px; font-weight: bold; color: #E65100; margin-bottom: 12px; }
.warn-list { display: flex; flex-direction: column; gap: 8px; }
.warn-item { display: flex; justify-content: space-between; background: rgba(255,255,255,0.6); padding: 8px 12px; border-radius: 8px; font-size: 14px; }
.w-name { color: #333; font-weight: bold; }
.w-stock { color: #D84315; font-weight: bold; }

/* List */
.sec-head { display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 20px; }
.sec-title { font-size: 18px; font-weight: bold; color: #333; }
.sec-sub { font-size: 12px; color: #999; }

.day-group { margin-bottom: 25px; }
.day-header { font-size: 14px; font-weight: bold; color: #555; margin-bottom: 12px; background: #E8EAF0; display: inline-block; padding: 4px 12px; border-radius: 6px; }

.log-item { display: flex; padding: 16px; border-radius: 14px; margin-bottom: 12px; background: #fff; box-shadow: 0 2px 8px rgba(0,0,0,0.03); border-left: 5px solid transparent; }
.log-item.taken { border-color: #4CAF50; }
.log-item.missed { border-color: #FF5252; background: #FFF5F5; }

.log-left { width: 65px; text-align: center; border-right: 1px solid #F0F0F0; margin-right: 15px; display: flex; flex-direction: column; justify-content: center; }
.log-time { font-weight: bold; color: #333; font-size: 17px; }

.log-content { flex: 1; display: flex; flex-direction: column; justify-content: center; }
.log-name { font-size: 16px; font-weight: bold; color: #333; }
.log-status { font-size: 13px; margin-top: 5px; }
.log-item.taken .log-status { color: #4CAF50; }
.log-item.missed .log-status { color: #FF5252; font-weight: bold; }

.empty-tip { text-align: center; color: #ccc; margin-top: 50px; }
</style>