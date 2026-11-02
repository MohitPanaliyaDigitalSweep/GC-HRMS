<template>
	<BaseLayout :pageTitle="__('Party Bills')">
		<template #body>
			<div class="party-bills-page p-4 flex flex-col gap-4">
				<div v-if="statusMsg" class="rounded-lg p-3 text-sm" :class="statusClass">{{ statusMsg }}</div>

				<!-- Summary cards -->
				<div class="grid grid-cols-2 gap-3">
					<div class="rounded-lg bg-white p-3 shadow-sm">
						<p class="text-xs text-gray-500">{{ __("Total Outstanding") }}</p>
						<p class="text-lg font-bold text-gray-800">{{ formatMoney(summary.total_outstanding) }}</p>
					</div>
					<div class="rounded-lg bg-white p-3 shadow-sm">
						<p class="text-xs text-gray-500">{{ __("Total Overdue") }}</p>
						<p class="text-lg font-bold text-red-600">{{ formatMoney(summary.total_overdue) }}</p>
					</div>
				</div>
				<div class="grid grid-cols-2 gap-3">
					<div class="rounded-lg bg-white p-3 shadow-sm">
						<p class="text-xs text-gray-500">{{ __("Overdue Invoices") }}</p>
						<p class="text-lg font-bold text-orange-600">{{ summary.overdue_count }}</p>
					</div>
					<div class="rounded-lg bg-white p-3 shadow-sm">
						<p class="text-xs text-gray-500">{{ __("Total Invoices") }}</p>
						<p class="text-lg font-bold text-gray-800">{{ summary.total_invoices }}</p>
					</div>
				</div>

				<!-- Filters -->
				<div class="rounded-lg bg-white p-4 shadow-sm">
					<div class="flex flex-col gap-3">
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Company") }}</label>
							<select v-model="company" @change="loadData" class="w-full rounded border border-gray-300 px-3 py-2 text-sm">
								<option v-for="c in companies" :key="c" :value="c">{{ c }}</option>
							</select>
						</div>
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Party Type") }}</label>
							<select v-model="partyType" @change="loadData" class="w-full rounded border border-gray-300 px-3 py-2 text-sm">
								<option value="Customer">{{ __("Customer (Receivable)") }}</option>
								<option value="Supplier">{{ __("Supplier (Payable)") }}</option>
							</select>
						</div>
						<div>
							<input v-model="search" @input="debouncedLoad" type="text" class="w-full rounded border border-gray-300 px-3 py-2 text-sm" placeholder="Search party..." />
						</div>
					</div>
				</div>

				<!-- Loading -->
				<div v-if="loading" class="text-center py-8 text-gray-400">
					<i class="fa fa-spinner fa-spin text-2xl"></i>
				</div>

				<!-- Party list -->
				<div v-for="party in parties" :key="party.party" class="rounded-lg bg-white shadow-sm overflow-hidden">
					<div
						class="flex items-center justify-between p-4 cursor-pointer"
						@click="toggleParty(party.party)"
					>
						<div class="flex-1 min-w-0">
							<p class="text-sm font-semibold text-gray-800 truncate">{{ party.party_name || party.party }}</p>
							<p class="text-xs text-gray-500">{{ party.invoice_count }} invoice{{ party.invoice_count !== 1 ? "s" : "" }}</p>
						</div>
						<div class="text-right">
							<p class="text-sm font-bold" :class="party.overdue_amount > 0 ? 'text-red-600' : 'text-gray-800'">
								{{ formatMoney(party.total_outstanding) }}
							</p>
							<p v-if="party.overdue_amount > 0" class="text-xs text-red-500">
								{{ __("Overdue:") }} {{ formatMoney(party.overdue_amount) }}
							</p>
						</div>
						<i class="fa ml-2 text-gray-400" :class="expanded[party.party] ? 'fa-chevron-up' : 'fa-chevron-down'"></i>
					</div>

					<!-- Invoice details -->
					<div v-if="expanded[party.party]" class="border-t border-gray-100">
						<div v-for="inv in party.invoices" :key="inv.name" class="flex items-center justify-between px-4 py-2 text-xs border-b border-gray-50 last:border-0">
							<div>
								<p class="font-medium text-gray-700">{{ inv.name }}</p>
								<p class="text-gray-500">{{ inv.posting_date }}{{ inv.due_date ? " · Due " + inv.due_date : "" }}</p>
							</div>
							<div class="text-right">
								<p class="font-semibold" :class="inv.is_overdue ? 'text-red-600' : 'text-gray-700'">
									{{ formatMoney(inv.outstanding_amount) }}
								</p>
								<p v-if="inv.is_overdue" class="text-xs text-red-500 font-medium">{{ __("Overdue") }}</p>
							</div>
						</div>
					</div>
				</div>

				<div v-if="!loading && parties.length === 0" class="text-center py-8 text-gray-400">
					{{ __("No pending bills found.") }}
				</div>
			</div>
		</template>
	</BaseLayout>
</template>

<script setup>
import { ref, onMounted } from "vue"
import BaseLayout from "@/components/BaseLayout.vue"
import { call } from "frappe-ui"

const company = ref("")
const partyType = ref("Customer")
const search = ref("")
const parties = ref([])
const summary = ref({ total_outstanding: 0, total_overdue: 0, overdue_count: 0, total_invoices: 0 })
const loading = ref(false)
const expanded = ref({})
const companies = ref([])

let searchTimer = null

onMounted(async () => {
	try {
		const comps = await call("frappe.client.get_list", { doctype: "Company", fields: ["name"], limit_page_length: 100 })
		companies.value = (comps || []).map((c) => c.name)
		if (companies.value.length > 0 && !company.value) company.value = companies.value[0]
	} catch (e) { console.error(e) }
	await loadData()
})

async function loadData() {
	loading.value = true
	try {
		const [bills, summ] = await Promise.all([
			call("fast_entry_app.api.party_bills.get_party_bills_summary", {
				company: company.value,
				party_type: partyType.value,
				search: search.value || "",
			}),
			call("fast_entry_app.api.party_bills.get_party_overdue_summary", {
				company: company.value,
				party_type: partyType.value,
			}),
		])
		parties.value = bills || []
		summary.value = summ || { total_outstanding: 0, total_overdue: 0, overdue_count: 0, total_invoices: 0 }
	} catch (e) {
		console.error(e)
		parties.value = []
	} finally {
		loading.value = false
	}
}

function debouncedLoad() {
	clearTimeout(searchTimer)
	searchTimer = setTimeout(loadData, 400)
}

function toggleParty(party) {
	expanded.value[party] = !expanded.value[party]
}

function formatMoney(v) {
	const n = parseFloat(v) || 0
	return n.toLocaleString("en-IN", { minimumFractionDigits: 2, maximumFractionDigits: 2 })
}
</script>