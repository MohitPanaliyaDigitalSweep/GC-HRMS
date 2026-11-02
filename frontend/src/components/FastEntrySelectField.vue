<template>
	<div class="flex flex-col gap-1.5 w-full">
		<span
			v-if="label"
			class="block text-sm leading-5 text-gray-700"
			:class="required ? `after:content-['_*'] after:text-red-600` : ``"
		>
			{{ label }}
		</span>
		<Input
			type="select"
			:model-value="modelValue"
			:options="options"
			:placeholder="placeholder"
			@update:model-value="(v) => $emit('update:modelValue', v)"
			@change="(v) => $emit('change', v)"
		/>
		<slot />
	</div>
</template>

<script setup>
import { Input } from "frappe-ui"

defineProps({
	label: {
		type: String,
		default: "",
	},
	required: {
		type: Boolean,
		default: false,
	},
	placeholder: {
		type: String,
		default: "",
	},
	options: {
		type: Array,
		default: () => [],
	},
	modelValue: {
		type: [String, Number],
		default: "",
	},
})

defineEmits(["update:modelValue", "change"])
</script>