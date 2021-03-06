<template>
	<view
		class="u-input"
		:class="{
			'u-input--border': border,
			'u-input--error': validateState
		}"
		:style="{
			padding: `0 ${border ? 20 : 0}rpx`,
			borderColor: borderColor,
			textAlign: inputAlign
		}"
		@tap.stop="inputClick"
	>
		<textarea
			v-if="type == 'textarea'"
			class="u-input__input u-input__textarea"
			:style="[getStyle]"
			:value="value"
			:placeholder="placeholder"
			:placeholderStyle="placeholderStyle"
			:disabled="disabled"
			:maxlength="inputMaxlength"
			:fixed="fixed"
			:focus="focus"
			:autoHeight="autoHeight"
			@input="handleInput"
			@blur="handleBlur"
			@focus="onFocus"
			@confirm="onConfirm"
		/>
		<input
			v-else
			class="u-input__input"
			:type="type == 'password' ? 'text' : type"
			:style="[getStyle]"
			:value="defaultValue"
			:password="type == 'password' && !showPassword"
			:placeholder="placeholder"
			:placeholderStyle="placeholderStyle"
			:disabled="disabled || type === 'select'"
			:maxlength="inputMaxlength"
			:focus="focus"
			:confirmType="confirmType"
			@focus="onFocus"
			@blur="handleBlur"
			@input="handleInput"
			@confirm="onConfirm"
		/>
		<view class="u-input__right-icon u-flex">
			<view class="u-input__right-icon__clear u-input__right-icon__item" v-if="clearable && value && focused">
				<u-icon size="32" name="close-circle-fill" color="#c0c4cc" @touchstart="onClear"/>
			</view>
			<view class="u-input__right-icon__clear u-input__right-icon__item" v-if="passwordIcon && type == 'password'">
				<u-icon size="32" :name="!showPassword ? 'eye' : 'eye-fill'" color="#c0c4cc" @click="showPassword = !showPassword"/>
			</view>
			<view class="u-input__right-icon--select u-input__right-icon__item" v-if="type == 'select'" :class="{
				'u-input__right-icon--select--reverse': selectOpen
			}">
				<u-icon name="arrow-down-fill" size="26" color="#c0c4cc"></u-icon>
			</view>
		</view>
	</view>
</template>

