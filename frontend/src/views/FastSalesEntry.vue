<template>
	<BaseLayout :pageTitle="__('Fast Sales Entry')">
		<template #body>
			<div class="fe-pwa-sales">
				<div class="wrap">
					<div class="hint-note">
						{{ __("Invoices are always created as") }}
						<b>{{ __("Draft") }}</b>
						{{ __("(not submitted) for later review & approval.") }}
					</div>

					<div class="card">
						<div class="card-title">{{ __("Party") }}</div>
						<div class="field">
							<label for="fe-company">
								{{ __("Company") }} <span class="reqd">*</span>
							</label>
							<select id="fe-company" v-model="company" @change="onCompanyChange">
								<option value="">{{ __("Select Company") }}</option>
								<option v-for="c in companyOptions" :key="c.value" :value="c.value">
									{{ c.label }}
								</option>
							</select>
						</div>
						<div class="field">
							<label for="fe-customer">
								{{ __("Customer") }} <span class="reqd">*</span>
							</label>
							<div class="rel">
								<input
									id="fe-customer"
									type="text"
									v-model="cust.q"
									:placeholder="__('Type to search...')"
									autocomplete="off"
									@input="onCustInput"
									@keydown="onCustKey"
									@focus="onCustFocus"
									@blur="onCustBlur"
								/>
								<div v-if="cust.open && cust.opts.length" class="dropdown">
									<div
										v-for="(o, i) in cust.opts"
										:key="i"
										class="dropdown-item"
										:class="{ active: i === cust.active }"
										@mousedown.prevent="pickCustomer(o)"
									>
										<span class="dd-primary">{{ o.primary }}</span>
										<span class="dd-secondary">{{ o.secondary }}</span>
									</div>
								</div>
							</div>
							<span
								v-if="customerBalance !== null"
								class="balance-tag"
								:class="customerBalanceUnpaid ? 'bal-unpaid' : 'bal-advance'"
							>
								{{
									(customerBalanceUnpaid ? __("Unpaid: ") : __("Advance: ")) +
									"Rs " +
									customerBalance.toFixed(2)
								}}
							</span>
						</div>
					</div>

					<div class="card">
						<div class="card-title">{{ __("Invoice") }}</div>
						<div class="row-3">
							<div class="field">
								<label for="fe-date">
									{{ __("Date") }} <span class="reqd">*</span>
								</label>
								<input id="fe-date" type="date" v-model="invoiceDate" />
							</div>
							<div class="field">
								<label for="fe-gst">{{ __("GST Type") }}</label>
								<select id="fe-gst" v-model="gstType" @change="calcTotals">
									<option value="intra">{{ __("Intra (CGST+SGST)") }}</option>
									<option value="inter">{{ __("Inter (IGST)") }}</option>
								</select>
							</div>
							<div class="field">
								<label for="fe-tax-override">{{ __("Tax Override") }}</label>
								<select id="fe-tax-override" v-model="taxOverride" @change="calcTotals">
									<option value="">{{ __("Per Item") }}</option>
									<option value="5">5%</option>
									<option value="12">12%</option>
									<option value="18">18%</option>
									<option value="28">28%</option>
								</select>
							</div>
							<div class="field" style="display:flex;align-items:center;gap:6px;">
								<label style="display:flex;align-items:center;gap:4px;margin:0;">
									<input type="checkbox" v-model="applyGst" @change="calcTotals"> {{ __("Apply GST") }}
								</label>
							</div>
						</div>
						<div class="field">
							<label for="fe-warehouse">{{ __("Warehouse") }}</label>
							<div class="rel">
								<input
									id="fe-warehouse"
									type="text"
									v-model="wh.q"
									:placeholder="__('Default warehouse')"
									autocomplete="off"
									@input="onWhInput"
									@keydown="onWhKey"
									@focus="onWhFocus"
									@blur="onWhBlur"
								/>
								<div v-if="wh.open && wh.opts.length" class="dropdown">
									<div
										v-for="(o, i) in wh.opts"
										:key="i"
										class="dropdown-item"
										:class="{ active: i === wh.active }"
										@mousedown.prevent="pickWarehouse(o)"
									>
										<span class="dd-primary">{{ o.primary }}</span>
										<span class="dd-secondary">{{ o.secondary }}</span>
									</div>
								</div>
							</div>
						</div>
						<div class="row-2">
							<div class="field check-field">
								<input type="checkbox" id="fe-stock-impact" v-model="updateStock" />
								<label for="fe-stock-impact">{{ __("Update Stock") }}</label>
							</div>
							<div class="field">
								<label for="fe-distance">{{ __("Distance (km)") }}</label>
								<input
									id="fe-distance"
									type="number"
									v-model="distance"
									placeholder="0"
									min="0"
								/>
							</div>
						</div>
					</div>

					<div class="card">
						<div class="card-title">{{ __("Dispatch") }}</div>
						<div class="row-2">
							<div class="field">
								<label for="fe-driver">{{ __("Delivery Person") }}</label>
								<select id="fe-driver" v-model="driver">
									<option value="">{{ __("Select") }}</option>
									<option v-for="d in driverOptions" :key="d.value" :value="d.value">
										{{ d.label }}
									</option>
								</select>
							</div>
							<div class="field">
								<label for="fe-vehicle">{{ __("Vehicle") }}</label>
								<select id="fe-vehicle" v-model="vehicle">
									<option value="">{{ __("Select") }}</option>
									<option v-for="v in vehicleOptions" :key="v.value" :value="v.value">
										{{ v.label }}
									</option>
								</select>
							</div>
						</div>
						<div class="field">
							<label for="fe-sales-person">{{ __("Sales Person") }}</label>
							<select id="fe-sales-person" v-model="salesPerson">
								<option value="">{{ __("Select") }}</option>
								<option v-for="s in salesPersonOptions" :key="s.value" :value="s.value">
									{{ s.label }}
								</option>
							</select>
						</div>
					</div>

					<div class="card">
						<div class="card-title">{{ __("Items") }}</div>
						<div id="fe-items">
							<div v-for="(row, index) in items" :key="row.id" class="item-card">
								<div class="item-head">
									<div class="item-num">{{ index + 1 }}</div>
									<div class="item-input-wrap">
										<input
											type="text"
											class="item-inp"
											v-model="row.q"
											:placeholder="__('Search item...')"
											autocomplete="off"
											@input="onItemInput(row)"
											@keydown="onItemKey(row, $event)"
											@focus="onItemFocus(row)"
											@blur="onItemBlur(row)"
										/>
										<div v-if="row.open && row.opts.length" class="dropdown">
											<div
												v-for="(o, i) in row.opts"
												:key="i"
												class="dropdown-item"
												:class="{ active: i === row.active }"
												@mousedown.prevent="pickItem(row, o)"
											>
												<span class="dd-primary">{{ o.primary }}</span>
												<span class="dd-secondary">{{ o.secondary }}</span>
											</div>
										</div>
										<div v-if="row.itemCode" class="item-code-hint">
											{{ row.itemCode
											}}<template v-if="row.gstRate"> | GST {{ row.gstRate }}%</template>
										</div>
									</div>
									<button type="button" class="del-item" @click="removeItemRow(index)">
										&times;
									</button>
								</div>
								<div class="row-3">
									<div class="field">
										<label :for="'fe-item-' + row.id + '-box'">{{ __("Box") }}</label>
										<input
											:id="'fe-item-' + row.id + '-box'"
											type="number"
											:value="row.box"
											min="0"
											step="1"
											@input="setFrom(row, 'box', $event.target.value)"
										/>
									</div>
									<div class="field">
										<label :for="'fe-item-' + row.id + '-pcs'">{{ __("PCS") }}</label>
										<input
											:id="'fe-item-' + row.id + '-pcs'"
											type="number"
											:value="row.pcs"
											min="0"
											@input="setFrom(row, 'pcs', $event.target.value)"
										/>
									</div>
									<div class="field">
										<label :for="'fe-item-' + row.id + '-ltr'">{{ __("LTR") }}</label>
										<input
											:id="'fe-item-' + row.id + '-ltr'"
											type="number"
											:value="row.ltr"
											min="0"
											step="0.01"
											@input="setFrom(row, 'ltr', $event.target.value)"
										/>
									</div>
								</div>
								<div class="row-3">
									<div class="field">
										<label :for="'fe-item-' + row.id + '-qty'">{{ __("Total Litre") }}</label>
										<input
											:id="'fe-item-' + row.id + '-qty'"
											type="number"
											:value="row.qty"
											readonly
										/>
									</div>
									<div class="field">
										<label :for="'fe-item-' + row.id + '-rate'">{{ __("Rate") }}</label>
										<input
											:id="'fe-item-' + row.id + '-rate'"
											type="number"
											:value="row.rate"
											min="0"
											step="0.01"
											@input="setFrom(row, 'rate', $event.target.value)"
										/>
									</div>
									<div class="field">
										<label :for="'fe-item-' + row.id + '-amount'">{{ __("Amount") }}</label>
										<input
											:id="'fe-item-' + row.id + '-amount'"
											type="number"
											:value="fmt(row.amount)"
											readonly
										/>
									</div>
								</div>
							</div>
						</div>
						<button type="button" class="add-item-btn" @click="addItemRow()">
							{{ __("+ Add Item") }}
						</button>
					</div>

					<div class="card">
						<div class="card-title">{{ __("Discounts & Totals") }}</div>
						<div class="row-3">
							<div class="field">
								<label for="fe-disc-pct">{{ __("Disc %") }}</label>
								<input
									id="fe-disc-pct"
									type="number"
									v-model="discPct"
									min="0"
									step="any"
									@input="onDiscPct"
								/>
							</div>
							<div class="field">
								<label for="fe-disc-amt">{{ __("Disc Amt") }}</label>
								<input
									id="fe-disc-amt"
									type="number"
									v-model="discAmt"
									min="0"
									step="any"
									@input="onDiscAmt"
								/>
							</div>
							<div class="field">
								<label for="fe-disc-on">{{ __("Disc On") }}</label>
								<select id="fe-disc-on" v-model="discOn" @change="calcTotals">
									<option value="Net Total">{{ __("Net Total") }}</option>
									<option value="Grand Total">{{ __("Grand Total") }}</option>
								</select>
							</div>
							<div class="field">
								<label for="fe-freight">{{ __("Freight") }}</label>
								<input
									id="fe-freight"
									type="number"
									v-model="freight"
									min="0"
									step="any"
									@input="calcTotals"
								/>
							</div>
						</div>
						<div v-if="discWarn" class="disc-warn">
							{{ __("Discount exceeds Sub Total — ERPNext will reject this bill on save.") }}
						</div>
						<div class="totals">
							<div class="t-row">
								<span>{{ __("Sub Total") }}</span><span>{{ formatMoney(subTotal) }}</span>
							</div>
							<div class="t-row">
								<span>{{ __("Discount") }}</span
								><span>-{{ formatMoney(discountFinal) }}</span>
							</div>
							<div class="t-row">
								<span>{{ __("Freight") }}</span
								><span>+{{ formatMoney(freightOnly) }}</span>
							</div>
							<div v-if="gstType === 'intra'" class="t-tax">
								<span>{{ __("CGST") }}</span><span>{{ formatMoney(cgst) }}</span>
							</div>
							<div v-if="gstType === 'intra'" class="t-tax">
								<span>{{ __("SGST") }}</span><span>{{ formatMoney(sgst) }}</span>
							</div>
							<div v-if="gstType === 'inter'" class="t-tax">
								<span>{{ __("IGST") }}</span><span>{{ formatMoney(igst) }}</span>
							</div>
							<div class="t-row">
								<span>{{ __("Round Off") }}</span><span>{{ formatMoney(roundOff) }}</span>
							</div>
							<div class="t-row total">
								<span>{{ __("Grand Total") }}</span
								><span>{{ formatMoney(grandTotal) }}</span>
							</div>
						</div>
					</div>
				</div>

				<div class="bottom-bar">
					<div class="bottom-inner">
						<button type="button" class="btn btn-secondary" :disabled="saving" @click="doSave(true)">
							{{ __("Save & New") }}
						</button>
						<button type="button" class="btn btn-primary" :disabled="saving" @click="doSave(false)">
							{{ saving ? __("Saving...") : __("Save as Draft") }}
						</button>
					</div>
				</div>
			</div>
		</template>
	</BaseLayout>
