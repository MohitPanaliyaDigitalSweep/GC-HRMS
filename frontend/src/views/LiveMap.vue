<template>
	<BaseLayout :pageTitle="__('Live Map')">
		<template #body>
			<div class="live-map-page p-4 flex flex-col gap-4">
				<div v-if="statusMsg" class="rounded-lg p-3 text-sm" :class="statusClass">{{ statusMsg }}</div>

				<div class="rounded-lg bg-white p-4 shadow-sm">
					<div class="flex flex-col gap-3">
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Company") }}</label>
							<select v-model="company" @change="refresh" class="w-full rounded border border-gray-300 px-3 py-2 text-sm">
								<option value="">{{ __("All Companies") }}</option>
								<option v-for="c in companies" :key="c" :value="c">{{ c }}</option>
							</select>
						</div>
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Employee") }}</label>
							<select v-model="employee" @change="refresh" class="w-full rounded border border-gray-300 px-3 py-2 text-sm">
								<option value="">{{ __("All Employees") }}</option>
								<option v-for="e in employees" :key="e.name" :value="e.name">{{ e.employee_name || e.name }}</option>
							</select>
						</div>
						<div class="flex gap-2">
							<button class="flex-1 rounded-lg bg-blue-600 py-2 text-sm font-semibold text-white" @click="refresh">
								<i class="fa fa-refresh mr-1"></i> {{ __("Refresh") }}
							</button>
							<button
								class="flex-1 rounded-lg py-2 text-sm font-semibold text-white"
								:class="tracking ? 'bg-red-600' : 'bg-green-600'"
								@click="toggleTracking"
							>
								<i class="fa mr-1" :class="tracking ? 'fa-stop' : 'fa-location-arrow'"></i>
								{{ tracking ? __("Stop Tracking") : __("Start Tracking") }}
							</button>
						</div>
						<p class="text-xs text-gray-500">{{ locationCount }} {{ __("employees") }}</p>
						<div class="flex gap-3 mt-1">
							<label class="text-xs flex items-center gap-1"><input type="checkbox" v-model="showEmployees" @change="refresh"> {{ __("Employees") }}</label>
							<label class="text-xs flex items-center gap-1"><input type="checkbox" v-model="showCustomers" @change="refresh"> {{ __("Customers") }}</label>
							<label class="text-xs flex items-center gap-1"><input type="checkbox" v-model="showVisits" @change="refresh"> {{ __("Visits") }}</label>
						</div>
					</div>
				</div>

				<!-- Map container -->
				<div class="rounded-lg bg-white shadow-sm overflow-hidden">
					<div ref="mapContainer" style="height: 400px; width: 100%;"></div>
				</div>

				<!-- Recent visits -->
				<div class="rounded-lg bg-white p-4 shadow-sm">
					<p class="text-sm font-semibold text-gray-700 mb-3">{{ __("Recent Visits") }}</p>
					<div v-if="recentVisits.length === 0" class="text-center text-sm text-gray-400 py-4">{{ __("No visits found.") }}</div>
					<div v-for="v in recentVisits" :key="v.name" class="flex items-start gap-3 py-2 border-b border-gray-100 last:border-0">
						<img v-if="v.photo" :src="v.photo" class="w-10 h-10 rounded object-cover" />
						<div v-else class="w-10 h-10 rounded bg-gray-100 flex items-center justify-center">
							<i class="fa fa-map-marker text-gray-400"></i>
						</div>
						<div class="flex-1 min-w-0">
							<p class="text-sm font-medium text-gray-800 truncate">{{ v.employee_name || v.employee }} — {{ v.visit_type }}</p>
							<p class="text-xs text-gray-500">{{ v.visit_date }} {{ v.visit_time }}<template v-if="v.party_name"> · {{ v.party_name }}</template></p>
							<p v-if="v.remarks" class="text-xs text-gray-600 mt-1">{{ v.remarks }}</p>
						</div>
						<a v-if="v.latitude" :href="'https://www.google.com/maps?q=' + v.latitude + ',' + v.longitude" target="_blank" class="text-xs text-blue-600">
							<i class="fa fa-external-link"></i>
						</a>
					</div>
				</div>
			</div>
		</template>
	</BaseLayout>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from "vue"
