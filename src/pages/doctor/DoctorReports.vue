<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import DashboardLayout from '@/layouts/DashboardLayout.vue'
import BaseButton from '@/components/ui/BaseButton.vue'
import BaseBadge from '@/components/ui/BaseBadge.vue'
import SkeletonCard from '@/components/skeletons/SkeletonCard.vue'
import { useUiStore } from '@/stores/ui'
import {
  ArrowDownTrayIcon,
  ArrowTopRightOnSquareIcon,
  BanknotesIcon,
  CalendarDaysIcon,
  ClipboardDocumentCheckIcon,
  CurrencyDollarIcon,
  DocumentTextIcon,
  EyeIcon,
  MagnifyingGlassIcon,
  ReceiptPercentIcon,
  UserGroupIcon,
  XMarkIcon,
} from '@heroicons/vue/24/outline'

const ui = useUiStore()

const loading = ref(true)
onMounted(() => setTimeout(() => (loading.value = false), 600))

/* ---------- helpers ---------- */
const SAMPLE_PDF = '/documents/sample.pdf' // Real file URL pele shudhu `file` ta bodle dao
const money = (n) => new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(n)
const fmtDate = (d) => new Date(d).toLocaleDateString('en-GB', { day: 'numeric', month: 'short', year: 'numeric' })

/* ---------- patient reports (kept with the doctor) ---------- */
const patientReports = ref([
  { id: 'pr_01', patient: 'Elena Whitfield', patientId: 'usr_004', title: 'HbA1c and lipid panel', type: 'Lab result', date: '2026-11-04', size: '240 KB', status: 'needs-review', file: SAMPLE_PDF },
  { id: 'pr_02', patient: 'Jonah Petrov', patientId: 'usr_010', title: '24-hour blood pressure monitoring', type: 'Progress note', date: '2026-11-03', size: '310 KB', status: 'needs-review', file: SAMPLE_PDF },
  { id: 'pr_03', patient: 'Marcus Webb', patientId: 'usr_012', title: 'Post-op cardiac echo', type: 'Imaging', date: '2026-11-01', size: '2.1 MB', status: 'needs-review', file: SAMPLE_PDF },
  { id: 'pr_04', patient: 'Naomi Reyes', patientId: 'usr_011', title: 'Spirometry results', type: 'Lab result', date: '2026-10-28', size: '180 KB', status: 'reviewed', file: SAMPLE_PDF },
  { id: 'pr_05', patient: 'Marcus Webb', patientId: 'usr_012', title: 'Cardiology referral letter', type: 'Referral', date: '2026-10-22', size: '95 KB', status: 'reviewed', file: SAMPLE_PDF },
  { id: 'pr_06', patient: 'Elena Whitfield', patientId: 'usr_004', title: 'Quarterly diabetes review', type: 'Progress note', date: '2026-10-10', size: '260 KB', status: 'reviewed', file: SAMPLE_PDF },
  { id: 'pr_07', patient: 'Jonah Petrov', patientId: 'usr_010', title: 'Discharge summary', type: 'Discharge', date: '2026-09-18', size: '420 KB', status: 'reviewed', file: SAMPLE_PDF },
])

const statusTone = { 'needs-review': 'warning', reviewed: 'success' }
const statusLabel = { 'needs-review': 'Needs review', reviewed: 'Reviewed' }

const prFilter = ref('all')
const prSearch = ref('')
const prChips = computed(() => [
  { key: 'all', label: 'All', count: patientReports.value.length },
  { key: 'needs-review', label: 'Needs review', count: patientReports.value.filter((r) => r.status === 'needs-review').length },
  { key: 'reviewed', label: 'Reviewed', count: patientReports.value.filter((r) => r.status === 'reviewed').length },
])
const filteredPatientReports = computed(() => {
  const q = prSearch.value.trim().toLowerCase()
  return patientReports.value.filter(
    (r) =>
      (prFilter.value === 'all' || r.status === prFilter.value) &&
      (!q || r.patient.toLowerCase().includes(q) || r.patientId.toLowerCase().includes(q) || r.title.toLowerCase().includes(q))
  )
})
const awaitingReview = computed(() => patientReports.value.filter((r) => r.status === 'needs-review').length)