</template>

<script setup>
import { onMounted, ref, reactive, computed, inject } from "vue"
import { toast, call, debounce } from "frappe-ui"

import BaseLayout from "@/components/BaseLayout.vue"

const __ = inject("$translate")

const company = ref("")
const companyOptions = ref([])
const cust = reactive({ q: "", opts: [], open: false, active: 0, loading: false, picked: "" })
const wh = reactive({ q: "", opts: [], open: false, active: 0, loading: false, picked: "" })
const customerBalance = ref(null)
const customerBalanceUnpaid = ref(false)
const invoiceDate = ref("")
const gstType = ref("intra")
const taxOverride = ref("")
const updateStock = ref(true)
const distance = ref("")
const driver = ref("")
const vehicle = ref("")
const salesPerson = ref("")
const items = ref([])
const discPct = ref(0)
const discAmt = ref(0)
const discOn = ref("Net Total")
const applyGst = ref(true)
const freight = ref(0)
const saving = ref(false)

const driverOptions = ref([])
const vehicleOptions = ref([])
const salesPersonOptions = ref([])
const companyTaxRates = ref({})

let rowSeq = 0
let currentDefaultSalesPerson = ""

const flt = (v) => {
	let n = parseFloat(v)
	return isNaN(n) ? 0 : n
}
const formatMoney = (v) => (parseFloat(v) || 0).toFixed(2)
const fmt = formatMoney

