<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRoute } from 'vue-router'
import DashboardLayout from '@/layouts/DashboardLayout.vue'
import BaseAvatar from '@/components/ui/BaseAvatar.vue'
import EmptyState from '@/components/ui/EmptyState.vue'
import SkeletonLine from '@/components/skeletons/SkeletonLine.vue'
import { useAuthStore } from '@/stores/auth'
import medicalRecordsData from '@/data/medicalRecords.json'
import billingData from '@/data/billing.json'
import prescriptionsData from '@/data/prescriptions.json'
import {
  HeartIcon, BeakerIcon, DocumentTextIcon, CalendarDaysIcon, BanknotesIcon,
  ExclamationTriangleIcon, MagnifyingGlassIcon, PrinterIcon, ChevronDownIcon,
  ClipboardDocumentListIcon, UserIcon,
} from '@heroicons/vue/24/outline'

const route = useRoute()
const auth = useAuthStore()
const isDoctor = computed(() => auth.role === 'doctor')
const loading = ref(true)

// Doctor lands here via /doctor/patients/:id — patient lands here on their own record.
const patientId = computed(() => route.params.id || auth.user?.id)

/* ------------------------------ Data ------------------------------ */
const visits = computed(() =>
  medicalRecordsData.filter((v) => v.patientId === patientId.value).sort((a, b) => (a.date < b.date ? 1 : -1))
)
const patientName = computed(() => visits.value[0]?.patient || auth.user?.name || 'Patient')
const invoices = computed(() => billingData.filter((i) => i.patientId === patientId.value))
const totalBilled = computed(() => invoices.value.reduce((s, i) => s + i.amount, 0))
const totalOutstanding = computed(() => invoices.value.filter((i) => i.status !== 'paid').reduce((s, i) => s + i.amount, 0))

function currency(v) {
  return `$${v.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`
}

// "2026-10-14" -> "14 Oct 2026". Anything not in that shape is shown as-is.
function formatDate(d) {
  if (!/^\d{4}-\d{2}-\d{2}$/.test(d || '')) return d
  return new Date(d).toLocaleDateString('en-GB', { day: 'numeric', month: 'short', year: 'numeric', timeZone: 'UTC' })
}

const conditionTags = computed(() => {
  const set = new Set()
  visits.value.forEach((v) => {
    const head = (v.diagnosis || '').split('—')[0]
    head.split(',').forEach((part) => {
      const trimmed = part.trim()
      if (trimmed) set.add(trimmed)
    })
  })
  return [...set]
})

const allergies = computed(() => {
  const set = new Set()
  visits.value.forEach((v) => (v.allergies || []).forEach((a) => set.add(a)))
  return [...set]
})

const prescriptions = computed(() =>
  prescriptionsData.filter((rx) => rx.patientId === patientId.value)
)

const currentMedications = computed(() =>
  prescriptions.value
    .filter((rx) => rx.status === 'active')
    .flatMap((rx) => rx.items.map((it) => ({ name: it.medicine, dose: `${it.dosage} · ${it.frequency}` })))
)

function medicationsForVisit(visit) {
  return prescriptions.value
    .filter((rx) => rx.issuedOn === visit.date)
    .flatMap((rx) => rx.items.map((it) => ({
      name: it.medicine,
      dose: `${it.dosage} · ${it.frequency}`,
      status: rx.status,
    })))
}

const latestVisit = computed(() => visits.value[0] || null)

/* ------------------------- Status styling & wording ------------------------- */
const rxStatusFlag = {
  active: 'text-emerald-700 bg-emerald-50 dark:text-[#34D399] dark:bg-[#065F46]/40',
  fulfilled: 'text-slate-600 bg-slate-100 dark:text-[#9CA3AF] dark:bg-[#374151]/40',
  pending: 'text-amber-700 bg-amber-50 dark:text-[#FBBF24] dark:bg-[#78350F]/40',
  expired: 'text-red-700 bg-red-50 dark:text-[#F87171] dark:bg-[#7F1D1D]/40',
}
const resultFlag = {
  normal: 'text-emerald-700 bg-emerald-50 dark:text-[#34D399] dark:bg-[#065F46]/40',
  high: 'text-red-700 bg-red-50 dark:text-[#F87171] dark:bg-[#7F1D1D]/40',
  low: 'text-amber-700 bg-amber-50 dark:text-[#FBBF24] dark:bg-[#78350F]/40',
  pending: 'text-slate-600 bg-slate-100 dark:text-[#9CA3AF] dark:bg-[#374151]/40',
}