function markReviewed(r) {
  r.status = 'reviewed'
  ui.toast({ type: 'success', title: 'Marked as reviewed', message: r.title })
}

/* ---------- net balance ---------- */
const PLATFORM_FEE = 0.12

const balance = { available: 6240, pending: 1180, nextPayout: '2026-11-15' }

const monthly = [
  { m: 'Jun', gross: 4100 },
  { m: 'Jul', gross: 4650 },
  { m: 'Aug', gross: 4300 },
  { m: 'Sep', gross: 5200 },
  { m: 'Oct', gross: 5750 },
  { m: 'Nov', gross: 3900 },
].map((x) => ({ ...x, net: Math.round(x.gross * (1 - PLATFORM_FEE)) }))

const maxNet = Math.max(...monthly.map((x) => x.net))

const current = computed(() => {
  const gross = monthly[monthly.length - 1].gross
  const refunds = 120
  const fee = Math.round(gross * PLATFORM_FEE)
  return { gross, fee, refunds, net: gross - fee - refunds }
})

const transactions = [
  { id: 'tx_01', patient: 'Elena Whitfield', kind: 'Consultation', date: '2026-11-05', amount: 85, status: 'paid' },
  { id: 'tx_02', patient: 'Jonah Petrov', kind: 'Follow-up', date: '2026-11-04', amount: 60, status: 'paid' },
  { id: 'tx_03', patient: 'Marcus Webb', kind: 'Consultation', date: '2026-11-03', amount: 120, status: 'pending' },
  { id: 'tx_04', patient: 'Naomi Reyes', kind: 'Refund', date: '2026-11-02', amount: -60, status: 'refunded' },
  { id: 'tx_05', patient: 'Elena Whitfield', kind: 'Follow-up', date: '2026-10-29', amount: 60, status: 'paid' },
]
const txTone = { paid: 'success', pending: 'warning', refunded: 'danger' }
const txLabel = { paid: 'Paid', pending: 'Pending', refunded: 'Refunded' }

const statements = [
  { id: 'st_10', period: 'October 2026', size: '198 KB', file: SAMPLE_PDF },
  { id: 'st_09', period: 'September 2026', size: '186 KB', file: SAMPLE_PDF },
  { id: 'st_08', period: 'August 2026', size: '174 KB', file: SAMPLE_PDF },
]

function requestPayout() {
  ui.toast({ type: 'success', title: 'Payout requested', message: `${money(balance.available)} will be sent to your account.` })
}

/* ---------- practice reports ---------- */
const practiceReports = [
  { id: 1, name: 'Appointments summary', period: 'October 2026', size: '176 KB', type: 'PDF' },
  { id: 2, name: 'Prescription activity log', period: 'Q3 2026', size: '420 KB', type: 'CSV' },
  { id: 3, name: 'Patient outcomes report', period: 'H1 2026', size: '1.1 MB', type: 'PDF' },
  { id: 4, name: 'Medication adherence summary', period: 'October 2026', size: '212 KB', type: 'PDF' },
  { id: 5, name: 'Referral and follow-up log', period: 'Q3 2026', size: '138 KB', type: 'CSV' },
  { id: 6, name: 'Annual earnings and tax statement', period: '2025', size: '305 KB', type: 'PDF' },
]
function download(r) {
  ui.toast({ type: 'success', title: 'Export started', message: `Preparing ${r.name}.${r.type.toLowerCase()}` })
}

/* ---------- tabs + summary tiles ---------- */
const tab = ref('patients')
const tabs = computed(() => [
  { key: 'patients', label: 'Patient reports', count: patientReports.value.length },
  { key: 'balance', label: 'Net balance', count: statements.length },
  { key: 'practice', label: 'Practice reports', count: practiceReports.length },
])

const tiles = computed(() => [
  { key: 'balance', label: 'Net balance', value: money(balance.available), icon: BanknotesIcon, go: () => (tab.value = 'balance') },
  { key: 'month', label: 'Earned this month', value: money(current.value.net), icon: CurrencyDollarIcon, go: () => (tab.value = 'balance') },
  { key: 'patients', label: 'Patient reports', value: patientReports.value.length, icon: UserGroupIcon, go: () => { tab.value = 'patients'; prFilter.value = 'all' } },
  { key: 'review', label: 'Awaiting review', value: awaitingReview.value, icon: ClipboardDocumentCheckIcon, accent: true, go: () => { tab.value = 'patients'; prFilter.value = 'needs-review' } },
])