// ---------------------------------------------------------------------------
// Search dropdowns (mirrors the standalone /sales_pwa page)
// ---------------------------------------------------------------------------
const custDeb = debounce((q) => runCustQuery(q), 300)
const whDeb = debounce((q) => runWhQuery(q), 300)
const itemDeb = {}

function onCustInput() {
	cust.picked = ""
	customerBalance.value = null
	customerBalanceUnpaid.value = false
	const q = (cust.q || "").trim()
	cust.opts = []
	cust.active = 0
	cust.open = false
	if (!q) return
	cust.loading = true
	custDeb(q)
}
async function runCustQuery(q) {
	try {
		const list = await call("fast_entry_app.api.party.search_customers", {
			search: q,
			company: company.value,
			limit: 12,
		})
		cust.opts = (list || []).map((c) => ({
			primary: c.customer_name || c.name,
			secondary: c.name,
			value: c.name,
		}))
		cust.open = cust.opts.length > 0
		cust.active = 0
	} catch (e) {
		cust.opts = []
	} finally {
		cust.loading = false
	}
}
function pickCustomer(o) {
	cust.q = o.primary
	cust.picked = o.value
	cust.open = false
	onCustomerPicked(o.value)
}
function onCustFocus() {
	if (cust.opts.length) cust.open = true
}
function onCustBlur() {
	setTimeout(() => (cust.open = false), 150)
}
function onCustKey(e) {
	handleNav(e, cust)
}