// Patients see plain language, doctors see clinical shorthand.
const rxLabelPatient = { active: 'Taking now', fulfilled: 'Completed', pending: 'Being prepared', expired: 'Expired' }
const resultLabelPatient = { normal: 'Normal', high: 'Higher than normal', low: 'Lower than normal', pending: 'Awaiting result' }
const rxLabel = (s) => (isDoctor.value ? s : rxLabelPatient[s] || s)
const resultLabel = (s) => (isDoctor.value ? s : resultLabelPatient[s] || s)

const visitTone = {
  'follow-up': { badge: 'text-blue-600 bg-blue-50 dark:text-[#60A5FA] dark:bg-[#1E3A8A]/30', dot: 'border-blue-500 dark:border-[#3B82F6]' },
  acute: { badge: 'text-red-600 bg-red-50 dark:text-[#F87171] dark:bg-[#991B1B]/30', dot: 'border-red-500 dark:border-[#EF4444]' },
  default: { badge: 'text-emerald-600 bg-emerald-50 dark:text-[#34D399] dark:bg-[#065F46]/30', dot: 'border-emerald-500 dark:border-[#10B981]' },
}
function toneFor(visitType) {
  return visitTone[(visitType || '').toLowerCase()] || visitTone.default
}

/* ------------------------------ Filters ------------------------------ */
const searchQuery = ref('')
const typeFilter = ref('all')
const tagFilter = ref(null)

const visitTypes = computed(() => {
  const set = new Set(visits.value.map((v) => (v.visitType || 'default').toLowerCase()))
  return ['all', ...set]
})

function toggleTagFilter(tag) {
  tagFilter.value = tagFilter.value === tag ? null : tag
}
function clearFilters() {
  searchQuery.value = ''
  typeFilter.value = 'all'
  tagFilter.value = null
}

const filteredVisits = computed(() =>
  visits.value.filter((v) => {
    if (typeFilter.value !== 'all' && (v.visitType || '').toLowerCase() !== typeFilter.value) return false
    if (tagFilter.value && !(v.diagnosis || '').includes(tagFilter.value)) return false
    if (searchQuery.value.trim()) {
      const q = searchQuery.value.trim().toLowerCase()
      if (!`${v.diagnosis || ''} ${v.notes || ''}`.toLowerCase().includes(q)) return false
    }
    return true
  })
)

/* ------------------------- Expand / collapse visits ------------------------- */
// The newest visit opens by default; everything else stays collapsed so the
// timeline is easy to scan.
const openIds = ref([])
const isOpen = (id) => openIds.value.includes(id)
function toggleVisit(id) {
  openIds.value = isOpen(id) ? openIds.value.filter((x) => x !== id) : [...openIds.value, id]
}
const allOpen = computed(() => filteredVisits.value.length > 0 && filteredVisits.value.every((v) => isOpen(v.id)))
function toggleAll() {
  openIds.value = allOpen.value ? [] : visits.value.map((v) => v.id)
}

onMounted(() => {
  openIds.value = latestVisit.value ? [latestVisit.value.id] : []
  setTimeout(() => (loading.value = false), 500)
})

function printRecord() {
  window.print()
}
</script>

