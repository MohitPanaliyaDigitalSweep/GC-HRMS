<template>
	<div class="flex flex-col gap-3 w-full">
		<p class="text-base font-semibold text-gray-800">{{ __("Items") }}</p>
		<div
			v-for="(row, index) in items"
			:key="index"
			class="flex flex-col gap-1.5 w-full rounded bg-white p-3 shadow-sm"
		>
			<div class="flex flex-row items-start gap-2">
				<span
					class="item-num flex items-center rounded bg-gray-100 px-2 py-2.5 text-sm font-semibold text-gray-600"
				>
					{{ index + 1 }}
				</span>
				<div class="flex-1 w-full min-w-0">
					<Autocomplete
						size="md"
						class="w-full"
						placeholder="Search item..."
						:options="itemOptions(index)"
						@update:model-value="(opt) => onSelect(index, opt)"
						@update:query="(q) => queryItems(index, q)"
					/>
				</div>
				<button
					class="flex items-center justify-center rounded bg-gray-50 px-2.5 py-2.5 text-gray-500 hover:bg-red-50 hover:text-red-600"
					type="button"
					:title="__('Remove row')"
					@click="$emit('remove', index)"
				>
					<span class="text-base leading-none">×</span>
				</button>
			</div>

			<div
				v-if="row.itemCode"
				class="item-code-hint w-full rounded bg-gray-50 px-2 py-1 text-xs text-gray-500"
			>
				{{ row.itemCode
				}}<template v-if="row.gstRate"> | GST {{ row.gstRate }}%</template>
			</div>

			<div v-if="row.itemCode && row.hasBatchNo" class="w-full">
				<Input
					label="Batch"
					type="text"
					:model-value="row.batchNo"
					@update:model-value="(v) => { row.batchNo = v; emit('change') }"
				/>
			</div>

			<div class="grid grid-cols-4 gap-1.5 w-full">
				<Input label="Box" type="number" :model-value="row.box" @input="(v) => setFrom(row, 'box', v)" />
				<Input label="PCS" type="number" :model-value="row.pcs" @input="(v) => setFrom(row, 'pcs', v)" />
				<Input label="Ltr" type="number" :model-value="row.ltr" @input="(v) => setFrom(row, 'ltr', v)" />
				<Input label="Rate" type="number" :model-value="row.rate" @input="(v) => setFrom(row, 'rate', v)" />
			</div>

			<div class="grid grid-cols-2 gap-1.5 w-full">
				<Input
					:label="__('Qty (Ltr)')"
					type="number"
					:model-value="row.qty"
					:disabled="true"
				/>
				<Input
					:label="__('Amount')"
					type="number"
					:model-value="fmt(row.amount)"
					:disabled="true"
				/>
			</div>
		</div>

		<Button
			class="w-full rounded py-4 text-base"
			variant="subtle"
			@click="$emit('add')"
		>
			{{ __("+ Add Item") }}
		</Button>
	</div>
</template>

<script setup>
import { reactive, watch } from "vue"
import { Autocomplete, Input, Button, call, debounce } from "frappe-ui"

const props = defineProps({
	items: {
		type: Array,
		default: () => [],
	},
	company: {
		type: String,
		default: "",
	},
	searchItems: {
		type: Function,
		required: true,
	},
})

const emit = defineEmits(["add", "remove", "change"])

const pageOptions = reactive({})
const pageQuery = reactive({})
const pageLoading = reactive({})

watch(
	() => props.company,
	() => {
		cache = null
	}
)

// See FastEntrySearchSelect: replace frappe-ui's misleading "No results found"
// row with an accurate status while the search is in flight. `value` mirrors the
// current query so frappe-ui's local substring filter keeps the row.
function statusOption(index, label) {
	return { label, value: pageQuery[index] || " ", __loading: true }
}

function itemOptions(index) {
	if (pageLoading[index]) return [statusOption(index, "Searching…")]
	const opts = pageOptions[index] || []
	if (opts.length) return opts
	if (!pageQuery[index]) return [statusOption(index, "Type to search…")]
	return []
}