function onWhInput() {
	wh.picked = ""
	const q = (wh.q || "").trim()
	wh.opts = []
	wh.active = 0
	wh.open = false
	if (!q) return
	wh.loading = true
	whDeb(q)
}
async function runWhQuery(q) {
	try {
		const list = await call("fast_entry_app.api.warehouse.search_warehouses", {
			search: q,
			company: company.value,
			limit: 10,
		})
		wh.opts = (list || []).map((w) => ({
			primary: w.warehouse_name || w.name,
			secondary: w.name,
			value: w.name,
		}))
		wh.open = wh.opts.length > 0
		wh.active = 0
	} catch (e) {
		wh.opts = []
	} finally {
		wh.loading = false
	}
}
function pickWarehouse(o) {
	wh.q = o.primary
	wh.picked = o.value
	wh.open = false
}
function onWhFocus() {
	if (wh.opts.length) wh.open = true
}
function onWhBlur() {
	setTimeout(() => (wh.open = false), 150)
}
function onWhKey(e) {
	handleNav(e, wh)
}

function onItemInput(row) {
	row.itemCode = ""
	const q = (row.q || "").trim()
	row.opts = []
	row.active = 0
	row.open = false
	if (!q) return
	if (!itemDeb[row.id]) itemDeb[row.id] = debounce((query) => runItemQuery(row, query), 300)
	itemDeb[row.id](q)
}
async function runItemQuery(row, q) {
	try {
		const list = await call("fast_entry_app.api.item.search_items", {
			search: q,
			company: company.value,
			limit: 12,
		})
		row.opts = (list || []).map((it) => {
			let hint = ""
			if (it.uoms && it.uoms.length > 1) {
				const alt = it.uoms.find((u) => flt(u.conversion_factor) !== 1)
				hint = alt ? alt.uom : ""
			}
			return {
				primary: it.item_name || it.item_code,
				secondary: it.item_code + (hint ? ` (${hint})` : ""),
				value: it.item_code,
				__raw: it,
			}
		})
		row.open = row.opts.length > 0
		row.active = 0
	} catch (e) {
		row.opts = []
	}
}
function pickItem(row, o) {
	row.q = o.primary
	row.itemCode = o.value
	row.open = false
	onItemSelected(row, o.value)
}
function onItemFocus(row) {
	if (row.opts.length) row.open = true
}
function onItemBlur(row) {
	setTimeout(() => (row.open = false), 150)
}
function onItemKey(row, e) {
	handleNav(e, row)
}

// Shared keyboard navigation for the custom dropdowns.
function handleNav(e, state) {
	const opts = state.opts || []
	if (!state.open || !opts.length) {
		if (e.key === "Escape") state.open = false
		return
	}
	if (e.key === "ArrowDown") {
		e.preventDefault()
		state.active = (state.active + 1) % opts.length
	} else if (e.key === "ArrowUp") {
		e.preventDefault()
		state.active = (state.active - 1 + opts.length) % opts.length
	} else if (e.key === "Enter") {
		if (opts[state.active]) {
			e.preventDefault()
			const o = opts[state.active]
			if (state === cust) pickCustomer(o)
			else if (state === wh) pickWarehouse(o)
			else pickItem(state, o)
		}
	} else if (e.key === "Escape") {
		state.open = false
	}
}

async function loadContext() {
	try {
		const ctx = await call("fast_entry_app.api.sales.fast_entry_context", {})
		const companies = ctx.companies || []
		companyOptions.value = companies.map((c) => ({ label: c, value: c }))
		companyTaxRates.value = ctx.company_tax_rates || {}
		if (companies.length) {
			let def = companies[0]
			const last = localStorage.getItem("fe_pwa_last_company")
			if (last && companies.includes(last)) def = last
			company.value = def
		}
		if (ctx.default_sales_person) {
			currentDefaultSalesPerson = ctx.default_sales_person
		}
	} catch (e) {
		console.error(e)
	}
	loadDispatch()
}

