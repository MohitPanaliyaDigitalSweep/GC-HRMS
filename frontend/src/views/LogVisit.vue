<template>
	<BaseLayout :pageTitle="__('Log Visit')">
		<template #body>
			<div class="log-visit-page p-4 flex flex-col gap-4">
				<!-- Today's visits counter -->
				<div class="rounded-lg bg-blue-50 p-3 flex items-center justify-between">
					<span class="text-sm font-medium text-blue-800">{{ __("Today's Visits") }}</span>
					<span class="text-lg font-bold text-blue-600">{{ todayCount }}</span>
				</div>
				<!-- Status -->
				<div v-if="statusMsg" class="rounded-lg p-3 text-sm" :class="statusClass">{{ statusMsg }}</div>

				<!-- Employee & Company -->
				<div class="rounded-lg bg-white p-4 shadow-sm">
					<p class="text-sm font-semibold text-gray-700 mb-3">{{ __("Visit Details") }}</p>
					<div class="flex flex-col gap-3">
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Employee") }} *</label>
							<select v-model="employee" class="w-full rounded border border-gray-300 px-3 py-2 text-sm">
								<option value="">{{ __("Select Employee") }}</option>
								<option v-for="e in employees" :key="e.name" :value="e.name">{{ e.employee_name || e.name }}</option>
							</select>
						</div>
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Company") }}</label>
							<select v-model="company" class="w-full rounded border border-gray-300 px-3 py-2 text-sm">
								<option value="">{{ __("Select Company") }}</option>
								<option v-for="c in companies" :key="c" :value="c">{{ c }}</option>
							</select>
						</div>
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Party Type") }}</label>
							<select v-model="partyType" class="w-full rounded border border-gray-300 px-3 py-2 text-sm">
								<option value="Customer">{{ __("Customer") }}</option>
								<option value="Supplier">{{ __("Supplier") }}</option>
								<option value="Lead">{{ __("Lead") }}</option>
								<option value="None">{{ __("None") }}</option>
							</select>
						</div>
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Party") }}</label>
							<input v-model="partySearch" @input="searchParty" type="text" class="w-full rounded border border-gray-300 px-3 py-2 text-sm" placeholder="Search..." />
							<div v-if="partyResults.length" class="mt-1 max-h-32 overflow-y-auto rounded border border-gray-200 bg-white">
								<div
									v-for="pr in partyResults"
									:key="pr.name"
									class="px-3 py-2 text-sm cursor-pointer hover:bg-gray-50"
									@click="pickParty(pr)"
								>
									{{ pr.party_name || pr.name }}
								</div>
							</div>
						</div>
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Visit Type") }}</label>
							<select v-model="visitType" class="w-full rounded border border-gray-300 px-3 py-2 text-sm">
								<option value="Customer Visit">{{ __("Customer Visit") }}</option>
								<option value="Site Survey">{{ __("Site Survey") }}</option>
								<option value="Delivery Follow-up">{{ __("Delivery Follow-up") }}</option>
								<option value="Collection">{{ __("Collection") }}</option>
								<option value="Other">{{ __("Other") }}</option>
							</select>
						</div>
					</div>
				</div>

				<!-- Location -->
				<div class="rounded-lg bg-white p-4 shadow-sm">
					<p class="text-sm font-semibold text-gray-700 mb-3">{{ __("Location") }}</p>
					<button
						class="w-full rounded-lg bg-blue-600 py-3 text-sm font-semibold text-white"
						:disabled="capturing"
						@click="captureLocation"
					>
						<span v-if="capturing" class="animate-pulse">{{ __("Capturing...") }}</span>
						<span v-else><i class="fa fa-crosshairs mr-1"></i> {{ __("Capture Current Location") }}</span>
					</button>
					<p v-if="locationStatus" class="mt-2 text-xs" :class="locationOk ? 'text-green-600' : 'text-red-600'">{{ locationStatus }}</p>
					<div v-if="lat && lng" class="mt-2 text-xs text-gray-500">
						{{ lat.toFixed(6) }}, {{ lng.toFixed(6) }}
					</div>
				</div>

				<!-- Photo & Remarks -->
				<div class="rounded-lg bg-white p-4 shadow-sm">
					<p class="text-sm font-semibold text-gray-700 mb-3">{{ __("Photo & Remarks") }}</p>
					<div class="flex flex-col gap-3">
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Photos") }}</label>
							<div class="flex gap-2 mb-2">
								<button type="button" class="flex-1 rounded-lg bg-blue-600 py-2 text-sm font-semibold text-white" @click="openCamera">
									<i class="fa fa-camera mr-1"></i> {{ __("Take Photo") }}
								</button>
								<label class="flex-1 rounded-lg bg-gray-100 py-2 text-sm font-semibold text-gray-700 text-center cursor-pointer">
									<i class="fa fa-folder-open mr-1"></i> {{ __("Choose File") }}
									<input type="file" accept="image/*" @change="onPhoto" class="hidden" multiple />
								</label>
							</div>
							<video v-if="cameraActive" ref="cameraVideo" autoplay playsinline class="w-full rounded-lg border mb-2" style="max-height:240px;"></video>
							<button v-if="cameraActive" type="button" class="w-full rounded-lg bg-green-600 py-2 text-sm font-semibold text-white mb-2" @click="capturePhoto">
								<i class="fa fa-circle mr-1"></i> {{ __("Capture") }}
							</button>
							<!-- Photo previews -->
							<div v-if="photoPreviews.length" class="flex flex-wrap gap-2 mt-2">
								<div v-for="(src, idx) in photoPreviews" :key="idx" class="relative">
									<img :src="src" class="w-20 h-20 object-cover rounded-lg border" />
									<button type="button" class="absolute -top-1 -right-1 w-5 h-5 bg-red-600 text-white rounded-full text-xs flex items-center justify-center" @click="removePhoto(idx)">
										&times;
									</button>
								</div>
							</div>
							<p v-if="photoPreviews.length" class="text-xs text-gray-500 mt-1">{{ photoPreviews.length }} photo(s) attached</p>
						</div>
						<div>
							<label class="text-xs font-medium text-gray-600 mb-1 block">{{ __("Remarks") }}</label>
							<textarea v-model="remarks" rows="3" class="w-full rounded border border-gray-300 px-3 py-2 text-sm" placeholder="Notes about this visit..."></textarea>
						</div>
					</div>
				</div>

				<button
					class="w-full rounded-lg bg-green-600 py-3 text-sm font-semibold text-white"
					:disabled="saving"
					@click="saveVisit"
				>
					<span v-if="saving" class="animate-pulse">{{ __("Saving...") }}</span>
					<span v-else>{{ __("Save Visit") }}</span>
				</button>

				<!-- Recent visits -->
				<div class="rounded-lg bg-white p-4 shadow-sm">
					<p class="text-sm font-semibold text-gray-700 mb-3">{{ __("Recent Visits") }}</p>
					<div v-if="recentVisits.length === 0" class="text-center text-sm text-gray-400 py-4">{{ __("No visits logged yet.") }}</div>
					<div v-for="v in recentVisits" :key="v.name" class="flex items-start gap-3 py-2 border-b border-gray-100 last:border-0">
						<div v-if="v.photos && v.photos.length" class="flex -space-x-2">
							<img v-for="(ph, pi) in v.photos.slice(0, 3)" :key="pi" :src="ph.photo" class="w-10 h-10 rounded object-cover border-2 border-white" />
							<span v-if="v.photos.length > 3" class="w-10 h-10 rounded bg-gray-200 flex items-center justify-center text-xs font-bold text-gray-600 border-2 border-white">+{{ v.photos.length - 3 }}</span>
						</div>
						<div v-else-if="v.photo" class="w-10 h-10 rounded bg-gray-100 flex items-center justify-center">
							<img :src="v.photo" class="w-10 h-10 rounded object-cover" />
						</div>
						<div v-else class="w-10 h-10 rounded bg-gray-100 flex items-center justify-center">
							<i class="fa fa-map-marker text-gray-400"></i>
						</div>
						<div class="flex-1 min-w-0">
							<p class="text-sm font-medium text-gray-800 truncate">{{ v.employee_name || v.employee }} — {{ v.visit_type }}</p>
							<p class="text-xs text-gray-500">{{ v.visit_date }} {{ v.visit_time }}<template v-if="v.party_name"> · {{ v.party_name }}</template></p>
							<p v-if="v.remarks" class="text-xs text-gray-600 mt-1">{{ v.remarks }}</p>
						</div>
						<span class="text-xs px-2 py-0.5 rounded" :class="v.status === 'Submitted' ? 'bg-green-100 text-green-700' : 'bg-yellow-100 text-yellow-700'">{{ v.status }}</span>
					</div>
				</div>
			</div>
		</template>
	</BaseLayout>
