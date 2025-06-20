<template>
	<view class="container">
		<!-- 头部区域 -->
		<view class="header">
			<view class="logo">
				<text class="logo-text">地区手机号查询</text>
			</view>
			<view class="subtitle">输入地区和部分手机号进行查询</view>
		</view>

		<!-- 主内容区域 -->
		<view class="content">
			<!-- 表单卡片 -->
			<view class="form-card">
				<!-- 省份选择 -->
				<view class="form-group">
					<view class="form-label">选择省份</view>
					<view class="picker-wrapper">
						<picker mode="selector" :range="provinces" :range-key="'shortname'" @change="onProvinceChange"
							:value="provinceIndex">
							<view class="picker">
								{{ provinces[provinceIndex].shortname || '请选择省份' }}
								<image class="arrow-icon" src="/static/icons/arrow-down.png" mode="aspectFill"></image>
							</view>
						</picker>
					</view>
				</view>

				<!-- 城市选择 -->
				<view class="form-group" :class="{ 'disabled': !cities.length }">
					<view class="form-label">选择城市</view>
					<view class="picker-wrapper">
						<picker mode="selector" :range="cities" :range-key="'shortname'" @change="onCityChange"
							:value="cityIndex" :disabled="!cities.length">
							<view class="picker">
								<!-- 修改了这一行，移除了可选链操作符 -->
								{{ cities.length > 0 && cities[cityIndex] ? cities[cityIndex].shortname : (cities.length ? '请选择城市' : '请先选择省份') }}
								<image class="arrow-icon" src="/static/icons/arrow-down.png" mode="aspectFill"></image>
							</view>
						</picker>
					</view>
				</view>

				<!-- 手机号输入 -->
				<view class="form-group">
					<view class="form-label">手机号码</view>
					<view class="phone-inputs">
						<!-- 前三位 -->
						<view class="phone-part">
							<input type="number" maxlength="3" placeholder="前3位" v-model="phonePrefix"
								@input="formatPhone" @blur="validatePhonePart(phonePrefix, 'prefix')">
						</view>

						<view class="phone-separator">-</view>

						<!-- 中间四位 -->
						<view class="phone-part disabled">
							<text class="placeholder">****</text>
						</view>

						<view class="phone-separator">-</view>

						<!-- 后四位 -->
						<view class="phone-part">
							<input type="number" maxlength="4" placeholder="后4位" v-model="phoneSuffix"
								@input="formatPhone" @blur="validatePhonePart(phoneSuffix, 'suffix')">
						</view>
					</view>
					<view class="phone-tip" v-if="phoneError">
						<text class="error-text">{{ phoneError }}</text>
					</view>
				</view>

				<!-- 提交按钮 -->
				<view class="form-group submit-group">
					<!-- :disabled="!isFormValid" -->
					<button class="submit-btn" @click="submitForm">
						查询号码信息
					</button>
				</view>
			</view>

			<!-- 查询结果区域 -->
			<view class="result-card" v-if="showResult">
				<view class="result-header">
					<view class="result-title">查询结果</view>
					<button class="close-btn" @click="closeResult">
						<image src="/static/icons/close.png" mode="aspectFill"></image>
					</button>
				</view>

				<view class="result-content">
					<view class="result-item">
						<view class="item-label">归属地</view>
						<view class="item-value">{{ provinceName }} · {{ cityName }}</view>
					</view>

					<view class="result-item">
						<view class="item-label">手机号码</view>
						<view class="item-value">
							<text>{{ phonePrefix }}</text>
							<text class="masked">****</text>
							<text>{{ phoneSuffix }}</text>
						</view>
					</view>

					<view class="result-item">
						<view class="item-label">运营商</view>
						<view class="item-value">{{ operator }}</view>
					</view>

					<view class="result-item">
						<view class="item-label">卡类型</view>
						<view class="item-value">{{ cardType }}</view>
					</view>

					<view class="result-item">
						<view class="item-label">区号</view>
						<view class="item-value">{{ areaCode }}</view>
					</view>

					<view class="result-item">
						<view class="item-label">邮编</view>
						<view class="item-value">{{ postalCode }}</view>
					</view>
				</view>

				<view class="result-footer">
					<text class="update-time">数据更新时间: {{ updateTime }}</text>
				</view>
			</view>
		</view>

		<!-- 底部信息 -->
		<view class="footer">
			<view class="footer-text">© 2025 地区手机号查询服务</view>
			<view class="footer-links">
				<text class="footer-link">隐私政策</text>
				<text class="footer-link">使用条款</text>
				<text class="footer-link">联系我们</text>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				// 省份数据
				provinces: [],

				// 城市数据
				cities: [],

				// 当前选择的省份和城市索引
				provinceIndex: 0,
				cityIndex: 0,

				// 当前选择的省份和城市对象
				// selectedProvince: {},
				// selectedCity: {},

				// 手机号相关
				phonePrefix: '',
				phoneSuffix: '',
				phoneError: '',

				// 查询结果相关
				showResult: false,
				operator: '',
				cardType: '',
				areaCode: '',
				postalCode: '',
				updateTime: '2025-06-20',

				// 表单验证状态
				formValid: {
					province: false,
					city: false,
					phonePrefix: false,
					phoneSuffix: false
				},
				host: "http://fast250430.loc",
				provinceId: 0,
				cityId: 0,
				provinceName: "",
				cityName: "",
			}
		},

		computed: {
			// 表单是否有效
			isFormValid() {
				return this.formValid.province &&
					this.formValid.city &&
					this.formValid.phonePrefix &&
					this.formValid.phoneSuffix;
			}
		},

		onLoad() {
			// 初始化省份数据
			// this.selectedProvince = this.provinces[this.provinceIndex];
			this.formValid.province = true;

			// 加载对应城市数据
			// this.loadCities(this.selectedProvince.id);
			this.reqArea();
		},

		methods: {
			/**
			 * 默认获取全部省份 ；传入pid 获取地级市列表 ；无需获取三级城市列表
			 */
			reqArea() {
				var that = this;
				uni.request({
					url: that.host + "/api/demo/area",
					data: {
						"pid": that.provinceId,
					},
					success: (res) => {
						// console.log(res.data);
						if (that.provinceId > 0) {
							that.cities = res.data.data;
						} else {
							that.provinces = res.data.data;
							console.log(that.provinces);
						}

						// this.text = 'request success';
					}
				})

			},
			reqInfo() {
				var that = this;
				uni.request({
					url: that.host + "/api/demo/index",
					data: {
						"province": that.provinceName,
						"city": that.cityName,
						"start":that.phonePrefix,
						"end":that.phoneSuffix,
					},
					success: (res) => {
						const code =  res.data.code ;
						if(code == 1){
						  uni.showToast({
						  	title:"查询到了"
						  })	
						}else{
							uni.showModal({
								title:res.data.msg,
								
							})
						}
						// console.log();
					}
				})
			},
			// 加载城市数据
			/* loadCities(provinceId) {
				// 模拟API请求获取城市数据
				// 实际项目中应该从API获取

				// 这里使用模拟数据
				const provinceData = [];
				const cityData = [];

				this.cities = cityData[provinceId] || [];
				this.cityIndex = 0;
				this.selectedCity = this.cities[this.cityIndex] || {};
				this.formValid.city = this.cities.length > 0;
			}, */

			// 省份选择变化
			onProvinceChange(e) {
				this.provinceIndex = e.detail.value;
				// this.selectedProvince = this.provinces[this.provinceIndex];
				this.formValid.province = true;
				this.provinceId = this.provinces[this.provinceIndex]["value"];
				this.provinceName = this.provinces[this.provinceIndex]["shortname"];

				console.log("省份 ", this.provinceName, this.provinces[this.provinceIndex]);
				this.reqArea();
			},

			// 城市选择变化
			onCityChange(e) {
				this.cityIndex = e.detail.value;
				this.selectedCity = this.cities[this.cityIndex];
				this.cityName = this.cities[this.cityIndex]["shortname"];
				this.formValid.city = true;
				console.log("城市  ", this.cityName );

			},

			// 格式化手机号输入
			formatPhone() {
				// 限制前三位长度
				if (this.phonePrefix.length > 3) {
					this.phonePrefix = this.phonePrefix.substring(0, 3);
				}

				// 限制后四位长度
				if (this.phoneSuffix.length > 4) {
					this.phoneSuffix = this.phoneSuffix.substring(0, 4);
				}

				// 验证手机号部分
				this.validatePhonePart(this.phonePrefix, 'prefix');
				this.validatePhonePart(this.phoneSuffix, 'suffix');
			},

			// 验证手机号部分
			validatePhonePart(value, type) {
				if (type === 'prefix') {
					if (!value) {
						this.formValid.phonePrefix = false;
						this.phoneError = '请输入手机号前三位';
					} else if (value.length < 3) {
						this.formValid.phonePrefix = false;
						this.phoneError = '手机号前三位必须为3位数字';
					} else {
						this.formValid.phonePrefix = true;
						this.phoneError = '';
					}
				} else if (type === 'suffix') {
					if (!value) {
						this.formValid.phoneSuffix = false;
						this.phoneError = '请输入手机号后四位';
					} else if (value.length < 4) {
						this.formValid.phoneSuffix = false;
						this.phoneError = '手机号后四位必须为4位数字';
					} else {
						this.formValid.phoneSuffix = true;
						this.phoneError = '';
					}
				}
			},

			// 提交表单
			submitForm() {
				if (!this.isFormValid) {
					// return;
				}
				this.reqInfo()
				// 模拟API请求
				// 实际项目中应该发送请求到后端API

				// 生成模拟结果
				// this.generateMockResult();

				// 显示结果
				this.showResult = true;

				// 滚动到结果区域
				this.scrollToResult();
			},

			// 生成模拟结果
			generateMockResult() {
				// 根据省份和城市生成模拟结果
				const operators = ['中国移动', '中国联通', '中国电信'];
				const cardTypes = ['4G套餐', '5G套餐', '不限量套餐', '流量卡'];

				// 随机选择运营商和卡类型
				const randomOperator = operators[Math.floor(Math.random() * operators.length)];
				const randomCardType = cardTypes[Math.floor(Math.random() * cardTypes.length)];

				// 根据城市生成区号和邮编
				const areaCodeMap = {
					101: '010', // 北京
					201: '021', // 上海
					301: '020', // 广州
					302: '0755', // 深圳
					401: '0571', // 杭州
					501: '025', // 南京
					601: '0531', // 济南
					701: '028', // 成都
					801: '0591', // 福州
					901: '027', // 武汉
					1001: '0731' // 长沙
				};

				const postalCodeMap = {
					101: '100000', // 北京
					201: '200000', // 上海
					301: '510000', // 广州
					302: '518000', // 深圳
					401: '310000', // 杭州
					501: '210000', // 南京
					601: '250000', // 济南
					701: '610000', // 成都
					801: '350000', // 福州
					901: '430000', // 武汉
					1001: '410000' // 长沙
				};

				this.operator = randomOperator;
				this.cardType = randomCardType;
				this.areaCode = areaCodeMap[this.selectedCity.id] || '00000';
				this.postalCode = postalCodeMap[this.selectedCity.id] || '000000';
			},

			// 滚动到结果区域
			scrollToResult() {
				uni.createSelectorQuery().select('.result-card').boundingClientRect(function(rect) {
					uni.pageScrollTo({
						scrollTop: rect.top - 100,
						duration: 300
					});
				}).exec();
			},

			// 关闭结果
			closeResult() {
				this.showResult = false;
			}
		}
	}