function onCompanyChange() {
	localStorage.setItem("fe_pwa_last_company", company.value)
	cust.picked = ""
	customerBalance.value = null
	customerBalanceUnpaid.value = false
	wh.picked = ""
}

async function onCustomerPicked(name) {
	try {
		const d = await call("fast_entry_app.api.party.get_customer_details", {
			customer: name,
			company: company.value,
		})
		if (d) {
			const bal = flt(d.outstanding)
			customerBalance.value = bal
			customerBalanceUnpaid.value = d.balance_type === "Dr"
			if (d.gst_type) gstType.value = d.gst_type
			calcTotals()
		}
	} catch (e) {
		console.error(e)
	}
}

function addItemRow(itemPatch) {
	rowSeq += 1
	items.value.push(
		Object.assign(
			{
				id: rowSeq,
				q: "",
				opts: [],
				open: false,
				active: 0,
				itemCode: "",
				itemName: "",
				stockUom: "Nos",
				gstRate: 0,
				box: 0,
				pcs: 0,
				ltr: 0,
				qty: 0,
				rate: 0,
				amount: 0,
				nosFactor: 1,
				litreFactor: 0,
				warehouse: wh.picked || "",
				batchNo: "",
				hasBatchNo: false,
			},
			itemPatch || {}
		)
	)
}

function removeItemRow(index) {
	items.value.splice(index, 1)
	if (!items.value.length) addItemRow()
	onItemsChanged()
}

function onItemsChanged() {
	calcTotals()
}

async function onItemSelected(row, itemCode) {
	try {
		const d = await call("fast_entry_app.api.item.get_item_details", {
			item_code: itemCode || row.itemCode,
			warehouse: wh.picked || "",
			company: company.value || "",
		})
		if (!d) return
		applyItemDetails(row, d)
	} catch (e) {
		console.error(e)
	}
}

function applyItemDetails(row, d) {
	row.itemCode = d.item_code || row.itemCode
	row.itemName = d.item_name || row.itemName
	row.stockUom = d.stock_uom
	row.gstRate = flt(d.gst_rate)

	const uoms = d.uoms || []
	row.nosFactor = 1
	row.litreFactor = 1
	let hasBox = false
	if (uoms.length) {
		const boxUom = uoms.find((u) => u.uom === "Box")
		const nosUom = uoms.find((u) => u.uom === "Nos")
		const litreUom = uoms.find((u) => u.uom === "Litre" || u.uom === "Kg")
		hasBox = !!boxUom
		// Option B (piece-based stock UOM): pieces-per-box now lives on the Box
		// row, because Nos is the stock UOM (cf 1). Legacy box-based masters
		// still keep it on the Nos row.
		const packRow = d.stock_uom === "Nos" ? boxUom || nosUom : nosUom
		if (packRow) row.nosFactor = flt(packRow.conversion_factor) || 1
		if (litreUom) row.litreFactor = flt(litreUom.conversion_factor) || 0
	}
	if (!hasBox && d.stock_uom === "Nos" && row.nosFactor === 1) hasBox = true

	row.box = hasBox ? 1 : 0
	row.pcs = hasBox ? flt(row.box * row.nosFactor) : 1
	row.ltr = row.litreFactor
	if (!row.warehouse && wh.picked) row.warehouse = wh.picked
	else if (d.warehouse) row.warehouse = d.warehouse
	if (!row.rate && d.rate) row.rate = flt(d.rate)

	row.q = row.itemName
	calcRow(row, "box")
}

function setFrom(row, key, val) {
	row[key] = flt(val)
	if (key === "box") calcRow(row, "box")
	else if (key === "pcs") calcRow(row, "pcs")
	else calcRow(row, "empty")
	onItemsChanged()
}

function calcRow(row, source) {
	const nf = row.nosFactor || 1
	if (source === "box") row.pcs = flt(row.box * nf)
	else if (source === "pcs") row.box = row.pcs && nf ? flt(row.pcs / nf) : 0
	row.qty = flt(row.pcs * row.ltr)
	row.amount = flt(row.pcs * row.rate)
	onItemsChanged()
}

function onDiscPct() {
	const sub = subTotal.value
	if (sub > 0) discAmt.value = ((sub * flt(discPct.value)) / 100).toFixed(2)
	calcTotals()
}

function onDiscAmt() {
	const sub = subTotal.value
	if (sub > 0) discPct.value = ((flt(discAmt.value) / sub) * 100).toFixed(2)
	calcTotals()
}

const subTotal = computed(() => {
	return (items.value || [])
		.filter((r) => r.itemCode)
		.reduce((s, r) => s + flt(r.amount), 0)
})

const discountFinal = computed(() => flt(discAmt.value))

const freightOnly = computed(() => flt(freight.value))

