<script setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'
import DashboardLayout from '@/layouts/DashboardLayout.vue'
import BaseBadge from '@/components/ui/BaseBadge.vue'
import facilitiesData from '@/data/facilities.json'

import {
  ArrowLeftIcon,
  BuildingOffice2Icon,
  BeakerIcon,
  HeartIcon,
  MapPinIcon,
  UsersIcon,
} from '@heroicons/vue/24/outline'

const route = useRoute()

const facility = computed(() => {
  const key = decodeURIComponent(String(route.params.id))
  return facilitiesData.find((f) => f.id === key) ?? null
})

const statusTone = { active: 'success', pending: 'warning', suspended: 'danger' }
const typeIcon = { Hospital: BuildingOffice2Icon, Clinic: HeartIcon, Pharmacy: BeakerIcon }
</script>

<template>
  <DashboardLayout>
    <!-- Back -->
    <RouterLink
      :to="{ name: 'super-admin-facilities' }"
      class="mb-5 inline-flex items-center gap-2 rounded-lg px-2 py-1.5 text-sm font-medium text-meridian-500 transition hover:text-meridian-800 focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500 dark:hover:text-white"
    >
      <ArrowLeftIcon class="h-4 w-4" />
      Back to facilities
    </RouterLink>

    <!-- Not found -->
    <div
      v-if="!facility"
      class="flex flex-col items-center rounded-2xl border border-dashed border-meridian-200 px-6 py-16 text-center dark:border-white/10"
    >
      <BuildingOffice2Icon class="h-12 w-12 text-meridian-400" />
      <h1 class="mt-3 font-display text-xl font-semibold text-meridian-800 dark:text-meridian-100">Facility not found</h1>
      <p class="mt-1 max-w-xs text-sm text-meridian-500">This facility may have been removed. Go back to the list and pick another one.</p>
      <RouterLink :to="{ name: 'super-admin-facilities' }" class="mt-5 text-sm font-medium text-meridian-600 hover:underline">
        Back to facilities
      </RouterLink>
    </div>

    <template v-else>
      <!-- Header -->
      <section
        class="overflow-hidden rounded-2xl border border-meridian-200 bg-white dark:border-white/10 dark:bg-meridian-900"
        :class="{ 'opacity-90': facility.status === 'suspended' }"
      >
        <div class="h-28 bg-meridian-600 sm:h-36" />

        <div class="px-5 pb-6 sm:px-8">
          <div class="flex flex-col items-center gap-4 text-center sm:flex-row sm:items-start sm:text-left">
            <div class="-mt-12 rounded-2xl bg-white p-1.5 dark:bg-meridian-900 sm:-mt-14">
              <span class="flex h-20 w-20 items-center justify-center rounded-xl bg-meridian-50 text-meridian-600 ring-1 ring-meridian-100 dark:bg-white/5 dark:ring-white/10 sm:h-24 sm:w-24">
                <component :is="typeIcon[facility.type]" class="h-10 w-10" />
              </span>
            </div>

            <div class="sm:pt-4">
              <h1 class="font-display text-2xl font-semibold text-meridian-900 dark:text-white">{{ facility.name }}</h1>
              <div class="mt-2 flex flex-wrap items-center justify-center gap-2 sm:justify-start">
                <span class="rounded-md bg-meridian-100/70 px-2.5 py-0.5 text-sm font-medium text-meridian-700 dark:bg-white/10 dark:text-meridian-100">
                  {{ facility.type }}
                </span>
                <BaseBadge :tone="statusTone[facility.status]" class="capitalize">{{ facility.status }}</BaseBadge>
                <span class="font-mono text-xs text-meridian-400">{{ facility.id.toUpperCase() }}</span>
              </div>
            </div>
          </div>
        </div>
      </section>

      <div class="mt-6 grid gap-6 lg:grid-cols-3">
        <!-- Summary -->
        <section class="h-fit rounded-2xl border border-meridian-200 bg-white p-5 dark:border-white/10 dark:bg-white/5">
          <h2 class="font-display text-base font-semibold text-meridian-900 dark:text-white">Summary</h2>

          <ul class="mt-4 space-y-4 text-sm">
            <li class="flex items-start gap-3">
              <span class="rounded-lg bg-meridian-50 p-2 text-meridian-600 dark:bg-white/5">
                <UsersIcon class="h-4 w-4" />
              </span>
              <div>
                <p class="text-meridian-500">Providers</p>
                <p class="font-medium tabular-nums text-meridian-800 dark:text-meridian-100">{{ facility.providers }}</p>
              </div>
            </li>
            <li class="flex items-start gap-3">
              <span class="rounded-lg bg-meridian-50 p-2 text-meridian-600 dark:bg-white/5">
                <MapPinIcon class="h-4 w-4" />
              </span>
              <div class="min-w-0">
                <p class="text-meridian-500">Location</p>
                <p class="break-words font-medium text-meridian-800 dark:text-meridian-100">{{ facility.location }}</p>
              </div>
            </li>
          </ul>
        </section>

        <!-- Facility information -->
        <section class="rounded-2xl border border-meridian-200 bg-white p-5 dark:border-white/10 dark:bg-white/5 lg:col-span-2">
          <h2 class="font-display text-base font-semibold text-meridian-900 dark:text-white">Facility information</h2>

          <dl class="mt-4 grid gap-3 sm:grid-cols-2">
            <div class="rounded-xl border border-meridian-200 px-4 py-3 dark:border-white/10">
              <dt class="text-xs text-meridian-500">Type</dt>
              <dd class="mt-1 text-sm font-medium text-meridian-800 dark:text-meridian-100">{{ facility.type }}</dd>
            </div>
            <div class="rounded-xl border border-meridian-200 px-4 py-3 dark:border-white/10">
              <dt class="text-xs text-meridian-500">Status</dt>
              <dd class="mt-1 text-sm font-medium capitalize text-meridian-800 dark:text-meridian-100">{{ facility.status }}</dd>
            </div>
            <div class="rounded-xl border border-meridian-200 px-4 py-3 dark:border-white/10">
              <dt class="text-xs text-meridian-500">Facility ID</dt>
              <dd class="mt-1 font-mono text-sm font-medium text-meridian-800 dark:text-meridian-100">{{ facility.id.toUpperCase() }}</dd>
            </div>
            <div class="rounded-xl border border-meridian-200 px-4 py-3 dark:border-white/10">
              <dt class="text-xs text-meridian-500">Providers</dt>
              <dd class="mt-1 text-sm font-medium tabular-nums text-meridian-800 dark:text-meridian-100">{{ facility.providers }}</dd>
            </div>
          </dl>
        </section>
      </div>
    </template>
  </DashboardLayout>
</template>