<script>
import Emitter from '../../libs/util/emitter.js';
export default {
	name: 'u-input',
	mixins: [Emitter],
	props: {
		value: {
			type: String,
			default: ''
		},
		// 输入框的类型，textarea，text，number
		type: {
			type: String,
			default: 'text'
		},
		clearable: {
			type: Boolean,
			default: true
		},
		inputAlign: {
			type: String,
			default: 'left'
		},
		placeholder: {
			type: String,
			default: '请输入内容'
		},
		disabled: {
			type: Boolean,
			default: false
		},
		maxlength: {
			type: [Number, String],
			default: 140
		},
		placeholderStyle: {
			type: String,
			default: 'color: #c0c4cc;'
		},
		confirmType: {
			type: String,
			default: 'done'
		},
		// 输入框的自定义样式
		customStyle: {
			type: Object,
			default() {
				return {};
			}
		},
		// 如果 textarea 是在一个 position:fixed 的区域，需要显示指定属性 fixed 为 true
		fixed: {
			type: Boolean,
			default: false
		},
		// 是否自动获得焦点
		focus: {
			type: Boolean,
			default: false
		},
		// 密码类型时，是否显示右侧的密码图标
		passwordIcon: {
			type: Boolean,
			default: true
		},
		// input|textarea是否显示边框
		border: {
			type: Boolean,
			default: false
		},
		// 输入框的边框颜色
		borderColor: {
			type: String,
			default: '#dcdfe6'
		},
		autoHeight: {
			type: Boolean,
			default: true
		},
		// type=select时，旋转右侧的图标，标识当前处于打开还是关闭select的状态
		// open-打开，close-关闭
		selectOpen: {
			type: Boolean,
			default: false
		},
		// 高度，单位rpx
		height: {
			type: [Number, String],
			default: ''
		},
		// 是否可清空
		clearable: {
			type: Boolean,
			default: true
		},
	},
	data() {
		return {
			defaultValue: this.value,
			inputHeight: 70, // input的高度
			textareaHeight: 100, // textarea的高度
			validateState: false, // 当前input的验证状态，用于错误时，边框是否改为红色
			focused: false, // 当前是否处于获得焦点的状态
			showPassword: false, // 是否预览密码
			marginRight: 0, // 输入框右边的距离，当获得焦点时各一个后面的距离，避免点击右边图标误触输入框
		};
	},
	watch: {
		value(nVal, oVal) {
			this.defaultValue = nVal;
			// 当值发生变化，且为select类型时(此时input被设置为disabled，不会触发@input事件)，模拟触发@input事件
			if(nVal != oVal && this.type == 'select') this.handleInput({
				detail: {
					value: nVal
				}
			})
		},
		focused(nVal) {
			if(this.clearable && this.value) {
				this.getMarginRight();
			}
		}
	},
	computed: {
		// 因为uniapp的input组件的maxlength组件必须要数值，这里转为数值，给用户可以传入字符串数值
		inputMaxlength() {
			return Number(this.maxlength);
		},
		getStyle() {
			let style = {};
			// 如果没有自定义高度，就根据type为input还是textare来分配一个默认的高度
			style.minHeight = this.height ? this.height + 'rpx' : this.type == 'textarea' ? 
				this.textareaHeight + 'rpx' : this.inputHeight + 'rpx';
			style.marginRight = this.marginRight + 'px';
			style = Object.assign(style, this.customStyle);
			return style;
		}
	},
	created() {
		// 监听u-form-item发出的错误事件，将输入框边框变红色
		this.$on('on-form-item-error', this.onFormItemError);
	},
	mounted() {
		this.getMarginRight();
	},
	methods: {
		// 计算输入框的右边距
		getMarginRight() {
			this.$nextTick(() => {
				this.$uGetRect('.u-input__right-icon').then(res => {
					// 此处20rpx为图标绝对定位右侧的“right”
					this.marginRight = res.width + uni.upx2px(20);
				})
			})
		},
		/**
		 * change 事件
		 * @param event
		 */
		handleInput(event) {
			// 当前model 赋值
			this.defaultValue = event.detail.value;
			// vue 原生的方法 return 出去
			this.$emit('input', event.detail.value);
			// 过一个生命周期再发送事件给u-form-item，否则this.$emit('input')更新了父组件的值，但是微信小程序上
			// 尚未更新到u-form-item，导致获取的值为空，从而校验混论
			this.$nextTick(() => {
				// 将当前的值发送到 u-form-item 进行校验
				this.dispatch('u-form-item', 'on-form-change', event.detail.value);
			});
		},
		/**
		 * blur 事件
		 * @param event
		 */
		handleBlur(event) {
			this.focused = false;
			// vue 原生的方法 return 出去
			this.$emit('blur', event.detail.value);
			this.$nextTick(() => {
				// 将当前的值发送到 u-form-item 进行校验
				this.dispatch('u-form-item', 'on-form-blur', event.detail.value);
			});
		},
		onFormItemError(status) {
			this.validateState = status;
		},
		onFocus(event) {
			this.focused = true;
			this.$emit('focus');
		},
		onConfirm(e) {
			this.$emit('confirm', e.detail.value);
		},
		onClear(event) {
			this.$emit('input', '');
		},
		inputClick() {
			this.$emit('click');
		}
	}
};
</script>

<style lang="scss" scoped>
.u-input {
	position: relative;
	flex: 1;
	
	&__input {
		//height: $u-form-item-height;
		font-size: 28rpx;
		color: $u-main-color;
	}
	
	&__textarea {
		width: auto;
		font-size: 28rpx;
		color: $u-main-color;
		padding: 10rpx 0;
		line-height: normal;
	}

	&--border {
		border-radius: 6rpx;
		border-radius: 4px;
		border: 1px solid $u-form-item-border-color;
	}
	
	&--error {
		border-color: $u-type-error!important;
	}
	
	&__right-icon {
		position: absolute;
		right: 20rpx;
		top: 50%;
		z-index: 3;
		transform: translateY(-50%);
		
		&__item {
			margin-left: 10rpx;
		}
		
		&--select {
			transition: transform .4s;
			
			&--reverse {
				transform: rotate(-180deg);
			}
		}
	}
}
</style>