</script>

<style>
	/* 基础样式 */
	.container {
		min-height: 100vh;
		background-color: #f7f9fc;
		display: flex;
		flex-direction: column;
	}

	.header {
		padding: 60rpx 30rpx;
		text-align: center;
		background: linear-gradient(135deg, #409eff, #165dff);
		color: #fff;
	}

	.logo {
		margin-bottom: 20rpx;
	}

	.logo-text {
		font-size: 48rpx;
		font-weight: bold;
		letter-spacing: 2rpx;
	}

	.subtitle {
		font-size: 28rpx;
		opacity: 0.9;
	}

	.content {
		flex: 1;
		padding: 40rpx 30rpx;
		max-width: 1200rpx;
		margin: 0 auto;
		width: 100%;
	}

	.form-card {
		background-color: #fff;
		border-radius: 20rpx;
		padding: 40rpx 30rpx;
		box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.05);
		margin-bottom: 40rpx;
	}

	.form-group {
		margin-bottom: 40rpx;
	}

	.form-label {
		font-size: 30rpx;
		font-weight: 500;
		color: #333;
		margin-bottom: 15rpx;
	}

	.picker-wrapper {
		position: relative;
		border: 1rpx solid #e6e6e6;
		border-radius: 12rpx;
		overflow: hidden;
		background-color: #fff;
	}

	.picker {
		height: 88rpx;
		line-height: 88rpx;
		padding: 0 30rpx;
		font-size: 28rpx;
		color: #666;
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.arrow-icon {
		width: 24rpx;
		height: 24rpx;
		opacity: 0.6;
	}

	.disabled {
		opacity: 0.5;
		pointer-events: none;
	}

	.phone-inputs {
		display: flex;
		align-items: center;
	}

	.phone-part {
		flex: 1;
		border: 1rpx solid #e6e6e6;
		border-radius: 12rpx;
		height: 88rpx;
		line-height: 88rpx;
		padding: 0 20rpx;
		font-size: 32rpx;
		text-align: center;
		background-color: #fff;
	}

	.phone-separator {
		width: 20rpx;
		text-align: center;
		font-size: 32rpx;
		color: #999;
	}

	.placeholder {
		color: #999;
	}

	.phone-tip {
		margin-top: 10rpx;
		padding-left: 10rpx;
	}

	.error-text {
		color: #f56c6c;
		font-size: 24rpx;
	}

	.submit-group {
		margin-top: 60rpx;
		margin-bottom: 0;
	}

	.submit-btn {
		width: 100%;
		height: 96rpx;
		line-height: 96rpx;
		background: linear-gradient(135deg, #409eff, #165dff);
		color: #fff;
		border-radius: 48rpx;
		font-size: 32rpx;
		font-weight: 500;
		box-shadow: 0 8rpx 20rpx rgba(64, 158, 255, 0.3);
	}

	.submit-btn[disabled] {
		opacity: 0.5;
		box-shadow: none;
	}

	.result-card {
		background-color: #fff;
		border-radius: 20rpx;
		padding: 40rpx 30rpx;
		box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.05);
		animation: fadeIn 0.3s ease-in-out;
	}

	.result-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 30rpx;
		padding-bottom: 20rpx;
		border-bottom: 1rpx solid #eee;
	}

	.result-title {
		font-size: 32rpx;
		font-weight: 500;
		color: #333;
	}

	.close-btn {
		width: 48rpx;
		height: 48rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		opacity: 0.6;
	}

	.close-btn image {
		width: 24rpx;
		height: 24rpx;
	}

	.result-content {
		margin-bottom: 30rpx;
	}

	.result-item {
		display: flex;
		padding: 20rpx 0;
		border-bottom: 1rpx solid #f5f5f5;
	}

	.result-item:last-child {
		border-bottom: none;
	}

	.item-label {
		width: 240rpx;
		font-size: 28rpx;
		color: #666;
	}

	.item-value {
		flex: 1;
		font-size: 28rpx;
		color: #333;
		font-weight: 500;
	}

	.masked {
		color: #999;
		margin: 0 10rpx;
	}

	.result-footer {
		text-align: right;
	}

	.update-time {
		font-size: 24rpx;
		color: #999;
	}

	.footer {
		padding: 40rpx 30rpx;
		text-align: center;
		background-color: #fff;
		border-top: 1rpx solid #eee;
	}

	.footer-text {
		font-size: 24rpx;
		color: #666;
		margin-bottom: 20rpx;
	}

	.footer-links {
		display: flex;
		justify-content: center;
	}

	.footer-link {
		font-size: 24rpx;
		color: #999;
		margin: 0 20rpx;
	}

	/* 动画效果 */
	@keyframes fadeIn {
		from {
			opacity: 0;
			transform: translateY(20rpx);
		}

		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	/* 响应式设计 */
	@media screen and (min-width: 750px) {
		.content {
			padding: 60rpx 30rpx;
		}

		.form-card,
		.result-card {
			padding: 60rpx 50rpx;
		}

		.form-group {
			margin-bottom: 50rpx;
		}

		.submit-group {
			margin-top: 80rpx;
		}
	}
</style>