<template>
	<view class="container">
		<view class="header-section">
			<view class="status-badge" :class="{ running: isRunning }">
				<view class="dot"></view>
				<text>{{ isRunning ? '智能监护运行中' : '服务未启动' }}</text>
			</view>
			<view class="date-display">
				<text class="today-cn">{{ todayCN }}</text>
				<text class="week-cn">{{ weekCN }}</text>
			</view>
			<view class="stats-row">
				<view class="stat-box" @click="navTo('/pages/report/report')">
					<text class="stat-num">{{ remainingCount }}</text>
					<text class="stat-txt">待服用(次)</text>
				</view>
				<view class="stat-box" @click="navTo('/pages/report/report')">
					<text class="stat-num success">{{ doneCount }}</text>
					<text class="stat-txt">已完成(次)</text>
				</view>
			</view>
		</view>

		<scroll-view scroll-y class="list-container">
			<view class="list-title">今日服药清单</view>
			
			<view v-if="todayList.length === 0" class="empty-state">
				<text>今日暂无计划，请点击右下角添加</text>
			</view>

			<view v-for="item in todayList" :key="item.uniqueId" 
				  class="med-card" 
				  :class="item.isTaken ? 'done' : 'active'">
				
				<view class="card-header-actions" @click="showOptions(item.medId)">
					<text class="more-icon">···</text>
				</view>

				<view class="card-main">
					<view class="time-col">
						<text class="time-text">{{ item.time }}</text>
						<view class="line-node"></view>
					</view>

					<view class="info-col">
						<view class="top-row">
							<text class="med-name">{{ item.name }}</text>
							<image v-if="item.photo" :src="item.photo" mode="aspectFill" class="tiny-img" />
						</view>
						<view class="med-details">
							<text class="med-dose">剂量：{{ item.dosage }}{{ item.unit || '份' }}</text>
							<text class="med-stock" :class="{'stock-warning': Number(item.stock) <= Number(item.alertStock)}">
								库存: {{ item.stock || 0 }}{{ item.unit || '份' }}
							</text>
						</view>
						<view class="tags">
							<text v-if="item.mealTiming && item.mealTiming !== 'none'" class="tag meal-tag">{{ getMealTimingText(item.mealTiming) }}</text>
							<text v-if="item.methods && item.methods.includes('voice')" class="tag">🔊 语音</text>
							<text v-if="item.methods && item.methods.includes('vibrate')" class="tag">📳 强震</text>
							<text v-if="Number(item.stock) <= Number(item.alertStock)" class="tag alert-tag">⚠️ 需补药</text>
						</view>
					</view>

					<view class="btn-col">
						<button class="action-btn" :disabled="item.isTaken" @click="confirmTake(item.medId, item.time, item.dosage)">
							{{ item.isTaken ? '已服' : '打卡' }}
						</button>
					</view>
				</view>
			</view>
			<view style="height: 120px;"></view>
		</scroll-view>

		<view class="fab-add" @click="navTo('/pages/add/add')">
			<text class="plus-txt">+ 药</text>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			todayList: [],
			todayStr: '', 
			todayCN: '',  
			weekCN: '',
			isRunning: false,
			monitorTimer: null,
			ttsEngine: null,
			alarmInterval: null, 
			processedAlarms: [] 
		}
	},
	computed: {
		doneCount() { return this.todayList.filter(i => i.isTaken).length; },
		remainingCount() { return this.todayList.length - this.doneCount; }
	},
	onLoad() {
		this.initDate();
		this.initTTS(); 
	},
	onShow() {
		// 每次展示页面重新初始化日期，避免隔夜未刷新的问题
		this.initDate();
		this.refreshList();
		this.startGuard();
	},
	onHide() {
		if (this.monitorTimer) clearInterval(this.monitorTimer);
	},
	methods: {
		// 使用本地时间避免UTC转换错误
		getLocalDateStr(date) {
			const y = date.getFullYear();
			const m = String(date.getMonth() + 1).padStart(2, '0');
			const d = String(date.getDate()).padStart(2, '0');
			return `${y}-${m}-${d}`;
		},
		initDate() {
			const date = new Date();
			this.todayStr = this.getLocalDateStr(date);
			const w = ['日','一','二','三','四','五','六'];
			this.todayCN = `${date.getFullYear()}年${date.getMonth() + 1}月${date.getDate()}日`;
			this.weekCN = `星期${w[date.getDay()]}`;
		},
		initTTS() {
			// #ifdef APP-PLUS
			if (uni.getSystemInfoSync().platform === 'android') {
				const TTS = plus.android.importClass("android.speech.tts.TextToSpeech");
				const MainActivity = plus.android.runtimeMainWindow();
				this.ttsEngine = new TTS(MainActivity, (status) => {
					console.log("TTS引擎初始化:", status);
				});
			}
			// #endif
		},
		getMealTimingText(val) {
			const map = { before: '餐前服用', with: '随餐服用', after: '餐后服用' };
			return map[val] || '';
		},
		refreshList() {
			try {
				const list = uni.getStorageSync('medicine_list') || [];
				// 过滤出今天需要服用的药品
				let validItems = list.filter(m => this.todayStr >= m.startDate && this.todayStr <= m.endDate);
				
				let flattenedList = [];
				validItems.forEach(med => {
					// 核心修复：兼容老数据，如果没有 times 数组，就把旧的 time 包装成数组
					let timesArr = (med.times && med.times.length > 0) ? med.times : (med.time ? [med.time] : ['08:00']);
					
					timesArr.forEach(time => {
						let isTaken = false;
						if (med.history && med.history[this.todayStr] && med.history[this.todayStr][time]) {
							isTaken = true;
						}
						flattenedList.push({
							...med,
							medId: med.id,
							time: time,
							uniqueId: `${med.id}_${time}`,
							isTaken: isTaken
						});
					});
				});
				
				this.todayList = flattenedList.sort((a, b) => {
					if (a.isTaken === b.isTaken) return a.time.localeCompare(b.time);
					return a.isTaken ? 1 : -1;
				});
			} catch (e) {
				console.error("加载列表失败:", e);
				uni.showToast({ title: '数据解析异常', icon: 'none' });
			}
		},
		showOptions(medId) {
			uni.showActionSheet({
				itemList: ['📝 修改计划', '🗑 删除计划'],
				success: (res) => {
					if (res.tapIndex === 0) {
						this.navTo(`/pages/add/add?id=${medId}`);
					} else if (res.tapIndex === 1) {
						uni.showModal({
							title: '确认删除',
							content: '删除后将清空该药物的所有记录，是否继续？',
							success: (modalRes) => {
								if (modalRes.confirm) {
									let list = uni.getStorageSync('medicine_list') || [];
									list = list.filter(m => m.id !== medId);
									uni.setStorageSync('medicine_list', list);
									uni.showToast({ title: '已删除', icon: 'none' });
									this.refreshList();
								}
							}
						});
					}
				}
			});
		},
		startGuard() {
			this.isRunning = true;
			if (this.monitorTimer) clearInterval(this.monitorTimer);
			
			this.monitorTimer = setInterval(() => {
				const now = new Date();
				const time = `${String(now.getHours()).padStart(2,'0')}:${String(now.getMinutes()).padStart(2,'0')}`;
				
				this.todayList.forEach(item => {
					const alarmKey = `${this.todayStr}-${item.medId}-${time}`;
					if (item.time === time && !item.isTaken && !this.processedAlarms.includes(alarmKey)) {
						this.triggerStrongAlarm(item, alarmKey);
					}
				});
			}, 3000);
		},
		triggerStrongAlarm(item, key) {
			this.processedAlarms.push(key);
			const mealTip = (item.mealTiming && item.mealTiming !== 'none') ? `，注意${this.getMealTimingText(item.mealTiming)}` : '';
			const unitStr = item.unit || '份';
			const contentText = `现在是${item.time}，该服用${item.name}了。剂量：${item.dosage}${unitStr}${mealTip}`;

			uni.vibrateLong();

			const methods = item.methods || [];
			if (methods.includes('voice')) {
				// #ifdef APP-PLUS
				if (this.ttsEngine) this.ttsEngine.speak(contentText, 0, null);
				// #endif
				// #ifdef H5
				window.speechSynthesis.speak(new SpeechSynthesisUtterance(contentText));
				// #endif
			}

			if (methods.includes('vibrate')) {
				let count = 0;
				this.alarmInterval = setInterval(() => {
					uni.vibrateLong(); 
					count++;
					if (count % 3 === 0 && methods.includes('voice')) {
						// #ifdef APP-PLUS
						if(this.ttsEngine) this.ttsEngine.speak("请按时吃药", 1, null);
						// #endif
					}
				}, 1500);
			}

			setTimeout(() => {
				uni.showModal({
					title: '⏰ 到了服药时间',
					content: `${contentText}\n\n请确认服药后关闭提醒`,
					confirmText: '✅ 我已服药',
					cancelText: '⏱ 稍后提醒',
					showCancel: true,
					success: (res) => {
						if (this.alarmInterval) clearInterval(this.alarmInterval);
						if (res.confirm) {
							this.confirmTake(item.medId, item.time, item.dosage);
						}
					}
				});
			}, 300);
		},
		confirmTake(medId, time, dosage) {
			let list = uni.getStorageSync('medicine_list') || [];
			const idx = list.findIndex(m => m.id === medId);
			if (idx !== -1) {
				if (!list[idx].history) list[idx].history = {};
				if (!list[idx].history[this.todayStr]) list[idx].history[this.todayStr] = {};
				
				// 记录该时间点已服药
				list[idx].history[this.todayStr][time] = true;
				
				// 扣除库存逻辑
				let currentStock = Number(list[idx].stock);
				let doseNum = Number(dosage);
				if (!isNaN(currentStock) && !isNaN(doseNum)) {
					list[idx].stock = Math.max(0, currentStock - doseNum);
				}

				uni.setStorageSync('medicine_list', list);
				uni.showToast({ title: '打卡成功', icon: 'success' });
				this.refreshList();
			}
		},
		navTo(url) { uni.navigateTo({ url }); }
	}
}
</script>