/* ---------- PDF viewer ---------- */
const viewing = ref(null)
function openViewer(title, subtitle, file) {
  viewing.value = { title, subtitle, file }
}
function closeViewer() {
  viewing.value = null
}
const onKey = (e) => e.key === 'Escape' && closeViewer()
watch(viewing, (open) => {
  if (open) window.addEventListener('keydown', onKey)
  else window.removeEventListener('keydown', onKey)
})
onBeforeUnmount(() => window.removeEventListener('keydown', onKey))

const inputCls =
  'w-full rounded-lg border border-meridian-200 dark:border-white/10 bg-white dark:bg-white/5 py-2 pl-9 pr-3 text-sm text-meridian-800 dark:text-meridian-100 placeholder:text-meridian-400 focus:border-meridian-500 focus:outline-none focus:ring-2 focus:ring-meridian-500/20'
const iconBtn =
  'inline-flex items-center justify-center rounded-lg border border-meridian-200 p-2 text-meridian-500 transition hover:text-meridian-700 focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500 dark:border-white/10'
const viewBtn =
  'inline-flex items-center gap-1.5 rounded-lg border border-meridian-200 px-3 py-2 text-sm font-medium text-meridian-600 transition hover:border-meridian-400 hover:text-meridian-800 focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500 dark:border-white/10 dark:text-meridian-100'
const chipCls = (active) => [
  'inline-flex items-center gap-1.5 rounded-full border px-3 py-1.5 text-sm font-medium transition',
  active
    ? 'border-meridian-600 bg-meridian-600 text-white'
    : 'border-meridian-200 bg-white text-meridian-600 hover:bg-meridian-50 dark:border-white/10 dark:bg-white/5 dark:text-meridian-200 dark:hover:bg-white/10',
]
</script>