<template>
  <DashboardLayout>
    <!-- Top bar -->
    <div class="flex items-center justify-between gap-3 mb-5">
      <p v-if="isDoctor" class="text-sm text-slate-500 dark:text-[#7A9A95]">
        Patients / <span class="font-semibold text-slate-900 dark:text-[#E2ECE9]">{{ patientName }}</span>
      </p>
      <div v-else></div>
      <button type="button" @click="printRecord"
        class="print:hidden flex items-center gap-1.5 text-sm font-medium text-slate-600 dark:text-[#A3C2BD] hover:text-slate-900 dark:hover:text-white border border-slate-200 dark:border-[#1E3A35] hover:border-emerald-500/60 rounded-lg px-3.5 py-2 transition-colors focus:outline-none focus-visible:ring-2 focus-visible:ring-emerald-500">
        <PrinterIcon class="h-4 w-4" /> Print record
      </button>
    </div>

    <div class="grid gap-6 items-start" :class="isDoctor ? 'grid-cols-1 lg:grid-cols-[300px_1fr]' : 'grid-cols-1'">

      <!-- ============ LEFT: doctor's patient summary ============ -->
      <aside v-if="isDoctor" class="lg:sticky lg:top-6 rounded-2xl border border-slate-200 dark:border-[#1E3A35] bg-white dark:bg-[#122522] p-6 shadow-xl">
        <div class="flex items-center gap-3.5 mb-5">
          <BaseAvatar :name="patientName" size="lg" />
          <div>
            <h1 class="font-display text-lg font-bold text-slate-900 dark:text-white leading-tight">{{ patientName }}</h1>
            <p class="text-sm text-slate-500 dark:text-[#7A9A95]">{{ visits.length }} recorded {{ visits.length === 1 ? 'visit' : 'visits' }}</p>
          </div>
        </div>

        <div v-if="loading" class="space-y-2 mb-5">
          <SkeletonLine height="2.5rem" v-for="i in 3" :key="i" />
        </div>

        <template v-else>
          <!-- Allergies first: the thing a doctor must see before prescribing -->
          <div v-if="allergies.length" class="rounded-xl border border-red-200 dark:border-[#7F1D1D] bg-red-50 dark:bg-[#7F1D1D]/20 px-4 py-3 mb-4">
            <p class="flex items-center gap-1.5 text-sm font-semibold text-red-600 dark:text-[#F87171] mb-2">
              <ExclamationTriangleIcon class="h-4 w-4" /> Allergies
            </p>
            <div class="flex flex-wrap gap-1.5">
              <span v-for="a in allergies" :key="a" class="text-xs font-semibold text-red-600 dark:text-[#F87171] bg-red-100 dark:bg-[#7F1D1D]/40 rounded-full px-2.5 py-1">{{ a }}</span>
            </div>
          </div>

          <div class="rounded-xl border border-slate-200 dark:border-[#1E3A35] bg-slate-50 dark:bg-[#162E2A] px-4 py-3 mb-5">
            <p class="text-sm font-semibold text-slate-700 dark:text-[#A3C2BD] mb-2">Current medications</p>
            <ul v-if="currentMedications.length" class="space-y-2.5">
              <li v-for="m in currentMedications" :key="m.name" class="text-sm text-slate-800 dark:text-[#E2ECE9]">
                <p class="font-medium leading-snug">{{ m.name }}</p>
                <p class="text-xs text-slate-500 dark:text-[#7A9A95]">{{ m.dose }}</p>
              </li>
            </ul>
            <p v-else class="text-sm text-slate-500 dark:text-[#7A9A95]">None on record</p>
          </div>

          <dl class="mb-5 text-sm">
            <div class="flex items-center justify-between py-2.5">
              <dt class="text-slate-500 dark:text-[#7A9A95]">Total visits</dt>
              <dd class="font-semibold tabular-nums text-slate-900 dark:text-white">{{ visits.length }}</dd>
            </div>
            <div class="flex items-center justify-between py-2.5 border-t border-slate-200 dark:border-[#1E3A35]">
              <dt class="text-slate-500 dark:text-[#7A9A95]">Total billed</dt>
              <dd class="font-semibold tabular-nums text-slate-900 dark:text-white">{{ currency(totalBilled) }}</dd>
            </div>
            <div class="flex items-center justify-between py-2.5 border-t border-slate-200 dark:border-[#1E3A35]">
              <dt class="text-slate-500 dark:text-[#7A9A95]">Outstanding balance</dt>
              <dd class="font-semibold tabular-nums" :class="totalOutstanding > 0 ? 'text-red-600 dark:text-[#EF4444]' : 'text-emerald-600 dark:text-[#10B981]'">{{ currency(totalOutstanding) }}</dd>
            </div>
          </dl>

          <div v-if="conditionTags.length">
            <p class="text-sm font-semibold text-slate-700 dark:text-[#A3C2BD] mb-2">Conditions</p>
            <div class="flex flex-wrap gap-1.5">
              <button v-for="tag in conditionTags" :key="tag" type="button" @click="toggleTagFilter(tag)"
                :aria-pressed="tagFilter === tag"
                class="text-xs rounded-full border px-2.5 py-1 transition-colors focus:outline-none focus-visible:ring-2 focus-visible:ring-emerald-500"
                :class="tagFilter === tag
                  ? 'border-emerald-500 bg-emerald-50 text-emerald-700 dark:border-[#10B981] dark:bg-[#065F46]/40 dark:text-[#34D399]'
                  : 'border-slate-200 bg-slate-50 text-slate-600 hover:border-emerald-500/60 dark:border-[#1E3A35] dark:bg-[#162E2A] dark:text-[#A3C2BD]'"
              >{{ tag }}</button>
            </div>
          </div>
        </template>
      </aside>

      <!-- ============ RIGHT: main column ============ -->
      <div class="min-w-0">

        <!-- Patient header -->
        <div v-if="!isDoctor" class="flex items-center gap-4 mb-6">
          <BaseAvatar :name="patientName" size="lg" />
          <div>
            <h1 class="font-display text-2xl font-semibold text-slate-900 dark:text-white leading-tight">{{ patientName }}</h1>
            <p class="text-sm text-slate-500 dark:text-[#7A9A95]">Your medical history and records</p>
          </div>
        </div>

        <!-- Patient at-a-glance -->
        <template v-if="!isDoctor">
          <div v-if="loading" class="grid gap-3 sm:grid-cols-3 mb-6">
            <SkeletonLine v-for="i in 3" :key="i" height="5.5rem" />
          </div>

          <template v-else-if="visits.length">
            <div class="grid gap-3 sm:grid-cols-3 mb-4">
              <div class="rounded-2xl border border-slate-200 dark:border-[#1E3A35] bg-white dark:bg-[#122522] p-4">
                <p class="flex items-center gap-1.5 text-sm text-slate-500 dark:text-[#7A9A95]"><CalendarDaysIcon class="h-4 w-4" /> Last visit</p>
                <p class="font-display text-lg font-semibold text-slate-900 dark:text-white mt-1">{{ formatDate(latestVisit.date) }}</p>
                <p class="text-sm text-slate-500 dark:text-[#7A9A95] truncate">{{ latestVisit.doctor }}</p>
              </div>
              <div class="rounded-2xl border border-slate-200 dark:border-[#1E3A35] bg-white dark:bg-[#122522] p-4">
                <p class="flex items-center gap-1.5 text-sm text-slate-500 dark:text-[#7A9A95]"><ClipboardDocumentListIcon class="h-4 w-4" /> Medicines you take</p>
                <p class="font-display text-lg font-semibold text-slate-900 dark:text-white mt-1">{{ currentMedications.length }}</p>
                <p class="text-sm text-slate-500 dark:text-[#7A9A95]">{{ currentMedications.length ? 'Listed below' : 'None right now' }}</p>
              </div>
              <div class="rounded-2xl border border-slate-200 dark:border-[#1E3A35] bg-white dark:bg-[#122522] p-4">
                <p class="flex items-center gap-1.5 text-sm text-slate-500 dark:text-[#7A9A95]"><BanknotesIcon class="h-4 w-4" /> Balance due</p>
                <p class="font-display text-lg font-semibold mt-1" :class="totalOutstanding > 0 ? 'text-red-600 dark:text-[#F87171]' : 'text-emerald-600 dark:text-[#34D399]'">{{ currency(totalOutstanding) }}</p>
                <p class="text-sm text-slate-500 dark:text-[#7A9A95]">{{ totalOutstanding > 0 ? 'Please settle at reception' : 'You are all paid up' }}</p>
              </div>
            </div>

            <div v-if="allergies.length" class="flex flex-wrap items-center gap-2 rounded-2xl border border-red-200 dark:border-[#7F1D1D] bg-red-50 dark:bg-[#7F1D1D]/20 px-4 py-3 mb-4">
              <span class="flex items-center gap-1.5 text-sm font-semibold text-red-600 dark:text-[#F87171]">
                <ExclamationTriangleIcon class="h-4 w-4" /> Your allergies
              </span>
              <span v-for="a in allergies" :key="a" class="text-sm font-medium text-red-600 dark:text-[#F87171] bg-red-100 dark:bg-[#7F1D1D]/40 rounded-full px-3 py-0.5">{{ a }}</span>
            </div>

            <div class="rounded-2xl border border-slate-200 dark:border-[#1E3A35] bg-slate-50 dark:bg-[#162E2A] px-5 py-4 mb-8">
              <p class="text-sm font-semibold text-slate-700 dark:text-[#A3C2BD] mb-2">Your current medicines</p>
              <ul v-if="currentMedications.length" class="divide-y divide-slate-200 dark:divide-[#1E3A35]">
                <li v-for="m in currentMedications" :key="m.name" class="flex flex-wrap items-baseline justify-between gap-x-4 gap-y-0.5 py-2 text-slate-800 dark:text-[#E2ECE9]">
                  <span class="font-medium">{{ m.name }}</span>
                  <span class="text-sm text-slate-500 dark:text-[#7A9A95]">{{ m.dose }}</span>
                </li>
              </ul>
              <p v-else class="text-sm text-slate-500 dark:text-[#7A9A95]">You have no active prescriptions.</p>
            </div>
          </template>
        </template>

        <!-- Section heading -->
        <div class="flex flex-wrap items-end justify-between gap-3 mb-3.5">
          <div>
            <h2 class="font-display text-lg font-bold text-slate-900 dark:text-white">Visit history</h2>
            <p class="text-sm text-slate-500 dark:text-[#7A9A95]">Showing {{ filteredVisits.length }} of {{ visits.length }} {{ visits.length === 1 ? 'visit' : 'visits' }}</p>
          </div>
          <button v-if="filteredVisits.length > 1" type="button" @click="toggleAll"
            class="print:hidden text-sm font-medium text-emerald-700 dark:text-[#34D399] hover:underline focus:outline-none focus-visible:ring-2 focus-visible:ring-emerald-500 rounded">
            {{ allOpen ? 'Collapse all' : 'Expand all' }}
          </button>
        </div>

        <!-- Search + filters -->
        <div v-if="visits.length" class="print:hidden mb-4 space-y-3">
          <div class="flex flex-wrap items-center gap-2.5">
            <div class="relative flex-1 min-w-[220px]">
              <MagnifyingGlassIcon class="h-4 w-4 text-slate-400 dark:text-[#7A9A95] absolute left-3 top-1/2 -translate-y-1/2" />
              <input v-model="searchQuery" type="text" aria-label="Search visits"
                :placeholder="isDoctor ? 'Search diagnosis or notes…' : 'Search by condition or doctor’s notes…'"
                class="w-full bg-white dark:bg-[#162E2A] border border-slate-200 dark:border-[#1E3A35] rounded-xl pl-9 pr-3 py-2.5 text-sm text-slate-900 dark:text-white placeholder-slate-400 dark:placeholder-[#7A9A95] outline-none focus:border-emerald-500 dark:focus:border-[#10B981] focus:ring-2 focus:ring-emerald-500/20" />
            </div>
            <select v-model="typeFilter" aria-label="Filter by visit type"
              class="bg-white dark:bg-[#162E2A] border border-slate-200 dark:border-[#1E3A35] rounded-xl px-3.5 py-2.5 text-sm text-slate-900 dark:text-white outline-none capitalize focus:border-emerald-500 dark:focus:border-[#10B981]">
              <option v-for="t in visitTypes" :key="t" :value="t">{{ t === 'all' ? 'All visit types' : t }}</option>
            </select>
          </div>

          <!-- Condition chips: in the sidebar for doctors, here for patients -->
          <div v-if="!isDoctor && conditionTags.length" class="flex flex-wrap items-center gap-1.5">
            <span class="text-sm text-slate-500 dark:text-[#7A9A95] mr-1">Conditions</span>
            <button v-for="tag in conditionTags" :key="tag" type="button" @click="toggleTagFilter(tag)"
              :aria-pressed="tagFilter === tag"
              class="text-sm rounded-full border px-3 py-1 transition-colors focus:outline-none focus-visible:ring-2 focus-visible:ring-emerald-500"
              :class="tagFilter === tag
                ? 'border-emerald-500 bg-emerald-50 text-emerald-700 dark:border-[#10B981] dark:bg-[#065F46]/40 dark:text-[#34D399]'
                : 'border-slate-200 bg-white text-slate-600 hover:border-emerald-500/60 dark:border-[#1E3A35] dark:bg-[#162E2A] dark:text-[#A3C2BD]'"
            >{{ tag }}</button>
          </div>

          <p v-if="tagFilter || typeFilter !== 'all' || searchQuery" class="text-sm text-slate-500 dark:text-[#7A9A95]">
            Filters are on.
            <button type="button" @click="clearFilters" class="underline hover:text-slate-900 dark:hover:text-white">Clear all</button>
          </p>
        </div>

        <!-- Loading -->
        <div v-if="loading" class="space-y-4">
          <SkeletonLine v-for="i in 3" height="6rem" :key="i" />
        </div>

        <!-- Timeline -->
        <ol v-else-if="filteredVisits.length"
          class="relative pl-7 before:content-[''] before:absolute before:left-[6px] before:top-2 before:bottom-2 before:w-px before:bg-slate-200 dark:before:bg-[#1E3A35]">
          <li v-for="(visit, i) in filteredVisits" :key="visit.id" class="enc-in relative mb-4" :style="{ animationDelay: `${i * 60}ms` }">
            <span class="absolute -left-7 top-5 h-3 w-3 rounded-full bg-white dark:bg-[#122522] border-2" :class="toneFor(visit.visitType).dot"></span>

            <article class="rounded-2xl border bg-white dark:bg-[#122522] shadow-lg transition-colors"
              :class="isOpen(visit.id) ? 'border-emerald-500/40' : 'border-slate-200 dark:border-[#1E3A35]'">

              <!-- Always-visible summary row -->
              <button type="button" @click="toggleVisit(visit.id)" :aria-expanded="isOpen(visit.id)" :aria-controls="`visit-${visit.id}`"
                class="w-full text-left p-5 flex items-start gap-4 rounded-2xl focus:outline-none focus-visible:ring-2 focus-visible:ring-emerald-500">
                <div class="flex-1 min-w-0">
                  <div class="flex flex-wrap items-center gap-x-3 gap-y-1.5 mb-1.5">
                    <span class="flex items-center gap-1.5 text-sm text-slate-500 dark:text-[#7A9A95]">
                      <CalendarDaysIcon class="h-4 w-4" /> {{ formatDate(visit.date) }}
                    </span>
                    <span class="text-xs font-medium rounded-full px-2.5 py-1 capitalize" :class="toneFor(visit.visitType).badge">{{ visit.visitType }}</span>
                  </div>
                  <h3 class="font-display text-base font-semibold text-slate-900 dark:text-white leading-snug">{{ visit.diagnosis }}</h3>
                  <p class="flex items-center gap-1.5 text-sm text-slate-500 dark:text-[#7A9A95] mt-1">
                    <UserIcon class="h-4 w-4" /> {{ visit.doctor }}
                  </p>
                  <p v-if="!isOpen(visit.id)" class="print:hidden text-sm text-slate-600 dark:text-[#A3C2BD] mt-2 line-clamp-1">{{ visit.notes }}</p>
                </div>
                <ChevronDownIcon class="print:hidden h-5 w-5 shrink-0 mt-1 text-slate-400 dark:text-[#7A9A95] transition-transform duration-200"
                  :class="isOpen(visit.id) ? 'rotate-180' : ''" />
              </button>

              <!-- Details (always rendered when printing) -->
              <div :id="`visit-${visit.id}`" :class="isOpen(visit.id) ? 'block' : 'hidden print:block'">
                <div class="px-5 pb-5 pt-4 border-t border-slate-200 dark:border-[#1E3A35] space-y-5">

                  <!-- Vitals -->
                  <dl class="grid grid-cols-2 sm:grid-cols-4 rounded-xl bg-slate-50 dark:bg-[#162E2A] border border-slate-200 dark:border-[#1E3A35] overflow-hidden">
                    <div class="px-4 py-3 border-b sm:border-b-0 border-r border-slate-200 dark:border-[#1E3A35]">
                      <dt class="text-xs text-slate-500 dark:text-[#7A9A95] mb-0.5">Blood pressure</dt>
                      <dd class="text-base font-semibold tabular-nums text-slate-900 dark:text-white">{{ visit.vitals.bp }}</dd>
                    </div>
                    <div class="px-4 py-3 border-b sm:border-b-0 sm:border-r border-slate-200 dark:border-[#1E3A35]">
                      <dt class="text-xs text-slate-500 dark:text-[#7A9A95] mb-0.5">Heart rate</dt>
                      <dd class="text-base font-semibold tabular-nums text-slate-900 dark:text-white">{{ visit.vitals.hr }}</dd>
                    </div>
                    <div class="px-4 py-3 border-r border-slate-200 dark:border-[#1E3A35]">
                      <dt class="text-xs text-slate-500 dark:text-[#7A9A95] mb-0.5">Temperature</dt>
                      <dd class="text-base font-semibold tabular-nums text-slate-900 dark:text-white">{{ visit.vitals.temp }}</dd>
                    </div>
                    <div class="px-4 py-3">
                      <dt class="text-xs text-slate-500 dark:text-[#7A9A95] mb-0.5">Weight</dt>
                      <dd class="text-base font-semibold tabular-nums text-slate-900 dark:text-white">{{ visit.vitals.weight }}</dd>
                    </div>
                  </dl>

                  <!-- Notes -->
                  <div>
                    <p class="flex items-center gap-1.5 text-sm font-semibold text-slate-700 dark:text-[#A3C2BD] mb-1.5">
                      <DocumentTextIcon class="h-4 w-4" /> {{ isDoctor ? 'Clinical notes' : 'Doctor’s notes' }}
                    </p>
                    <p class="text-sm leading-relaxed text-slate-600 dark:text-[#A3C2BD] max-w-prose">{{ visit.notes }}</p>
                  </div>

                  <!-- Medications -->
                  <div v-if="medicationsForVisit(visit).length">
                    <p class="flex items-center gap-1.5 text-sm font-semibold text-slate-700 dark:text-[#A3C2BD] mb-2">
                      <ClipboardDocumentListIcon class="h-4 w-4" /> {{ isDoctor ? 'Medications prescribed' : 'Medicines prescribed' }}
                    </p>
                    <ul class="grid gap-2 sm:grid-cols-2">
                      <li v-for="m in medicationsForVisit(visit)" :key="m.name"
                        class="flex items-start justify-between gap-3 rounded-xl border border-slate-200 dark:border-[#1E3A35] bg-slate-50 dark:bg-[#162E2A] px-3.5 py-2.5">
                        <div class="min-w-0">
                          <p class="text-sm font-semibold text-slate-900 dark:text-white">{{ m.name }}</p>
                          <p class="text-xs text-slate-500 dark:text-[#7A9A95]">{{ m.dose }}</p>
                        </div>
                        <span class="shrink-0 rounded-full px-2.5 py-0.5 text-xs font-medium" :class="rxStatusFlag[m.status] || rxStatusFlag.fulfilled">{{ rxLabel(m.status) }}</span>
                      </li>
                    </ul>
                  </div>

                  <!-- Test results -->
                  <div v-if="visit.testResults?.length">
                    <p class="flex items-center gap-1.5 text-sm font-semibold text-slate-700 dark:text-[#A3C2BD] mb-2">
                      <BeakerIcon class="h-4 w-4" /> Test results
                    </p>
                    <ul class="grid gap-2 sm:grid-cols-2">
                      <li v-for="(t, ti) in visit.testResults" :key="ti"
                        class="flex items-start justify-between gap-3 rounded-xl border border-slate-200 dark:border-[#1E3A35] bg-slate-50 dark:bg-[#162E2A] px-3.5 py-2.5">
                        <div class="min-w-0">
                          <p class="text-sm font-semibold text-slate-900 dark:text-white">{{ t.name }}</p>
                          <p class="text-xs text-slate-500 dark:text-[#7A9A95]">{{ t.result }}</p>
                        </div>
                        <span class="shrink-0 rounded-full px-2.5 py-0.5 text-xs font-medium capitalize" :class="resultFlag[t.status] || resultFlag.pending">{{ resultLabel(t.status) }}</span>
                      </li>
                    </ul>
                  </div>
                </div>
              </div>
            </article>
          </li>
        </ol>

        <!-- Empty states -->
        <EmptyState v-else-if="visits.length" title="No visits match your filters" message="Try clearing the search, visit type or condition filter.">
          <template #icon><MagnifyingGlassIcon class="h-8 w-8 text-slate-400 dark:text-[#7A9A95]" /></template>
        </EmptyState>

        <EmptyState v-else title="No medical records yet" message="Your visit history will appear here after your first consultation.">
          <template #icon><HeartIcon class="h-8 w-8 text-slate-400 dark:text-[#7A9A95]" /></template>
        </EmptyState>
      </div>
    </div>
  </DashboardLayout>
</template>

<style scoped>
/* One entrance moment: the timeline staggers in on load. */
@keyframes enc-in {
  from { opacity: 0; transform: translateY(12px); }
  to { opacity: 1; transform: translateY(0); }
}
.enc-in {
  animation: enc-in 0.45s cubic-bezier(0.16, 1, 0.3, 1) both;
}

@media (prefers-reduced-motion: reduce) {
  .enc-in {
    animation: none !important;
    opacity: 1 !important;
    transform: none !important;
  }
}
</style>