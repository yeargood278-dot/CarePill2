<template>
	<view class="container">
		<view class="scan-bar" @click="scanPrescription">
			<text class="scan-icon">🖨️</text>
			<view class="scan-text">
				<text class="st-main">扫描医嘱或药盒自动录入</text>
				<text class="st-sub">支持智能识别药名、剂量、频次</text>
			</view>
			<text class="scan-arrow">></text>
		</view>

		<view class="card">
			<view class="section-title">💊 药品基本信息</view>
			
			<view class="input-group">
				<text class="label">药品名称</text>
				<input class="input" v-model="form.name" placeholder="请输入药名 (如: 降压药)" />
			</view>
			
			<view class="row-group">
				<view class="input-group flex-2">
					<text class="label">单次剂量</text>
					<input class="input" type="digit" v-model="form.dosage" placeholder="数量 (如: 1)" />
				</view>
				<view class="input-group flex-1" style="margin-left: 15px;">
					<text class="label">单位</text>
					<input class="input" v-model="form.unit" placeholder="(如: 片/粒)" />
				</view>
			</view>

			<view class="input-group">
				<text class="label">用药提示 (餐时要求)</text>
				<picker @change="bindMealChange" :value="mealIndex" :range="mealOptions" range-key="label">
					<view class="picker-view">{{ mealOptions[mealIndex].label }}</view>
				</picker>
			</view>

			<view class="photo-area" @click="takePhoto">
				<image v-if="form.photo" :src="form.photo" mode="aspectFill" class="preview-img" />
				<view v-else class="upload-placeholder">
					<text class="icon">📷</text>
					<text class="hint">点击拍摄药盒 (方便辨认)</text>
				</view>
			</view>
		</view>

		<view class="card">
			<view class="section-title">⏰ 设定服药时间 (可多次)</view>
			
			<view class="time-list">
				<view class="time-item" v-for="(t, index) in form.times" :key="index">
					<picker mode="time" :value="t" @change="e => updateTime(index, e.detail.value)">
						<view class="time-display">{{ t }} <text class="edit-badge">修改</text></view>
					</picker>
					<view class="del-time-btn" @click="removeTime(index)" v-if="form.times.length > 1">×</view>
				</view>
			</view>
			<button class="add-time-btn" @click="addTime">+ 增加服药时间</button>
			
			<view class="divider"></view>

			<view class="date-range-box">
				<view class="date-col">
					<text class="sub-label">开始服用</text>
					<picker mode="date" :value="form.startDate" @change="e => form.startDate = e.detail.value">
						<view class="date-val">{{ form.startDate }}</view>
					</picker>
				</view>
				<view class="arrow-icon">➡</view>
				<view class="date-col">
					<text class="sub-label">结束日期</text>
					<picker mode="date" :value="form.endDate" @change="e => form.endDate = e.detail.value">
						<view class="date-val">{{ form.endDate }}</view>
					</picker>
				</view>
			</view>
		</view>

		<view class="card">
			<view class="section-title">📦 药物库存与提醒</view>
			<view class="row-group">
				<view class="input-group flex-1">
					<text class="label">已取总药量</text>
					<input class="input" type="digit" v-model="form.stock" placeholder="总量" />
				</view>
				<view class="input-group flex-1" style="margin-left: 15px;">
					<text class="label">余量提醒阈值</text>
					<input class="input" type="digit" v-model="form.alertStock" placeholder="低于几提示" />
				</view>
			</view>
		</view>

		<view class="card">
			<view class="section-title">🔔 提醒方式设置</view>
			<checkbox-group class="methods-box" @change="e => form.methods = e.detail.value">
				<label class="method-item" :class="{active: form.methods.includes('voice')}">
					<checkbox value="voice" :checked="form.methods.includes('voice')" hidden />
					<text class="m-icon">🔊</text>
					<text>语音播报</text>
				</label>
				<label class="method-item" :class="{active: form.methods.includes('vibrate')}">
					<checkbox value="vibrate" :checked="form.methods.includes('vibrate')" hidden />
					<text class="m-icon">📳</text>
					<text>强震提醒</text>
				</label>
				<label class="method-item" :class="{active: form.methods.includes('popup')}">
					<checkbox value="popup" :checked="form.methods.includes('popup')" hidden />
					<text class="m-icon">📱</text>
					<text>弹窗显示</text>
				</label>
			</checkbox-group>
		</view>

		<button class="save-btn" @click="save">保存服药计划</button>
	</view>
