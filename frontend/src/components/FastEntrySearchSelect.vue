<template>
	<div class="flex flex-col gap-1.5 w-full">
		<span
			v-if="label"
			class="block text-sm leading-5 text-gray-700"
			:class="required ? `after:content-['_*'] after:text-red-600` : ``"
		>
			{{ label }}
		</span>
		<Autocomplete
			size="md"
			:model-value="modelValue"
			:placeholder="placeholder"
			:options="shownOptions"
			:disabled="disabled"
			@update:model-value="onSelect"
			@update:query="handleQueryUpdate"
		/>
		<slot />
	</div>
</template>

<script setup>
import { Autocomplete, call, debounce } from "frappe-ui"
import { computed, ref } from "vue"

const props = defineProps({
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
	modelValue: {
		type: Object,
		default: null,
	},
	method: {
		type: String,
		required: true,
	},
	params: {
		type: Function,
		default: () => ({}),
	},
	disabled: {
		type: Boolean,
		default: false,
	},
})

const emit = defineEmits(["update:modelValue", "pick"])

const options = ref([])
const searchText = ref("")
const loading = ref(false)

// frappe-ui's Autocomplete renders a hard-coded "No results found" row whenever
// the (locally filtered) option list is empty. During the debounce + network
// window that produced a misleading "data not loaded" flash. Feed it a
// synthetic, non-selectable status row instead so the caller always sees an
// accurate state. `value` mirrors the current query so frappe-ui's local filter
// (label OR value contains query) never drops it.
function statusOption(label) {
	return { label, value: searchText.value || " ", __loading: true }
}

const shownOptions = computed(() => {
	if (loading.value) return [statusOption("Searching…")]
	if (options.value.length) return options.value
	if (!searchText.value) return [statusOption("Type to search…")]
	return []
})

let fetchSeq = 0
async function fetchOptions(query) {
	const seq = ++fetchSeq
	try {
		const extra = props.params ? props.params() : {}
		const data = await call(props.method, { ...extra, search: query })
		if (seq !== fetchSeq) return
		options.value = (data || []).map((item) => {
			const label = item.label || item.customer_name || item.item_name || item.warehouse_name || item.name
			return {
				label: label || item.name,
				value: item.value || item.name,
				__raw: item,
			}
		})
	} catch (e) {
		if (seq === fetchSeq) options.value = []
		console.error(e)
	} finally {
		if (seq === fetchSeq) loading.value = false
	}
}

const runFetch = debounce((query) => fetchOptions(query), 300)

// Runs on every keystroke (not debounced) so the loading placeholder appears
// immediately; the actual request stays debounced.
function handleQueryUpdate(query) {
	searchText.value = query
	options.value = []
	loading.value = true
	runFetch(query)
}

function onSelect(option) {
	if (option && option.__loading) return
	emit("update:modelValue", option)
	emit("pick", option)
}

defineExpose({ fetchOptions })
</script>