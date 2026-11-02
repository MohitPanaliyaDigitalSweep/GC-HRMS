<template>
	<div class="flex flex-row justify-between items-center w-full">
		<span class="text-sm leading-5 text-gray-600">{{ label }}</span>
		<span
			class="text-sm leading-5 font-medium text-gray-900"
			:class="{ 'text-red-600': amount < 0 }"
		>
			{{ prefix }}{{ formatMoney(amount) }}
		</span>
	</div>
</template>

<script setup>
const props = defineProps({
	label: {
		type: String,
		required: true,
	},
	amount: {
		type: Number,
		default: 0,
	},
	prefix: {
		type: String,
		default: "",
	},
})

function formatMoney(v) {
	return (parseFloat(v) || 0).toFixed(2)
}
</script>