// ERPNext-exact totals, mode-aware (Scenario A / Scenario B):
//   Net Total   -> discount reduces the taxable value first
//   Grand Total -> post-tax discount; GST stays on the full value
// GST is charged on (taxable + freight); grand = (sub - D) + freight + GST.
// With freight the server points the GST rows at the freight row
// ("On Previous Row Total"), so GST covers taxable base + freight.
const effectiveRate = computed(() => {
	if (!applyGst.value) return 0
	const ovr = flt(taxOverride.value)
	if (ovr) return ovr
	for (const it of items.value) {
		if (flt(it.gstRate) > 0) return flt(it.gstRate)
	}
	return flt(companyTaxRates.value[company.value]) || 0
})

const discBase = computed(() => subTotal.value - discountFinal.value)

const taxable = computed(() =>
	discOn.value === "Grand Total" ? subTotal.value : discBase.value,
)

// Grand Total rejects discount reaching the distribution base sub * (1 + rate);
// Net Total allows 100% free invoices.
const discWarn = computed(() => {
	const sub = flt(subTotal.value)
	return discOn.value === "Grand Total"
		? sub > 0 && flt(discAmt.value) >= sub * (1 + effectiveRate.value / 100)
		: flt(discAmt.value) > sub
})

const mainBase = computed(() => taxable.value + (freightOnly.value > 0 ? freightOnly.value : 0))

const mainTax = computed(() => (mainBase.value * effectiveRate.value) / 100)

const cgst = computed(() => {
	if (!effectiveRate.value || gstType.value !== "intra") return 0
	return mainTax.value / 2
})

const sgst = computed(() => {
	if (!effectiveRate.value || gstType.value !== "intra") return 0
	return mainTax.value / 2
})

const igst = computed(() => {
	if (!effectiveRate.value || gstType.value !== "inter") return 0
	return mainTax.value
})

const raw = computed(() => discBase.value + freightOnly.value + mainTax.value)

const roundOff = computed(() => Math.round(raw.value) - raw.value)

const grandTotal = computed(() => Math.round(raw.value))

function calcTotals() {
	// vue reactives handle it
}

function validate() {
	const errs = []
	if (!company.value) errs.push(__("Company is required."))
	if (!cust.picked) errs.push(__("Customer is required."))
	if (!invoiceDate.value) errs.push(__("Invoice Date is required."))
	if (updateStock.value && !wh.picked)
		errs.push(__("Warehouse is required for stock impact."))
	const valid = (items.value || []).filter((r) => r.itemCode && flt(r.pcs) > 0)
	if (!valid.length) errs.push(__("At least one item with PCS > 0 is required."))
	for (const r of valid) {
		if (flt(r.rate) <= 0) errs.push(__("Item {0}: rate required.", [r.itemCode]))
	}
	return errs
}

function getPayload() {
	return {
		company: company.value,
		customer: cust.picked,
		invoice_date: invoiceDate.value,
		posting_date: invoiceDate.value,
		gst_type: gstType.value,
		tax_override: flt(taxOverride.value),
		apply_gst: applyGst.value ? 1 : 0,
		warehouse: wh.picked,
		update_stock: updateStock.value ? 1 : 0,
		auto_einvoice: 0,
		auto_ewaybill: 0,
		delivery_person: driver.value,
		delivery_vehicle: vehicle.value,
		distance: flt(distance.value),
		sales_person: salesPerson.value,
		discount: flt(discAmt.value),
		discount_pct: flt(discPct.value),
		apply_discount_on: discOn.value || "Net Total",
		freight: flt(freight.value),
		freight_gst: 0,
		draft: 1,
		items: (items.value || [])
			.filter((r) => r.itemCode)
			.map((r) => ({
				item_code: r.itemCode,
				item_name: r.itemName,
				box: r.box,
				pcs: r.pcs,
				ltr: r.ltr,
				qty: r.qty,
				rate: r.rate,
				amount: r.amount,
				uom: "Nos",
				stock_uom: r.stockUom,
				nos_factor: r.nosFactor || 1,
				litre_factor: r.litreFactor || 0,
				conversion_factor: r.ltr ? flt(1 / r.ltr) : 1,
				total_ltr: r.qty || 0,
				warehouse: r.warehouse || wh.picked,
				gst_rate: r.gstRate || 0,
				batch_no: r.batchNo || "",
			})),
	}
}

async function doSave(afterSave) {
	const errs = validate()
	if (errs.length) {
		toast({ title: errs.join("\n"), variant: "error" })
		return
	}
	saving.value = true
	try {
		const msg = await call("fast_entry_app.api.sales.create_sales_invoice", {
			data: getPayload(),
		})
		if (msg && msg.name) {
			toast({
				title: `Draft saved: ${msg.name}`,
				action: {
					label: "Open",
					onClick: () => window.open(`/app/sales-invoice/${msg.name}`, "_blank"),
				},
			})
			if (afterSave) clearForm()
		}
	} catch (e) {
		// frappe-ui `call` puts the generic "method ExcType" wrapper in
		// e.message but the actionable server text (parsed _server_messages)
		// in e.messages — surface that first so users see the real cause.
		const detail =
			(e && Array.isArray(e.messages) && e.messages.filter(Boolean).join("\n")) ||
			String(e.message || e)
		toast({ title: detail.slice(0, 300), variant: "error" })
		console.error(e)
	} finally {
		saving.value = false
	}
}