</template>

<script setup>
import { ref, onMounted, watch, nextTick } from "vue"
import BaseLayout from "@/components/BaseLayout.vue"
import { call } from "frappe-ui"

const employee = ref("")
const company = ref("")
const partyType = ref("Customer")
const partySearch = ref("")
const partyResults = ref([])
const partyPicked = ref("")
const visitType = ref("Customer Visit")
const remarks = ref("")
const lat = ref(null)
const lng = ref(null)
const geoJson = ref("")
const capturing = ref(false)
const locationStatus = ref("")
const locationOk = ref(false)
const photoFiles = ref([])
const photoPreviews = ref([])
const cameraActive = ref(false)
const cameraStream = ref(null)
const cameraVideo = ref(null)
const saving = ref(false)
const statusMsg = ref("")
const statusClass = ref("")
const employees = ref([])
const companies = ref([])
const recentVisits = ref([])
const todayCount = ref(0)

onMounted(async () => {
	try {
		const emps = await call("fast_entry_app.api.field_visit.get_employees", { company: "" })
		employees.value = emps || []
		try {
			const me = await call("fast_entry_app.api.field_visit.get_my_employee", {})
			if (me && me.name) {
				employee.value = me.name
				if (me.company && companies.value.includes(me.company)) company.value = me.company
			} else if (employees.value.length === 1) {
				employee.value = employees.value[0].name
			}
		} catch (e) { console.error(e) }
	} catch (e) { console.error(e) }
	try {
		const comps = await call("frappe.client.get_list", { doctype: "Company", fields: ["name"], limit_page_length: 100 })
		companies.value = (comps || []).map((c) => c.name)
		if (companies.value.length === 1) company.value = companies.value[0]
	} catch (e) { console.error(e) }
	loadRecentVisits()
	loadTodayCount()
})