<style>
.container { background: #F5F7FA; min-height: 100vh; padding-bottom: 50px; }

/* Header */
.header-section { background: linear-gradient(135deg, #4A90E2, #0052D4); padding: 50px 20px 30px; border-radius: 0 0 30px 30px; color: #fff; box-shadow: 0 4px 15px rgba(0,82,212,0.3); }
.status-badge { display: inline-flex; align-items: center; background: rgba(0,0,0,0.2); padding: 4px 12px; border-radius: 20px; font-size: 13px; margin-bottom: 12px; }
.status-badge.running .dot { background: #00FF00; box-shadow: 0 0 8px #00FF00; }
.dot { width: 8px; height: 8px; background: #ccc; border-radius: 50%; margin-right: 8px; }
.date-display { margin-bottom: 25px; }
.today-cn { font-size: 30px; font-weight: bold; display: block; letter-spacing: 1px; }
.week-cn { font-size: 16px; opacity: 0.9; margin-top: 5px; display: block; }
.stats-row { display: flex; justify-content: space-between; }
.stat-box { background: rgba(255,255,255,0.15); flex: 1; margin: 0 8px; padding: 16px; border-radius: 14px; text-align: center; backdrop-filter: blur(5px); }
.stat-num { font-size: 26px; font-weight: bold; display: block; color: #FFEB3B; }
.stat-num.success { color: #69F0AE; }
.stat-txt { font-size: 13px; opacity: 0.9; margin-top: 4px; display: block; }

/* List */
.list-container { padding: 20px; }
.list-title { font-size: 19px; font-weight: bold; color: #333; margin-bottom: 18px; border-left: 5px solid #4A90E2; padding-left: 12px; }
.empty-state { text-align: center; padding: 50px 20px; color: #999; background: #fff; border-radius: 16px; font-size: 15px; }

/* Card Style */
.med-card { background: #fff; padding: 20px; margin-bottom: 18px; border-radius: 18px; position: relative; overflow: hidden; transition: all 0.3s; }
.card-header-actions { position: absolute; top: 10px; right: 15px; z-index: 10; padding: 5px; }
.more-icon { font-size: 24px; color: #999; font-weight: bold; line-height: 1; }
.card-main { display: flex; align-items: center; margin-top: 5px; }

/* 未服药 Active */
.med-card.active { box-shadow: 0 8px 20px rgba(74, 144, 226, 0.08); border-left: 6px solid #4A90E2; }
.med-card.active .time-text { color: #4A90E2; font-weight: 800; font-size: 24px; }
.med-card.active .med-name { color: #333; font-weight: bold; font-size: 20px; }
.med-card.active .action-btn { background: linear-gradient(135deg, #4A90E2, #0052D4); color: #fff; box-shadow: 0 4px 12px rgba(74,144,226,0.3); }

/* 已服药 Done */
.med-card.done { background: #F8F9FA; opacity: 0.8; border-left: 6px solid #B0B0B0; }
.med-card.done .time-text { color: #999; font-weight: bold; font-size: 20px; text-decoration: line-through; }
.med-card.done .med-name { color: #888; text-decoration: line-through; font-size: 18px; }
.med-card.done .action-btn { background: #E0E0E0; color: #999; }

.time-col { display: flex; flex-direction: column; align-items: center; width: 75px; margin-right: 15px; border-right: 1px dashed #eee; padding-right: 10px; }
.info-col { flex: 1; }
.top-row { display: flex; align-items: center; }
.tiny-img { width: 34px; height: 34px; border-radius: 6px; margin-left: 10px; background: #eee; }

.med-details { display: flex; justify-content: space-between; align-items: center; margin-top: 6px; padding-right: 10px; }
.med-dose { font-size: 14px; color: #666; }
.med-stock { font-size: 13px; color: #888; background: #F0F0F0; padding: 2px 8px; border-radius: 10px; }
.stock-warning { color: #FF5252; background: #FFEAEA; font-weight: bold; }

.tags { margin-top: 10px; display: flex; flex-wrap: wrap; gap: 6px; }
.tag { font-size: 11px; padding: 4px 8px; background: #F0F4F8; color: #555; border-radius: 6px; }
.meal-tag { background: #E8F5E9; color: #2E7D32; font-weight: 500; }
.alert-tag { background: #FFEBEE; color: #C62828; font-weight: bold; }

.btn-col { margin-left: 10px; }
.action-btn { font-size: 14px; border-radius: 25px; padding: 0 18px; height: 42px; line-height: 42px; border: none; font-weight: bold; transition: transform 0.1s; }
.action-btn:active { transform: scale(0.95); }

/* FAB Button */
.fab-add { position: fixed; bottom: 40px; right: 25px; width: 65px; height: 65px; background: linear-gradient(135deg, #FF7043, #FF5722); color: #fff; border-radius: 50%; display: flex; align-items: center; justify-content: center; box-shadow: 0 8px 25px rgba(255, 87, 34, 0.4); z-index: 100; transition: transform 0.1s; }
.fab-add:active { transform: scale(0.95); }
.plus-txt { font-size: 18px; font-weight: bold; }
</style>