<template>
  <DashboardLayout>
    <!-- Header -->
    <div class="mb-6 flex flex-wrap items-end justify-between gap-4">
      <div>
        <h1 class="font-display text-2xl font-semibold text-meridian-900 dark:text-white">Reports</h1>
        <p class="mt-1 text-sm text-meridian-500 dark:text-meridian-300">
          Your patient reports, earnings and practice summaries in one place.
        </p>
      </div>
      <BaseButton variant="outline"><template #icon-left><ArrowDownTrayIcon class="h-5 w-5" /></template>Export all</BaseButton>
    </div>

    <!-- Summary tiles -->
    <div class="mb-6 grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
      <template v-if="loading"><SkeletonCard v-for="i in 4" :key="i" /></template>
      <template v-else>
        <button
          v-for="t in tiles"
          :key="t.key"
          type="button"
          @click="t.go()"
          class="flex items-center gap-4 rounded-2xl border border-meridian-100 bg-white p-4 text-left transition hover:border-meridian-300 hover:shadow-md focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500 dark:border-white/10 dark:bg-white/5 dark:hover:border-white/25"
        >
          <span
            :class="[
              'flex h-11 w-11 shrink-0 items-center justify-center rounded-xl',
              t.accent ? 'bg-[#ff6b5b]/10 text-[#ff6b5b]' : 'bg-meridian-50 text-meridian-600 dark:bg-white/10',
            ]"
          >
            <component :is="t.icon" class="h-5 w-5" />
          </span>
          <div>
            <p class="text-2xl font-semibold tabular-nums text-meridian-900 dark:text-white">{{ t.value }}</p>
            <p class="text-sm text-meridian-500 dark:text-meridian-300">{{ t.label }}</p>
          </div>
        </button>
      </template>
    </div>

    <!-- Tabs -->
    <div class="mb-5 inline-flex rounded-xl border border-meridian-100 bg-white p-1 dark:border-white/10 dark:bg-white/5" role="tablist">
      <button
        v-for="t in tabs"
        :key="t.key"
        type="button"
        role="tab"
        :aria-selected="tab === t.key"
        @click="tab = t.key"
        :class="[
          'inline-flex items-center gap-2 rounded-lg px-4 py-1.5 text-sm font-medium transition',
          tab === t.key
            ? 'bg-meridian-600 text-white shadow-sm'
            : 'text-meridian-600 hover:bg-meridian-50 dark:text-meridian-200 dark:hover:bg-white/10',
        ]"
      >
        {{ t.label }}
        <span :class="['text-xs tabular-nums', tab === t.key ? 'text-white/80' : 'text-meridian-400']">{{ t.count }}</span>
      </button>
    </div>

    <div v-if="loading" class="h-72 animate-pulse rounded-2xl bg-meridian-100 dark:bg-white/5" />

    <template v-else>
      <!-- Patient reports -->
      <section v-if="tab === 'patients'">
        <div class="mb-4 flex flex-wrap items-center justify-between gap-3">
          <div class="flex flex-wrap items-center gap-2">
            <button v-for="c in prChips" :key="c.key" type="button" @click="prFilter = c.key" :class="chipCls(prFilter === c.key)">
              {{ c.label }}
              <span :class="['text-xs tabular-nums', prFilter === c.key ? 'text-white/80' : 'text-meridian-400']">{{ c.count }}</span>
            </button>
          </div>

          <div class="relative w-full sm:w-72">
            <MagnifyingGlassIcon class="pointer-events-none absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-meridian-400" />
            <input v-model="prSearch" type="search" placeholder="Search patient, ID or report…" aria-label="Search patient reports" :class="inputCls" />
          </div>
        </div>

        <ul v-if="filteredPatientReports.length" class="card-base divide-y divide-meridian-100 dark:divide-white/5">
          <li v-for="r in filteredPatientReports" :key="r.id" class="flex flex-col gap-3 p-5 lg:flex-row lg:items-center lg:justify-between">
            <div class="flex min-w-0 items-center gap-3">
              <span class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl bg-meridian-50 text-meridian-600 dark:bg-white/5">
                <DocumentTextIcon class="h-5 w-5" />
              </span>
              <div class="min-w-0">
                <p class="truncate text-sm font-medium text-meridian-800 dark:text-meridian-100">{{ r.title }}</p>
                <p class="text-xs text-meridian-500">
                  <router-link :to="`/doctor/patients/${r.patientId}`" class="font-medium hover:underline">{{ r.patient }}</router-link>
                  · {{ r.patientId }} · {{ fmtDate(r.date) }} · {{ r.size }}
                </p>
              </div>
            </div>

            <div class="flex flex-wrap items-center gap-2 lg:shrink-0">
              <BaseBadge tone="neutral">{{ r.type }}</BaseBadge>
              <BaseBadge :tone="statusTone[r.status]">{{ statusLabel[r.status] }}</BaseBadge>
              <button
                v-if="r.status === 'needs-review'"
                type="button"
                @click="markReviewed(r)"
                class="inline-flex items-center gap-1.5 rounded-lg px-3 py-2 text-sm font-medium text-meridian-600 transition hover:bg-meridian-50 focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500 dark:text-meridian-200 dark:hover:bg-white/10"
              >
                <ClipboardDocumentCheckIcon class="h-4 w-4" />
                Mark reviewed
              </button>
              <button type="button" :class="viewBtn" @click="openViewer(r.title, `${r.patient} · ${r.type}`, r.file)">
                <EyeIcon class="h-4 w-4" />
                View PDF
              </button>
              <a :href="r.file" :download="`${r.title}.pdf`" :class="iconBtn" :aria-label="`Download ${r.title}`">
                <ArrowDownTrayIcon class="h-4 w-4" />
              </a>
            </div>
          </li>
        </ul>
        <div v-else class="flex flex-col items-center rounded-2xl border border-dashed border-meridian-200 px-6 py-14 text-center dark:border-white/10">
          <UserGroupIcon class="h-10 w-10 text-meridian-400" />
          <h2 class="mt-3 font-medium text-meridian-800 dark:text-meridian-100">No patient reports found</h2>
          <p class="mt-1 text-sm text-meridian-500">Try a different search or switch the filter.</p>
        </div>
      </section>

      <!-- Net balance -->
      <section v-else-if="tab === 'balance'" class="space-y-6">
        <!-- Balance + breakdown -->
        <div class="grid grid-cols-1 gap-4 lg:grid-cols-3">
          <div class="card-base flex flex-col justify-between p-6 lg:col-span-1">
            <div>
              <p class="text-sm text-meridian-500 dark:text-meridian-300">Available balance</p>
              <p class="mt-1 font-display text-3xl font-semibold tabular-nums text-meridian-900 dark:text-white">{{ money(balance.available) }}</p>
              <p class="mt-2 text-xs text-meridian-500">
                {{ money(balance.pending) }} pending clearance · Next payout {{ fmtDate(balance.nextPayout) }}
              </p>
            </div>
            <BaseButton class="mt-6" @click="requestPayout">
              <template #icon-left><BanknotesIcon class="h-5 w-5" /></template>
              Request payout
            </BaseButton>
          </div>

          <div class="card-base p-6 lg:col-span-2">
            <div class="mb-4 flex items-center justify-between">
              <h2 class="font-display text-base font-semibold text-meridian-900 dark:text-white">This month</h2>
              <ReceiptPercentIcon class="h-5 w-5 text-meridian-400" />
            </div>
            <dl class="grid grid-cols-2 gap-4 sm:grid-cols-4">
              <div>
                <dt class="text-xs text-meridian-500">Gross fees</dt>
                <dd class="mt-1 text-lg font-semibold tabular-nums text-meridian-900 dark:text-white">{{ money(current.gross) }}</dd>
              </div>
              <div>
                <dt class="text-xs text-meridian-500">Platform fee ({{ PLATFORM_FEE * 100 }}%)</dt>
                <dd class="mt-1 text-lg font-semibold tabular-nums text-meridian-900 dark:text-white">− {{ money(current.fee) }}</dd>
              </div>
              <div>
                <dt class="text-xs text-meridian-500">Refunds</dt>
                <dd class="mt-1 text-lg font-semibold tabular-nums text-meridian-900 dark:text-white">− {{ money(current.refunds) }}</dd>
              </div>
              <div>
                <dt class="text-xs text-meridian-500">Net earnings</dt>
                <dd class="mt-1 text-lg font-semibold tabular-nums text-meridian-600 dark:text-white">{{ money(current.net) }}</dd>
              </div>
            </dl>

            <!-- Monthly net bars -->
            <div class="mt-6 flex h-32 items-end gap-3 border-t border-meridian-100 pt-4 dark:border-white/10" role="img" aria-label="Net earnings for the last six months">
              <div v-for="(x, i) in monthly" :key="x.m" class="flex flex-1 flex-col items-center gap-2">
                <span class="text-[11px] tabular-nums text-meridian-500">{{ money(x.net) }}</span>
                <div
                  :class="['w-full rounded-t-lg transition-all', i === monthly.length - 1 ? 'bg-meridian-600' : 'bg-meridian-200 dark:bg-white/15']"
                  :style="{ height: `${Math.max((x.net / maxNet) * 64, 6)}px` }"
                />
                <span class="text-xs text-meridian-500">{{ x.m }}</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Transactions -->
        <div>
          <h2 class="mb-3 font-display text-base font-semibold text-meridian-900 dark:text-white">Recent transactions</h2>
          <ul class="card-base divide-y divide-meridian-100 dark:divide-white/5">
            <li v-for="t in transactions" :key="t.id" class="flex items-center justify-between gap-3 p-5">
              <div class="flex min-w-0 items-center gap-3">
                <span class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl bg-meridian-50 text-meridian-600 dark:bg-white/5">
                  <CalendarDaysIcon class="h-5 w-5" />
                </span>
                <div class="min-w-0">
                  <p class="truncate text-sm font-medium text-meridian-800 dark:text-meridian-100">{{ t.patient }}</p>
                  <p class="text-xs text-meridian-500">{{ t.kind }} · {{ fmtDate(t.date) }}</p>
                </div>
              </div>
              <div class="flex shrink-0 items-center gap-3">
                <BaseBadge :tone="txTone[t.status]">{{ txLabel[t.status] }}</BaseBadge>
                <p class="w-20 text-right text-sm font-semibold tabular-nums text-meridian-800 dark:text-meridian-100">{{ money(t.amount) }}</p>
              </div>
            </li>
          </ul>
        </div>

        <!-- Statements -->
        <div>
          <h2 class="mb-3 font-display text-base font-semibold text-meridian-900 dark:text-white">Monthly statements</h2>
          <ul class="card-base divide-y divide-meridian-100 dark:divide-white/5">
            <li v-for="s in statements" :key="s.id" class="flex flex-col gap-3 p-5 sm:flex-row sm:items-center sm:justify-between">
              <div class="flex min-w-0 items-center gap-3">
                <span class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl bg-meridian-50 text-meridian-600 dark:bg-white/5">
                  <DocumentTextIcon class="h-5 w-5" />
                </span>
                <div class="min-w-0">
                  <p class="truncate text-sm font-medium text-meridian-800 dark:text-meridian-100">Earnings statement</p>
                  <p class="text-xs text-meridian-500">{{ s.period }} · {{ s.size }}</p>
                </div>
              </div>
              <div class="flex items-center gap-2 sm:shrink-0">
                <button type="button" :class="viewBtn" @click="openViewer(`Earnings statement`, s.period, s.file)">
                  <EyeIcon class="h-4 w-4" />
                  View PDF
                </button>
                <a :href="s.file" :download="`Earnings statement ${s.period}.pdf`" :class="iconBtn" :aria-label="`Download statement ${s.period}`">
                  <ArrowDownTrayIcon class="h-4 w-4" />
                </a>
              </div>
            </li>
          </ul>
        </div>
      </section>

      <!-- Practice reports -->
      <section v-else>
        <div class="card-base divide-y divide-meridian-100 dark:divide-white/5">
          <div v-for="r in practiceReports" :key="r.id" class="flex items-center justify-between p-5">
            <div class="flex items-center gap-3">
              <span class="flex h-10 w-10 items-center justify-center rounded-xl bg-meridian-50 dark:bg-white/5 text-meridian-600">
                <DocumentTextIcon class="h-5 w-5" />
              </span>
              <div>
                <p class="text-sm font-medium text-meridian-800 dark:text-meridian-100">{{ r.name }}</p>
                <p class="text-xs text-meridian-500">{{ r.period }} · {{ r.size }}</p>
              </div>
            </div>
            <div class="flex items-center gap-3">
              <BaseBadge tone="neutral">{{ r.type }}</BaseBadge>
              <button @click="download(r)" class="text-xs font-medium text-meridian-600 hover:text-pulse-500">Download</button>
            </div>
          </div>
        </div>
      </section>
    </template>

    <!-- PDF viewer -->
    <Teleport to="body">
      <div v-if="viewing" class="fixed inset-0 z-50 flex items-center justify-center p-4" role="dialog" aria-modal="true" :aria-label="viewing.title">
        <div class="absolute inset-0 bg-black/50" @click="closeViewer" />

        <div class="relative flex h-[88vh] w-full max-w-4xl flex-col overflow-hidden rounded-2xl border border-meridian-100 bg-white shadow-xl dark:border-white/10 dark:bg-meridian-900">
          <header class="flex items-center justify-between gap-3 border-b border-meridian-100 px-5 py-4 dark:border-white/10">
            <div class="min-w-0">
              <h2 class="truncate font-display text-base font-semibold text-meridian-900 dark:text-white">{{ viewing.title }}</h2>
              <p class="truncate text-xs text-meridian-500">{{ viewing.subtitle }}</p>
            </div>
            <div class="flex shrink-0 items-center gap-2">
              <a :href="viewing.file" target="_blank" rel="noopener noreferrer" :class="iconBtn" aria-label="Open in new tab">
                <ArrowTopRightOnSquareIcon class="h-4 w-4" />
              </a>
              <a :href="viewing.file" :download="`${viewing.title}.pdf`" :class="iconBtn" aria-label="Download PDF">
                <ArrowDownTrayIcon class="h-4 w-4" />
              </a>
              <button type="button" @click="closeViewer" :class="iconBtn" aria-label="Close">
                <XMarkIcon class="h-4 w-4" />
              </button>
            </div>
          </header>

          <iframe :src="viewing.file" :title="viewing.title" class="h-full w-full flex-1 bg-meridian-50 dark:bg-white/5" />
        </div>
      </div>
    </Teleport>
  </DashboardLayout>
</template>