async function searchParty() {
	if (partySearch.value.length < 1) { partyResults.value = []; return }
	const method = partyType.value === "Customer"
		? "fast_entry_app.api.field_visit.get_customers"
		: partyType.value === "Supplier"
		? "fast_entry_app.api.field_visit.get_suppliers"
		: null
	if (!method) { partyResults.value = []; return }
	try {
		const res = await call(method, { search: partySearch.value, limit: 10 })
		partyResults.value = res || []
	} catch (e) { console.error(e); partyResults.value = [] }
}

function pickParty(pr) {
	partyPicked.value = pr.name
	partySearch.value = pr.party_name || pr.name
	partyResults.value = []
}

function captureLocation() {
	capturing.value = true
	locationStatus.value = ""
	if (!navigator.geolocation) {
		locationStatus.value = "Geolocation not supported."
		locationOk.value = false
		capturing.value = false
		return
	}
	navigator.geolocation.getCurrentPosition(
		(pos) => {
			lat.value = pos.coords.latitude
			lng.value = pos.coords.longitude
			const acc = pos.coords.accuracy
			geoJson.value = JSON.stringify({ type: "Feature", geometry: { type: "Point", coordinates: [lng.value, lat.value] }, properties: { accuracy: acc } })
			locationStatus.value = "Location captured: " + lat.value.toFixed(6) + ", " + lng.value.toFixed(6) + " (±" + Math.round(acc) + "m)"
			locationOk.value = true
			capturing.value = false
		},
		(err) => {
			locationStatus.value = "Location error: " + err.message
			locationOk.value = false
			capturing.value = false
		},
		{ enableHighAccuracy: true, timeout: 15000, maximumAge: 0 }
	)
}

function onPhoto(e) {
	const files = e.target.files
	if (!files || !files.length) return
	for (let i = 0; i < files.length; i++) {
		const file = files[i]
		photoFiles.value.push(file)
		const reader = new FileReader()
		reader.onload = (ev) => { photoPreviews.value.push(ev.target.result) }
		reader.readAsDataURL(file)
	}
	// Reset input so same file can be selected again
	e.target.value = ""
}

function removePhoto(idx) {
	photoFiles.value.splice(idx, 1)
	photoPreviews.value.splice(idx, 1)
}

async function openCamera() {
	try {
		const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } })
		cameraStream.value = stream
		cameraActive.value = true
	} catch (e) {
		console.error(e)
		showStatus("Camera access denied or not available.", "err")
	}
}

watch(cameraActive, async (val) => {
	if (val && cameraStream.value) {
		await nextTick()
		// Retry a few times in case the video element isn't rendered yet
		for (let i = 0; i < 10; i++) {
			if (cameraVideo.value) {
				cameraVideo.value.srcObject = cameraStream.value
				try { await cameraVideo.value.play() } catch (e) {}
				return
			}
			await new Promise((r) => setTimeout(r, 50))
		}
	}
})