</template>

<script>
export default {
	data() {
		return {
			isEditMode: false,
			mealOptions: [
				{ label: '无特殊要求', value: 'none' },
				{ label: '餐前服用', value: 'before' },
				{ label: '随餐服用', value: 'with' },
				{ label: '餐后服用', value: 'after' }
			],
			mealIndex: 0,
			form: {
				id: '',
				name: '',
				dosage: '',
				unit: '片',
				times: ['08:00'], 
				mealTiming: 'none', 
				startDate: '', // 将在 onLoad 初始化
				endDate: '',   // 将在 onLoad 初始化
				photo: '',
				stock: '',     
				alertStock: '5', 
				methods: ['voice', 'vibrate', 'popup'],
				history: {}
			}
		}
	},
	onLoad(options) {
		this.initDate();
		if (options.id) {
			this.isEditMode = true;
			this.loadData(options.id);
		}
	},
	methods: {
		getLocalDateStr(date) {
			const y = date.getFullYear();
			const m = String(date.getMonth() + 1).padStart(2, '0');
			const d = String(date.getDate()).padStart(2, '0');
			return `${y}-${m}-${d}`;
		},
		initDate() {
			const now = new Date();
			// 今日
			this.form.startDate = this.getLocalDateStr(now);
			// 一个月后（核心修改点：将周期设定为一个月，避免过长）
			const nextMonth = new Date(now.getFullYear(), now.getMonth() + 1, now.getDate());
			this.form.endDate = this.getLocalDateStr(nextMonth);
		},
		loadData(id) {
			let list = uni.getStorageSync('medicine_list') || [];
			const target = list.find(m => m.id === id);
			if (target) {
				this.form = JSON.parse(JSON.stringify(target));
				// 兼容老数据结构
				if (this.form.time && (!this.form.times || this.form.times.length === 0)) {
					this.form.times = [this.form.time];
				}
				this.mealIndex = this.mealOptions.findIndex(o => o.value === this.form.mealTiming);
				if(this.mealIndex === -1) this.mealIndex = 0;
			}
		},
		scanPrescription() {
			uni.chooseImage({
				count: 1,
				success: (res) => {
					uni.showLoading({ title: '智能识别中...' });
					setTimeout(() => {
						uni.hideLoading();
						this.form.name = '阿莫西林胶囊 (识别示例)';
						this.form.dosage = '2';
						this.form.unit = '粒';
						this.form.times = ['08:00', '12:00', '18:00'];
						this.form.mealTiming = 'after';
						this.mealIndex = 3;
						this.form.stock = '36';
						uni.showToast({ title: '已提取医嘱信息', icon: 'success' });
					}, 1500);
				}
			});
		},
		bindMealChange(e) {
			this.mealIndex = e.detail.value;
			this.form.mealTiming = this.mealOptions[this.mealIndex].value;
		},
		addTime() {
			this.form.times.push('12:00');
		},
		updateTime(index, val) {
			this.$set(this.form.times, index, val);
		},
		removeTime(index) {
			this.form.times.splice(index, 1);
		},
		takePhoto() {
			uni.chooseImage({
				count: 1,
				success: (res) => { this.form.photo = res.tempFilePaths[0]; }
			});
		},
		save() {
			if (!this.form.name || !this.form.dosage) {
				return uni.showToast({ title: '请填写药名和剂量', icon: 'none' });
			}
			
			// 去重排序时间
			this.form.times = [...new Set(this.form.times)].sort();

			let list = uni.getStorageSync('medicine_list') || [];
			
			if (this.isEditMode) {
				const idx = list.findIndex(m => m.id === this.form.id);
				if (idx !== -1) list[idx] = this.form;
			} else {
				this.form.id = 'MED_' + Date.now();
				list.push(this.form);
			}
			
			uni.setStorageSync('medicine_list', list);
			
			uni.showToast({ title: '计划已保存', icon: 'success' });
			// 返回首页
			setTimeout(() => {
				uni.navigateBack({
					delta: 1,
					fail: () => {
						uni.reLaunch({ url: '/pages/index/index' });
					}
				});
			}, 1000);
		}
	}
}
</script>