function clearForm() {
	items.value = []
	cust.q = ""
	cust.picked = ""
	cust.opts = []
	customerBalance.value = null
	wh.q = ""
	wh.picked = ""
	wh.opts = []
	invoiceDate.value = new Date().toISOString().slice(0, 10)
	taxOverride.value = ""
	applyGst.value = true
	discPct.value = 0
	discAmt.value = 0
	discOn.value = "Net Total"
	freight.value = 0
	distance.value = ""
	updateStock.value = true
	addItemRow()
}

async function loadDispatch() {
	try {
		const drivers = await call("fast_entry_app.api.sales.search_drivers", {
			search: "",
			limit: 100,
		})
		driverOptions.value = (drivers || []).map((d) => ({ label: d.label, value: d.label }))
	} catch (e) {
		console.error(e)
	}
	try {
		const vehicles = await call("fast_entry_app.api.sales.search_vehicles", {
			search: "",
			limit: 100,
		})
		vehicleOptions.value = (vehicles || []).map((v) => ({
			label: v.label + (v.make ? ` (${v.make})` : ""),
			value: v.label,
		}))
	} catch (e) {
		console.error(e)
	}
	try {
		const persons = await call("fast_entry_app.api.sales.search_sales_persons", {
			search: "",
			limit: 100,
		})
		salesPersonOptions.value = (persons || []).map((s) => ({
			label: s.label,
			value: s.label,
		}))
		if (currentDefaultSalesPerson) {
			const found = (salesPersonOptions.value || []).find(
				(s) => s.value === currentDefaultSalesPerson
			)
			if (found) salesPerson.value = currentDefaultSalesPerson
		}
	} catch (e) {
		console.error(e)
	}
}

onMounted(() => {
	invoiceDate.value = new Date().toISOString().slice(0, 10)
	loadContext()
	addItemRow()
})
</script>