function capturePhoto() {
	if (!cameraVideo.value) return
	const video = cameraVideo.value
	const canvas = document.createElement("canvas")
	canvas.width = video.videoWidth
	canvas.height = video.videoHeight
	const ctx = canvas.getContext("2d")
	ctx.drawImage(video, 0, 0)
	const dataUrl = canvas.toDataURL("image/jpeg", 0.85)
	photoPreviews.value.push(dataUrl)
	// Convert to file
	fetch(dataUrl).then(function (r) { return r.blob() }).then(function (blob) {
		photoFiles.value.push(new File([blob], "visit_photo_" + Date.now() + ".jpg", { type: "image/jpeg" }))
	})
	// Stop camera
	stopCamera()
}

function stopCamera() {
	if (cameraStream.value) {
		cameraStream.value.getTracks().forEach(function (t) { t.stop() })
		cameraStream.value = null
	}
	cameraActive.value = false
}

async function saveVisit() {
	if (!employee.value) { showStatus("Select an employee.", "err"); return }
	if (!lat.value || !lng.value) { showStatus("Capture location first.", "err"); return }
	saving.value = true
	statusMsg.value = ""
	try {
		// Upload all photos first (as standalone files, no doc link needed)
		let photoUrls = []
		for (let i = 0; i < photoFiles.value.length; i++) {
			const file = photoFiles.value[i]
			const b64 = await new Promise((resolve) => {
				const reader = new FileReader()
				reader.onload = (ev) => resolve(ev.target.result.split(",")[1])
				reader.readAsDataURL(file)
			})
			try {
				const r = await call("frappe.client.insert", {
					doc: JSON.stringify({
						doctype: "File",
						file_name: file.name,
						folder: "Home",
						is_private: 0,
						content: b64,
						decode: 1,
						ignore_duplicate_entry_error: 1,
					}),
				})
				if (r && r.file_url) photoUrls.push(r.file_url)
				else if (r && r.name) {
					const f = await call("frappe.client.get_value", { doctype: "File", name: r.name, fieldname: "file_url" })
					if (f && f.file_url) photoUrls.push(f.file_url)
				}
			} catch (e) {
				console.error("Photo upload failed:", e)
			}
		}

		// Create the visit as Submitted with photo
		const data = {
			employee: employee.value,
			company: company.value || "",
			visit_date: new Date().toISOString().slice(0, 10),
			visit_time: new Date().toTimeString().slice(0, 5),
			party_type: partyType.value,
			party: partyPicked.value,
			party_name: partySearch.value || "",
			visit_type: visitType.value,
			geolocation: geoJson.value || "",
			latitude: lat.value,
			longitude: lng.value,
			photo: photoUrls.length > 0 ? photoUrls[0] : "",
			photos: photoUrls,
			remarks: remarks.value || "",
			status: "Submitted",
		}
		const res = await call("fast_entry_app.api.field_visit.create_field_visit", { data: JSON.stringify(data) })
		if (!res || !res.name) {
			showStatus("Failed to create visit.", "err")
			saving.value = false
			return
		}
		const visitName = res.name

		showStatus("Visit " + visitName + " saved!", "ok")
		loadRecentVisits()
		loadTodayCount()
		remarks.value = ""
		photoFiles.value = []
		photoPreviews.value = []
		lat.value = null
		lng.value = null
		geoJson.value = ""
		locationStatus.value = "No location captured."
		locationOk.value = false
	} catch (e) {
		console.error(e)
		var errMsg = "Failed to save visit."
		if (e && e.message) errMsg = e.message
		else if (e && e.response && e.response._data && e.response._data.exception) errMsg = e.response._data.exception
		showStatus(errMsg, "err")
	} finally {
		saving.value = false
	}
}

async function loadRecentVisits() {
	try {
		const res = await call("fast_entry_app.api.field_visit.get_field_visits", { limit: 10 })
		recentVisits.value = res || []
	} catch (e) { console.error(e); recentVisits.value = [] }
}

async function loadTodayCount() {
	try {
		const res = await call("fast_entry_app.api.field_visit.get_today_visit_count", { employee: employee.value || "" })
		todayCount.value = res || 0
	} catch (e) { console.error(e) }
}

function showStatus(msg, type) {
	statusMsg.value = msg
	statusClass.value = type === "ok" ? "status ok" : type === "err" ? "status err" : "status info"
}
</script>