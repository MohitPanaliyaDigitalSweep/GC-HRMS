<template>
	<ion-tab-bar
		slot="bottom"
		class="bg-white shadow-md sm:w-96 py-2 pb-2 standalone:pb-safe-bottom"
	>
		<ion-tab-button
			v-for="item in mainTabs"
			:key="item.title"
			:tab="item.title"
			:href="item.route"
			:class="[
				'bg-white text-xs space-y-1.5 !hover:border-gray-300 !hover:text-gray-700 transition active:scale-95',
				route.path === item.route
					? 'border-gray-900 text-gray-800 font-semibold'
					: 'text-gray-600 font-normal',
			]"
		>
			<component :is="item.icon" class="h-5 w-5" />
			<div>{{ item.title }}</div>
		</ion-tab-button>

		<!-- More button -->
		<ion-tab-button
			tab="more"
			@click="showMore = true"
			:class="[
				'bg-white text-xs space-y-1.5 !hover:border-gray-300 !hover:text-gray-700 transition active:scale-95',
				moreActive ? 'border-gray-900 text-gray-800 font-semibold' : 'text-gray-600 font-normal',
			]"
		>
			<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
				<circle cx="12" cy="12" r="1" />
				<circle cx="19" cy="12" r="1" />
				<circle cx="5" cy="12" r="1" />
			</svg>
			<div>{{ __("More") }}</div>
		</ion-tab-button>
	</ion-tab-bar>

	<!-- More menu modal -->
	<ion-modal :is-open="showMore" @didDismiss="showMore = false" class="more-modal">
		<ion-header>
			<ion-toolbar>
				<ion-title>{{ __("More") }}</ion-title>
				<ion-buttons slot="end">
					<ion-button @click="showMore = false">{{ __("Close") }}</ion-button>
				</ion-buttons>
			</ion-toolbar>
		</ion-header>
		<ion-content class="ion-padding">
			<div class="grid grid-cols-3 gap-4 py-4">
				<div
					v-for="item in moreItems"
					:key="item.title"
					class="flex flex-col items-center gap-2 p-3 rounded-lg cursor-pointer hover:bg-gray-50 active:scale-95 transition"
					@click="navigate(item.route)"
				>
					<div class="w-12 h-12 rounded-full bg-blue-50 flex items-center justify-center text-blue-600">
						<component :is="item.icon" class="h-6 w-6" />
					</div>
					<span class="text-xs text-center text-gray-700 font-medium">{{ item.title }}</span>
				</div>
			</div>
		</ion-content>
	</ion-modal>
</template>

<script setup>
import { useRoute, useRouter } from "vue-router"
import { ref, computed } from "vue"

import { IonTabBar, IonTabButton, IonModal, IonHeader, IonToolbar, IonTitle, IonButtons, IonButton, IonContent } from "@ionic/vue"

import HomeIcon from "@/components/icons/HomeIcon.vue"
import LeaveIcon from "@/components/icons/LeaveIcon.vue"
import ExpenseIcon from "@/components/icons/ExpenseIcon.vue"
import SalaryIcon from "@/components/icons/SalaryIcon.vue"
import AttendanceIcon from "@/components/icons/AttendanceIcon.vue"
import SalesInvoiceIcon from "@/components/icons/SalesInvoiceIcon.vue"
import LocationIcon from "@/components/icons/LocationIcon.vue"
import MapIcon from "@/components/icons/MapIcon.vue"
import BillsIcon from "@/components/icons/BillsIcon.vue"
import { inject } from "vue"

const __ = inject("$translate")

const route = useRoute()
const router = useRouter()

const showMore = ref(false)

const mainTabs = [
	{
		icon: HomeIcon,
		title: __("Home"),
		route: "/home",
	},
	{
		icon: AttendanceIcon,
		title: __("Attendance"),
		route: "/dashboard/attendance",
	},
	{
		icon: LeaveIcon,
		title: __("Leaves"),
		route: "/dashboard/leaves",
	},
	{
		icon: ExpenseIcon,
		title: __("Expenses"),
		route: "/dashboard/expense-claims",
	},
	{
		icon: SalaryIcon,
		title: __("Salary"),
		route: "/dashboard/salary-slips",
	},
]

const moreItems = [
	{
		icon: SalesInvoiceIcon,
		title: __("Sales Invoice"),
		route: "/sales-invoice",
	},
	{
		icon: LocationIcon,
		title: __("Log Visit"),
		route: "/log-visit",
	},
	{
		icon: MapIcon,
		title: __("Live Map"),
		route: "/live-map",
	},
	{
		icon: BillsIcon,
		title: __("Party Bills"),
		route: "/party-bills",
	},
]

const moreActive = computed(() => moreItems.some((item) => item.route === route.path))

function navigate(routePath) {
	showMore.value = false
	router.push(routePath)
}
</script>

<style scoped>
.more-modal {
	--height: 50%;
}
</style>