<style scoped>
.fe-pwa-sales {
	--primary: #007be0;
	--primary-tint: #c8e6ff;
	--bg: #f8f8f8;
	--card: #ffffff;
	--border: #e2e2e2;
	--text: #171717;
	--text-2: #525252;
	--text-3: #999999;
	background: var(--bg);
	color: var(--text);
	font-family: -apple-system, BlinkMacSystemFont, "Inter", "Segoe UI", Roboto, sans-serif;
	font-size: 14px;
	line-height: 1.45;
	-webkit-font-smoothing: antialiased;
	min-height: 100%;
}
.fe-pwa-sales * {
	box-sizing: border-box;
}
.wrap {
	max-width: 520px;
	margin: 0 auto;
	padding: 12px;
}
.hint-note {
	background: #fff7ed;
	border: 1px solid #fed7aa;
	color: #9a3412;
	font-size: 12px;
	padding: 8px 10px;
	border-radius: 8px;
	margin-bottom: 12px;
}
.card {
	background: var(--card);
	border: 1px solid var(--border);
	border-radius: 10px;
	padding: 14px;
	margin-bottom: 12px;
}
.card-title {
	font-size: 11px;
	font-weight: 700;
	text-transform: uppercase;
	letter-spacing: 0.05em;
	color: var(--text-3);
	margin-bottom: 10px;
}
.field {
	margin-bottom: 10px;
}
.field:last-child {
	margin-bottom: 0;
}
.rel {
	position: relative;
}
label {
	display: block;
	font-size: 12px;
	font-weight: 600;
	color: var(--text-2);
	margin-bottom: 4px;
}
label .reqd {
	color: #ef4444;
}
.fe-pwa-sales input[type="text"],
.fe-pwa-sales input[type="number"],
.fe-pwa-sales input[type="date"],
.fe-pwa-sales select {
	width: 100%;
	padding: 9px 10px;
	font-size: 14px;
	color: var(--text);
	border: 1px solid #c7c7c7;
	border-radius: 7px;
	background: #fff;
	outline: none;
	-webkit-appearance: none;
	appearance: none;
}
.fe-pwa-sales input:focus,
.fe-pwa-sales select:focus {
	border-color: var(--primary);
	box-shadow: 0 0 0 2px #dff0ff;
}
.fe-pwa-sales input[readonly] {
	background: #f6f6f6;
	color: var(--text-2);
}
.fe-pwa-sales input[type="number"]::-webkit-inner-spin-button,
.fe-pwa-sales input[type="number"]::-webkit-outer-spin-button,
.fe-pwa-sales input[type="number"]:hover::-webkit-inner-spin-button,
.fe-pwa-sales input[type="number"]:focus::-webkit-inner-spin-button,
.fe-pwa-sales input[type="number"]:hover::-webkit-outer-spin-button,
.fe-pwa-sales input[type="number"]:focus::-webkit-outer-spin-button {
	-webkit-appearance: none;
	margin: 0;
}
.fe-pwa-sales input[type="number"] {
	-moz-appearance: textfield;
	appearance: textfield;
}
.row-2 {
	display: grid;
	grid-template-columns: 1fr 1fr;
	gap: 10px;
}
.row-3 {
	display: grid;
	grid-template-columns: 1fr 1fr 1fr;
	gap: 10px;
}
.check-field {
	display: flex;
	align-items: center;
	gap: 8px;
}
.check-field input[type="checkbox"] {
	width: 17px;
	height: 17px;
	accent-color: var(--primary);
}
.check-field label {
	margin: 0;
	font-weight: 500;
}
.balance-tag {
	display: inline-block;
	margin-top: 5px;
	font-size: 11px;
	font-weight: 600;
	padding: 2px 8px;
	border-radius: 999px;
}
.bal-unpaid {
	background: #fef2f2;
	color: #b91c1c;
	border: 1px solid #fdc2c2;
}
.bal-advance {
	background: #ecfdf5;
	color: #047857;
	border: 1px solid #a7f3d0;
}
.item-card {
	border: 1px solid var(--border);
	border-radius: 10px;
	padding: 10px;
	margin-bottom: 10px;
	background: #fff;
}
.item-head {
	display: flex;
	gap: 8px;
	margin-bottom: 8px;
	align-items: flex-start;
}
.item-num {
	font-size: 12px;
	font-weight: 700;
	color: var(--text-3);
	padding-top: 9px;
	min-width: 20px;
}
.item-input-wrap {
	flex: 1;
	position: relative;
}
.item-code-hint {
	font-size: 11px;
	color: var(--text-3);
	margin-top: 3px;
}
.del-item {
	background: none;
	border: none;
	color: var(--text-3);
	font-size: 18px;
	padding: 6px 4px;
	cursor: pointer;
}
.del-item:hover {
	color: #ef4444;
}
.add-item-btn {
	width: 100%;
	padding: 10px;
	font-weight: 600;
	font-size: 13px;
	background: #fff;
	color: var(--primary);
	border: 1px dashed var(--primary);
	border-radius: 9px;
	cursor: pointer;
}
.add-item-btn:active {
	background: var(--primary-tint);
}
.dropdown {
	position: absolute;
	top: calc(100% + 3px);
	left: 0;
	right: 0;
	z-index: 80;
	background: #fff;
	border: 1px solid var(--border);
	border-radius: 8px;
	box-shadow: 0 6px 20px rgba(0, 0, 0, 0.14);
	max-height: 260px;
	overflow-y: auto;
}
.dropdown-item {
	padding: 9px 12px;
	cursor: pointer;
	border-bottom: 1px solid #f3f3f3;
}
.dropdown-item:last-child {
	border-bottom: none;
}
.dropdown-item.active {
	background: var(--primary-tint);
}
.dd-primary {
	display: block;
	font-size: 13px;
	font-weight: 550;
}
.dd-secondary {
	display: block;
	font-size: 11px;
	color: var(--text-3);
}
.disc-warn {
	color: #e74c3c;
	font-size: 11px;
	font-weight: 600;
	margin-bottom: 8px;
}
.totals {
	display: flex;
	flex-direction: column;
	gap: 7px;
}
.t-row {
	display: flex;
	justify-content: space-between;
	font-size: 13px;
	color: var(--text-2);
}
.t-row.total {
	font-size: 16px;
	font-weight: 700;
	color: var(--text);
	border-top: 1px solid var(--border);
	padding-top: 8px;
}
.t-tax {
	display: flex;
	justify-content: space-between;
	font-size: 12px;
	color: var(--text-2);
}
.bottom-bar {
	position: sticky;
	bottom: 0;
	z-index: 50;
	width: 100%;
	background: #fff;
	border-top: 1px solid var(--border);
	padding: 12px 16px calc(12px + env(safe-area-inset-bottom));
}
.bottom-inner {
	max-width: 520px;
	margin: 0 auto;
	display: flex;
	gap: 10px;
}
.btn {
	border: none;
	border-radius: 8px;
	padding: 13px 16px;
	font-size: 14px;
	font-weight: 650;
	cursor: pointer;
	flex: 1;
}
.btn-primary {
	background: var(--primary);
	color: #fff;
}
.btn-primary:active {
	background: #0066bc;
}
.btn-secondary {
	background: #fff;
	color: var(--text-2);
	border: 1px solid #c7c7c7;
	flex: 0 0 auto;
}
.btn[disabled] {
	opacity: 0.5;
	cursor: default;
}
</style>