async function runQueryItems(index, query) {
	try {
		const data = await props.searchItems(query)
		pageOptions[index] = (data || []).map((it) => ({
			label: it.item_name || it.item_code,
			value: it.item_code,
			__raw: it,
		}))
	} catch (e) {
		console.error(e)
		pageOptions[index] = []
	} finally {
		pageLoading[index] = false
	}
}

// One debouncer per row: a shared one would let typing in row B cancel row A's
// pending fetch and leave row A stuck on the loading placeholder.
const debouncers = {}
function queryItems(index, query) {
	pageQuery[index] = query
	pageOptions[index] = []
	pageLoading[index] = true
	if (!debouncers[index]) {
		debouncers[index] = debounce((q) => runQueryItems(index, q), 300)
	}
	debouncers[index](query)
}

async function onSelect(index, option) {
	if (option && option.__loading) return
	const it = option && option.__raw
	if (!it) return
	const row = props.items[index]
	if (!row) return
	try {
		const d = await searchItemDetails(it.item_code)
		row.itemCode = (d.item_code || it.item_code)
		row.itemName = (d.item_name || it.item_name)
		row.stockUom = d.stock_uom
		row.gstRate = flt(d.gst_rate)
		row.hasBatchNo = !!d.has_batch_no
		if (!row.hasBatchNo) row.batchNo = ""

		const uoms = (it.uoms && it.uoms.length) ? it.uoms : (d.uoms || [])
		row.nosFactor = 1
		row.litreFactor = 1
		let hasBox = false
		if (uoms.length) {
			const boxUom = uoms.find((u) => u.uom === "Box")
			const nosUom = uoms.find((u) => u.uom === "Nos")
			const litreUom = uoms.find((u) => u.uom === "Litre" || u.uom === "Kg")
			hasBox = !!boxUom
			// Option B (piece-based stock UOM): pieces-per-box now lives on the
			// Box row, because Nos is the stock UOM (cf 1). Legacy box-based
			// masters still keep it on the Nos row.
			const packRow = d.stock_uom === "Nos" ? (boxUom || nosUom) : nosUom
			if (packRow) row.nosFactor = flt(packRow.conversion_factor) || 1
			if (litreUom) row.litreFactor = flt(litreUom.conversion_factor) || 0
		}
		if (!hasBox && d.stock_uom === "Nos" && row.nosFactor === 1) hasBox = true

		row.box = hasBox ? 1 : 0
		row.pcs = hasBox ? flt(row.box * row.nosFactor) : 1
		row.ltr = row.litreFactor
		if (!row.rate && d.rate) row.rate = flt(d.rate)

		calcRow(row, "box")
	} catch (e) {
		console.error(e)
	}
}

function setFrom(row, key, val) {
	row[key] = flt(val)
	if (key === "box" || key === "pcs" || key === "ltr" || key === "rate") {
		if (key === "box" && (row.box || row.box === 0)) calcRow(row, "box")
		else if (key === "pcs") calcRow(row, "pcs")
		else calcRow(row, "empty")
	}
	emit("change")
}

function calcRow(row, source) {
	const nf = row.nosFactor || 1
	if (source === "box") row.pcs = flt(row.box * nf)
	else if (source === "pcs") row.box = (row.pcs && nf) ? flt(row.pcs / nf) : 0
	row.qty = flt(row.pcs * row.ltr)
	row.amount = flt(row.pcs * row.rate)
	emit("change")
}

let cache = null
async function searchItemDetails(itemCode) {
	if (cache && cache.itemCode === itemCode) return cache
	const d = await call("fast_entry_app.api.item.get_item_details", {
		item_code: itemCode,
		warehouse: "",
		company: props.company || "",
	})
	if (d) {
		cache = { ...d, itemCode: d.item_code }
		return cache
	}
	return {}
}

const flt = (v) => {
	const n = parseFloat(v)
	return isNaN(n) ? 0 : n
}
const fmt = (v) => (parseFloat(v) || 0).toFixed(2)
</script>