<style>
.container { padding: 20px; background: #F5F7FA; min-height: 100vh; padding-bottom: 60px; }

/* 扫码录入条 */
.scan-bar { background: linear-gradient(135deg, #11998E, #38EF7D); border-radius: 16px; padding: 18px 20px; display: flex; align-items: center; margin-bottom: 20px; color: #fff; box-shadow: 0 4px 15px rgba(17, 153, 142, 0.3); }
.scan-icon { font-size: 28px; margin-right: 15px; }
.scan-text { flex: 1; display: flex; flex-direction: column; }
.st-main { font-size: 16px; font-weight: bold; margin-bottom: 4px; }
.st-sub { font-size: 12px; opacity: 0.9; }
.scan-arrow { font-size: 20px; font-weight: bold; opacity: 0.8; }

.card { background: #fff; border-radius: 18px; padding: 22px; margin-bottom: 20px; box-shadow: 0 4px 12px rgba(0,0,0,0.03); }
.section-title { font-size: 17px; font-weight: bold; color: #333; margin-bottom: 20px; border-left: 5px solid #4A90E2; padding-left: 12px; }

.input-group { margin-bottom: 20px; }
.row-group { display: flex; justify-content: space-between; }
.flex-1 { flex: 1; }
.flex-2 { flex: 2; }

.label { font-size: 14px; color: #555; margin-bottom: 8px; display: block; font-weight: 500; }
.input { border-bottom: 1px solid #E0E0E0; height: 42px; font-size: 16px; color: #333; }
.picker-view { border-bottom: 1px solid #E0E0E0; height: 42px; line-height: 42px; font-size: 16px; color: #4A90E2; font-weight: bold; }

/* 多时间点设置 */
.time-list { display: flex; flex-direction: column; gap: 12px; margin-bottom: 15px; }
.time-item { display: flex; align-items: center; justify-content: space-between; background: #F0F7FF; padding: 10px 20px; border-radius: 12px; }
.time-display { font-size: 24px; font-weight: 800; color: #4A90E2; display: flex; align-items: center; }
.edit-badge { font-size: 12px; background: #4A90E2; color: #fff; padding: 2px 8px; border-radius: 4px; margin-left: 12px; font-weight: normal; }
.del-time-btn { width: 30px; height: 30px; border-radius: 50%; background: #FFCDD2; color: #D32F2F; text-align: center; line-height: 28px; font-size: 20px; font-weight: bold; }

.add-time-btn { background: #fff; color: #4A90E2; border: 1px dashed #4A90E2; border-radius: 12px; height: 44px; line-height: 42px; font-size: 15px; font-weight: bold; }
.add-time-btn::after { border: none; }

.photo-area { height: 130px; background: #FAFAFA; border: 2px dashed #D0D0D0; border-radius: 14px; display: flex; align-items: center; justify-content: center; overflow: hidden; margin-top: 10px; }
.upload-placeholder { display: flex; flex-direction: column; align-items: center; color: #888; }
.icon { font-size: 32px; margin-bottom: 8px; }
.preview-img { width: 100%; height: 100%; }

.divider { height: 1px; background: #F0F0F0; margin: 25px 0; }
.date-range-box { display: flex; justify-content: space-between; align-items: center; }
.date-col { flex: 1; display: flex; flex-direction: column; align-items: center; }
.sub-label { font-size: 13px; color: #888; margin-bottom: 8px; }
.date-val { background: #F5F7FA; color: #333; padding: 10px 15px; border-radius: 10px; font-weight: bold; font-size: 14px; }
.arrow-icon { color: #ccc; font-size: 20px; padding: 0 10px; }

.methods-box { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 10px; }
.method-item { display: flex; flex-direction: column; align-items: center; background: #F9F9F9; padding: 15px 5px; border-radius: 12px; font-size: 13px; color: #666; border: 2px solid transparent; transition: all 0.2s; }
.method-item.active { background: #E3F2FD; border-color: #4A90E2; color: #1976D2; font-weight: bold; }
.m-icon { font-size: 22px; margin-bottom: 5px; }

.save-btn { background: linear-gradient(135deg, #4A90E2, #0052D4); color: #fff; border-radius: 30px; height: 56px; line-height: 56px; font-size: 18px; font-weight: bold; margin-top: 30px; box-shadow: 0 8px 20px rgba(74,144,226,0.3); }
</style>