import BaseLayout from "@/components/BaseLayout.vue"
import { call } from "frappe-ui"

const company = ref("")
const employee = ref("")
const companies = ref([])
const employees = ref([])
const recentVisits = ref([])
const locationCount = ref(0)
const tracking = ref(false)
const showCustomers = ref(true)
const showEmployees = ref(true)
const showVisits = ref(true)
const statusMsg = ref("")
const statusClass = ref("")

let map = null
let markers = {}
let watchId = null
let refreshInterval = null

onMounted(async () => {
	try {
		const comps = await call("frappe.client.get_list", { doctype: "Company", fields: ["name"], limit_page_length: 100 })
		companies.value = (comps || []).map((c) => c.name)
	} catch (e) { console.error(e) }
	await loadEmployees()
	await initMap()
	refresh()
	refreshInterval = setInterval(refresh, 30000)
})

onUnmounted(() => {
	if (refreshInterval) clearInterval(refreshInterval)
	if (watchId) navigator.geolocation.clearWatch(watchId)
	if (map) map.remove()
})

async function loadEmployees() {
	try {
		const emps = await call("fast_entry_app.api.field_visit.get_employees", { company: company.value || "" })
		employees.value = emps || []
	} catch (e) { console.error(e) }
}

async function initMap() {
	await nextTick()
	const mapEl = document.querySelector("[ref='mapContainer']") || document.querySelector(".live-map-page div[style*='height: 400px']")
	// Find the map container
	const containers = document.querySelectorAll("div")
	let mapContainer = null
	for (const c of containers) {
		if (c.style && c.style.height === "400px") { mapContainer = c; break }
	}
	if (!mapContainer) return

	// Load leaflet
	if (!window.L) {
		await loadScript("https://unpkg.com/leaflet@1.9.4/dist/leaflet.js")
		await loadCss("https://unpkg.com/leaflet@1.9.4/dist/leaflet.css")
	}
	map = window.L.map(mapContainer).setView([20.5937, 78.9629], 5)
	window.L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
		attribution: "&copy; OpenStreetMap contributors",
		maxZoom: 19,
	}).addTo(map)
}

function loadScript(src) {
	return new Promise((resolve, reject) => {
		if (document.querySelector('script[src="' + src + '"]')) { resolve(); return }
		const s = document.createElement("script")
		s.src = src
		s.onload = resolve
		s.onerror = reject
		document.head.appendChild(s)
	})
}

function loadCss(href) {
	return new Promise((resolve, reject) => {
		if (document.querySelector('link[href="' + href + '"]')) { resolve(); return }
		const l = document.createElement("link")
		l.rel = "stylesheet"
		l.href = href
		l.onload = resolve
		l.onerror = reject
		document.head.appendChild(l)
	})
}

async function refresh() {
	try {
		const comp = company.value || ""
		// Fetch employee locations, customer locations, and visits in parallel
		const [empLocs, custLocs, visits] = await Promise.all([
			call("fast_entry_app.api.field_visit.get_live_locations", { company: comp }).catch(() => []),
			call("fast_entry_app.api.field_visit.get_customer_locations", { company: comp }).catch(() => []),
			call("fast_entry_app.api.field_visit.get_field_visits", { company: comp, employee: employee.value || "", limit: 500 }).catch(() => []),
		])

		// Employee markers (green)
		const empMarkers = (empLocs || []).filter((l) => showEmployees.value).map((l) => ({
			type: "employee",
			id: "emp_" + l.employee,
			name: l.employee_name || l.employee,
			company: l.company,
			lat: parseFloat(l.latitude),
			lng: parseFloat(l.longitude),
			info: l.party_name ? "Visiting: " + l.party_name : (l.remarks || ""),
			time: (l.visit_date || "") + " " + (l.visit_time || ""),
			color: "#059669",
		}))

		// Customer markers (blue)
		const custMarkers = (custLocs || []).filter((l) => showCustomers.value).map((l) => ({
			type: "customer",
			id: "cust_" + l.name,
			name: l.party_name || l.name,
			company: comp,
			lat: parseFloat(l.latitude),
			lng: parseFloat(l.longitude),
			info: "Customer",
			time: "",
			color: "#2563eb",
		}))

		// Visit markers (red) - show latest visit per location
		const visitMarkers = (visits || []).filter((v) => v.latitude && v.longitude && showVisits.value).map((v) => ({
			type: "visit",
			id: "visit_" + v.name,
			name: v.employee_name || v.employee,
			company: v.company || comp,
			lat: parseFloat(v.latitude),
			lng: parseFloat(v.longitude),
			info: (v.party_name ? "Visiting: " + v.party_name + " | " : "") + (v.remarks || ""),
			time: (v.visit_date || "") + " " + (v.visit_time || ""),
			color: "#dc2626",
			photo: v.photo,
		}))

		const allMarkers = [...empMarkers, ...custMarkers, ...visitMarkers]
		locationCount.value = empMarkers.length
		updateMarkers(allMarkers)
		recentVisits.value = (visits || []).slice(0, 30)
	} catch (e) {
		console.error(e)
	}
}

function updateMarkers(locations) {
	if (!map || !window.L) return
	const L = window.L
	// Remove all old markers
	for (const key of Object.keys(markers)) {
		map.removeLayer(markers[key])
		delete markers[key]
	}
	const bounds = []
	for (const loc of locations) {
		const lat = loc.lat || parseFloat(loc.latitude)
		const lng = loc.lng || parseFloat(loc.longitude)
		if (isNaN(lat) || isNaN(lng)) continue
		bounds.push([lat, lng])
		const color = loc.color || "#059669"
		// Create colored circle marker
		const marker = L.circleMarker([lat, lng], {
			radius: 8,
			fillColor: color,
			color: "#fff",
			weight: 2,
			opacity: 1,
			fillOpacity: 0.9,
		}).addTo(map)
		const popup = `<div style="min-width:180px;">
			<b style="font-size:13px;color:${color};">${esc(loc.name || loc.employee || "")}</b><br>
			<small style="color:#6b7280;">${esc(loc.type || "")}</small><br>
			${loc.info ? `<div style="margin-top:4px;font-size:12px;">${esc(loc.info)}</div>` : ""}
			${loc.photo ? `<img src="${loc.photo}" style="max-width:150px;margin-top:4px;border-radius:4px;" />` : ""}
			<div style="margin-top:4px;font-size:10px;color:#9ca3af;"><i class="fa fa-clock-o"></i> ${loc.time || ""}</div>
		</div>`
		marker.bindPopup(popup)
		markers[loc.id] = marker
	}
	if (bounds.length > 0 && bounds.length <= 100) {
		try { map.fitBounds(bounds, { padding: [30, 30], maxZoom: 14 }) } catch (e) {}
	}
	// Legend
	const legendParts = []
	if (showEmployees.value) legendParts.push('<span style="color:#059669;">●</span> Employees')
	if (showCustomers.value) legendParts.push('<span style="color:#2563eb;">●</span> Customers')
	if (showVisits.value) legendParts.push('<span style="color:#dc2626;">●</span> Visits')
	const legendEl = document.querySelector("#lm-legend")
	if (legendEl) legendEl.innerHTML = legendParts.join(" &nbsp; ") + " &nbsp; (" + locations.length + " markers)"
}

function toggleTracking() {
	if (tracking.value) {
		if (watchId) navigator.geolocation.clearWatch(watchId)
		watchId = null
		tracking.value = false
		showStatus("Location tracking stopped", "info")
		return
	}
	if (!navigator.geolocation) {
		showStatus("Geolocation not supported", "err")
		return
	}
	tracking.value = true
	showStatus("Location tracking started", "ok")
	watchId = navigator.geolocation.watchPosition(
		async (pos) => {
			try {
				await call("fast_entry_app.api.location.ping_location", {
					latitude: pos.coords.latitude,
					longitude: pos.coords.longitude,
					accuracy: pos.coords.accuracy,
				})
				refresh()
			} catch (e) { console.error(e) }
		},
		(err) => {
			showStatus("Location error: " + err.message, "err")
			tracking.value = false
		},
		{ enableHighAccuracy: true, timeout: 30000, maximumAge: 60000 }
	)
}

function showStatus(msg, type) {
	statusMsg.value = msg
	statusClass.value = type === "ok" ? "status ok" : type === "err" ? "status err" : "status info"
}

const esc = (v) => String(v == null ? "" : v).replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;")
</script>