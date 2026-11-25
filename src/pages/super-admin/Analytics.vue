<script setup>
import { ref, reactive, computed, watch, nextTick, onMounted, onBeforeUnmount } from 'vue'
import DashboardLayout from '@/layouts/DashboardLayout.vue'
import ChartCard from '@/components/cards/ChartCard.vue'
import BarChart from '@/components/charts/BarChart.vue'
import StatCard from '@/components/cards/StatCard.vue'
import SkeletonCard from '@/components/skeletons/SkeletonCard.vue'
import { useUiStore } from '@/stores/ui'
import {
  ChartBarIcon,
  UsersIcon,
  ClockIcon,
  ClipboardDocumentListIcon,
  SparklesIcon,
  TrophyIcon,
  BeakerIcon,
  ShieldExclamationIcon,
  PlusIcon,
  MagnifyingGlassIcon,
  XMarkIcon,
} from '@heroicons/vue/24/outline'

const ui = useUiStore()
const loading = ref(true)
onMounted(() => setTimeout(() => (loading.value = false), 600))

/* ---------- data ---------- */
const roles = [
  { label: 'Patients', value: 62 },
  { label: 'Doctors', value: 14 },
  { label: 'Pharmacists', value: 10 },
  { label: 'Caretakers', value: 14 },
]
const specialties = [
  { label: 'Cardiology', value: 4200 },
  { label: 'Endocrinology', value: 3600 },
  { label: 'Respiratory', value: 2100 },
  { label: 'Psychiatry', value: 1800 },
  { label: 'General', value: 5400 },
]

const rolesTotal = roles.reduce((s, r) => s + r.value, 0)
const topRole = roles.reduce((a, b) => (b.value > a.value ? b : a))
const rxTotal = specialties.reduce((s, r) => s + r.value, 0)
const topSpecialty = specialties.reduce((a, b) => (b.value > a.value ? b : a))
const pct = (v, t) => Math.round((v / t) * 100)

/* ---------- medicines & diseases ---------- */
const medicineForms = ['Tablet', 'Capsule', 'Syrup', 'Injection', 'Inhaler', 'Cream', 'Drops']
const medicineCategories = ['Antidiabetic', 'Antihypertensive', 'Statin', 'Bronchodilator', 'Antibiotic', 'Analgesic', 'Antacid', 'Other']
const diseaseCategories = ['Cardiology', 'Endocrinology', 'Respiratory', 'Psychiatry', 'Gastroenterology', 'General']

const medicines = ref([
  { id: 'med_01', name: 'Metformin', strength: '500 mg', form: 'Tablet', category: 'Antidiabetic' },
  { id: 'med_02', name: 'Amlodipine', strength: '5 mg', form: 'Tablet', category: 'Antihypertensive' },
  { id: 'med_03', name: 'Atorvastatin', strength: '20 mg', form: 'Tablet', category: 'Statin' },
  { id: 'med_04', name: 'Salbutamol', strength: '100 mcg', form: 'Inhaler', category: 'Bronchodilator' },
  { id: 'med_05', name: 'Omeprazole', strength: '20 mg', form: 'Capsule', category: 'Antacid' },
  { id: 'med_06', name: 'Amoxicillin', strength: '500 mg', form: 'Capsule', category: 'Antibiotic' },
  { id: 'med_07', name: 'Paracetamol', strength: '500 mg', form: 'Tablet', category: 'Analgesic' },
  { id: 'med_08', name: 'Insulin Glargine', strength: '100 IU/mL', form: 'Injection', category: 'Antidiabetic' },
])

const diseases = ref([
  { id: 'dis_01', name: 'Type 2 Diabetes', category: 'Endocrinology' },
  { id: 'dis_02', name: 'Hypertension', category: 'Cardiology' },
  { id: 'dis_03', name: 'Coronary Artery Disease', category: 'Cardiology' },
  { id: 'dis_04', name: 'Asthma', category: 'Respiratory' },
  { id: 'dis_05', name: 'Depression', category: 'Psychiatry' },
  { id: 'dis_06', name: 'GERD', category: 'Gastroenterology' },
  { id: 'dis_07', name: 'Hypothyroidism', category: 'Endocrinology' },
])

const medSearch = ref('')
const disSearch = ref('')

const filteredMedicines = computed(() => {
  const q = medSearch.value.trim().toLowerCase()
  return medicines.value.filter(
    (m) => !q || m.name.toLowerCase().includes(q) || m.category.toLowerCase().includes(q)
  )
})
const filteredDiseases = computed(() => {
  const q = disSearch.value.trim().toLowerCase()
  return diseases.value.filter(
    (d) => !q || d.name.toLowerCase().includes(q) || d.category.toLowerCase().includes(q)
  )
})

/* ---------- add modal ---------- */
const addType = ref(null) // 'medicine' | 'disease' | null
const form = reactive({ name: '', strength: '', dosageForm: 'Tablet', category: '' })
const formError = ref('')
const nameInput = ref(null)

const addTitle = computed(() => (addType.value === 'medicine' ? 'Add medicine' : 'Add disease'))
const addHint = computed(() =>
  addType.value === 'medicine'
    ? 'Add a medicine to the platform catalogue so it can be used in prescriptions.'
    : 'Add a disease or condition so it can be used across the platform.'
)

const inputCls =
  'w-full rounded-lg border border-meridian-200 dark:border-white/10 bg-white dark:bg-white/5 px-3 py-2 text-sm text-meridian-800 dark:text-meridian-100 placeholder:text-meridian-400 focus:border-meridian-500 focus:outline-none focus:ring-2 focus:ring-meridian-500/20'

function openAdd(type) {
  addType.value = type
  form.name = ''
  form.strength = ''
  form.dosageForm = 'Tablet'
  form.category = type === 'disease' ? 'General' : ''
  formError.value = ''
  nextTick(() => nameInput.value?.focus())
}
function closeAdd() {
  addType.value = null
}

function submitAdd() {
  const name = form.name.trim()
  if (!name) {
    formError.value = 'Enter a name.'
    return
  }

  if (addType.value === 'medicine') {
    const strength = form.strength.trim()
    const exists = medicines.value.some(
      (m) => m.name.toLowerCase() === name.toLowerCase() && m.strength.toLowerCase() === strength.toLowerCase()
    )
    if (exists) {
      formError.value = 'This medicine with the same strength is already in the catalogue.'
      return
    }
    medicines.value.unshift({
      id: `med_${Date.now()}`,
      name,
      strength,
      form: form.dosageForm,
      category: form.category.trim() || 'Other',
    })
    ui.toast({ type: 'success', title: 'Medicine added' })
  } else {
    const exists = diseases.value.some((d) => d.name.toLowerCase() === name.toLowerCase())
    if (exists) {
      formError.value = 'This disease is already in the list.'
      return
    }
    diseases.value.unshift({ id: `dis_${Date.now()}`, name, category: form.category })
    ui.toast({ type: 'success', title: 'Disease added' })
  }
  closeAdd()
}

const onKey = (e) => e.key === 'Escape' && closeAdd()
watch(addType, (open) => {
  if (open) window.addEventListener('keydown', onKey)
  else window.removeEventListener('keydown', onKey)
})
onBeforeUnmount(() => window.removeEventListener('keydown', onKey))
</script>

<template>
  <DashboardLayout>
    <!-- Header -->
    <div class="mb-6">
      <h1 class="font-display text-2xl font-semibold text-meridian-900 dark:text-white">Analytics</h1>
      <p class="mt-1 text-sm text-meridian-500 dark:text-meridian-300">How the platform is performing, at a glance.</p>
    </div>

    <!-- Stat cards -->
    <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4 mb-6">
      <template v-if="loading"><SkeletonCard v-for="i in 4" :key="i" /></template>
      <template v-else>
        <StatCard label="Avg. session length" value="6m 42s" delta="+8%" tone="meridian"><template #icon><ClockIcon class="h-5 w-5" /></template></StatCard>
        <StatCard label="Daily active users" value="4,209" delta="+3.1%" tone="meridian"><template #icon><UsersIcon class="h-5 w-5" /></template></StatCard>
        <StatCard label="Adherence rate" value="87.4%" delta="+2.4%" tone="pulse"><template #icon><ChartBarIcon class="h-5 w-5" /></template></StatCard>
        <StatCard label="Rx processed" value="48,920" delta="+15.7%" tone="pulse"><template #icon><ClipboardDocumentListIcon class="h-5 w-5" /></template></StatCard>
      </template>
    </div>

    <!-- Medicines + Diseases -->
    <div v-if="loading" class="grid grid-cols-1 lg:grid-cols-2 gap-4 mb-4">
      <div class="h-96 rounded-2xl bg-meridian-100 dark:bg-white/5 animate-pulse" />
      <div class="h-96 rounded-2xl bg-meridian-100 dark:bg-white/5 animate-pulse" />
    </div>
    <div v-else class="grid grid-cols-1 lg:grid-cols-2 gap-4 mb-4">
      <!-- Medicines -->
      <section class="flex flex-col rounded-2xl border border-meridian-100 dark:border-white/10 bg-white dark:bg-white/5">
        <header class="flex items-center justify-between gap-3 border-b border-meridian-100 dark:border-white/10 p-5">
          <div class="flex items-center gap-3">
            <span class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl bg-meridian-50 dark:bg-white/10 text-meridian-600">
              <BeakerIcon class="h-5 w-5" />
            </span>
            <div>
              <h2 class="font-display text-base font-semibold text-meridian-900 dark:text-white">Medicines</h2>
              <p class="text-sm text-meridian-500 dark:text-meridian-300">{{ medicines.length }} in the catalogue</p>
            </div>
          </div>
          <button
            type="button"
            @click="openAdd('medicine')"
            class="inline-flex shrink-0 items-center gap-1.5 rounded-lg bg-meridian-600 px-3 py-2 text-sm font-medium text-white transition hover:bg-meridian-700 focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500"
          >
            <PlusIcon class="h-4 w-4" />
            Add medicine
          </button>
        </header>

        <div class="px-5 pt-4">
          <div class="relative">
            <MagnifyingGlassIcon class="pointer-events-none absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-meridian-400" />
            <input v-model="medSearch" type="search" placeholder="Search medicines…" aria-label="Search medicines" :class="[inputCls, 'pl-9']" />
          </div>
        </div>

        <ul v-if="filteredMedicines.length" class="max-h-80 divide-y divide-meridian-100 dark:divide-white/10 overflow-y-auto px-5 pb-2 pt-1">
          <li v-for="m in filteredMedicines" :key="m.id" class="flex items-center justify-between gap-3 py-3">
            <div class="min-w-0">
              <p class="truncate font-medium text-meridian-900 dark:text-meridian-50">{{ m.name }}</p>
              <p class="text-xs text-meridian-500 dark:text-meridian-300">{{ [m.strength, m.form].filter(Boolean).join(', ') }}</p>
            </div>
            <span class="shrink-0 rounded-md bg-meridian-50 dark:bg-white/5 px-2 py-0.5 text-xs font-medium text-meridian-700 dark:text-meridian-200">
              {{ m.category }}
            </span>
          </li>
        </ul>
        <p v-else class="px-5 py-10 text-center text-sm text-meridian-500">No medicines match your search.</p>
      </section>

      <!-- Diseases -->
      <section class="flex flex-col rounded-2xl border border-meridian-100 dark:border-white/10 bg-white dark:bg-white/5">
        <header class="flex items-center justify-between gap-3 border-b border-meridian-100 dark:border-white/10 p-5">
          <div class="flex items-center gap-3">
            <span class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl bg-[#ff6b5b]/10 text-[#ff6b5b]">
              <ShieldExclamationIcon class="h-5 w-5" />
            </span>
            <div>
              <h2 class="font-display text-base font-semibold text-meridian-900 dark:text-white">Diseases</h2>
              <p class="text-sm text-meridian-500 dark:text-meridian-300">{{ diseases.length }} conditions tracked</p>
            </div>
          </div>
          <button
            type="button"
            @click="openAdd('disease')"
            class="inline-flex shrink-0 items-center gap-1.5 rounded-lg bg-meridian-600 px-3 py-2 text-sm font-medium text-white transition hover:bg-meridian-700 focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500"
          >
            <PlusIcon class="h-4 w-4" />
            Add disease
          </button>
        </header>

        <div class="px-5 pt-4">
          <div class="relative">
            <MagnifyingGlassIcon class="pointer-events-none absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-meridian-400" />
            <input v-model="disSearch" type="search" placeholder="Search diseases…" aria-label="Search diseases" :class="[inputCls, 'pl-9']" />
          </div>
        </div>

        <ul v-if="filteredDiseases.length" class="max-h-80 divide-y divide-meridian-100 dark:divide-white/10 overflow-y-auto px-5 pb-2 pt-1">
          <li v-for="d in filteredDiseases" :key="d.id" class="flex items-center justify-between gap-3 py-3">
            <p class="min-w-0 truncate font-medium text-meridian-900 dark:text-meridian-50">{{ d.name }}</p>
            <span class="shrink-0 rounded-md bg-meridian-50 dark:bg-white/5 px-2 py-0.5 text-xs font-medium text-meridian-700 dark:text-meridian-200">
              {{ d.category }}
            </span>
          </li>
        </ul>
        <p v-else class="px-5 py-10 text-center text-sm text-meridian-500">No diseases match your search.</p>
      </section>
    </div>

    <!-- Specialty chart + insight callouts -->
    <div class="grid grid-cols-1 lg:grid-cols-3 gap-4">
      <div class="lg:col-span-2">
        <ChartCard title="Prescriptions by specialty" subtitle="This quarter">
          <BarChart
            :labels="specialties.map((s) => s.label)"
            :datasets="[{ label: 'Rx count', data: specialties.map((s) => s.value), color: '#ff6b5b' }]"
            horizontal
          />
        </ChartCard>
      </div>

      <div class="flex flex-col gap-4">
        <div class="flex-1 rounded-2xl border border-meridian-100 dark:border-white/10 bg-white dark:bg-white/5 p-5">
          <span class="flex h-9 w-9 items-center justify-center rounded-lg bg-meridian-50 dark:bg-white/10 text-meridian-600"><TrophyIcon class="h-5 w-5" /></span>
          <div class="mt-3 text-sm text-meridian-500 dark:text-meridian-300">Top specialty</div>
          <div class="text-lg font-semibold text-meridian-900 dark:text-white">{{ topSpecialty.label }}</div>
          <div class="mt-1 text-sm text-meridian-500 dark:text-meridian-300">
            {{ pct(topSpecialty.value, rxTotal) }}% of this quarter's prescriptions
          </div>
        </div>

        <div class="flex-1 rounded-2xl border border-meridian-100 dark:border-white/10 bg-white dark:bg-white/5 p-5">
          <span class="flex h-9 w-9 items-center justify-center rounded-lg bg-[#ff6b5b]/10 text-[#ff6b5b]"><SparklesIcon class="h-5 w-5" /></span>
          <div class="mt-3 text-sm text-meridian-500 dark:text-meridian-300">Largest audience</div>
          <div class="text-lg font-semibold text-meridian-900 dark:text-white">{{ topRole.label }}</div>
          <div class="mt-1 text-sm text-meridian-500 dark:text-meridian-300">
            {{ pct(topRole.value, rolesTotal) }}% of active sessions
          </div>
        </div>
      </div>
    </div>

    <!-- Add modal -->
    <Teleport to="body">
      <div
        v-if="addType"
        class="fixed inset-0 z-50 flex items-center justify-center p-4"
        role="dialog"
        aria-modal="true"
        :aria-label="addTitle"
      >
        <div class="absolute inset-0 bg-black/50" @click="closeAdd" />

        <form
          class="relative w-full max-w-md rounded-2xl border border-meridian-100 dark:border-white/10 bg-white dark:bg-meridian-900 p-6 shadow-xl"
          @submit.prevent="submitAdd"
        >
          <button
            type="button"
            @click="closeAdd"
            class="absolute right-3 top-3 rounded-full p-1.5 text-meridian-400 hover:text-meridian-700 dark:hover:text-white"
            aria-label="Close"
          >
            <XMarkIcon class="h-5 w-5" />
          </button>

          <h2 class="pr-8 font-display text-lg font-semibold text-meridian-900 dark:text-white">{{ addTitle }}</h2>
          <p class="mt-1 text-sm text-meridian-500 dark:text-meridian-300">{{ addHint }}</p>

          <div class="mt-5 space-y-4">
            <div>
              <label for="add-name" class="mb-1.5 block text-sm font-medium text-meridian-700 dark:text-meridian-200">Name</label>
              <input
                id="add-name"
                ref="nameInput"
                v-model="form.name"
                type="text"
                :placeholder="addType === 'medicine' ? 'e.g. Ibuprofen' : 'e.g. Migraine'"
                :class="inputCls"
              />
            </div>

            <template v-if="addType === 'medicine'">
              <div class="grid grid-cols-2 gap-3">
                <div>
                  <label for="add-strength" class="mb-1.5 block text-sm font-medium text-meridian-700 dark:text-meridian-200">Strength</label>
                  <input id="add-strength" v-model="form.strength" type="text" placeholder="e.g. 400 mg" :class="inputCls" />
                </div>
                <div>
                  <label for="add-form" class="mb-1.5 block text-sm font-medium text-meridian-700 dark:text-meridian-200">Form</label>
                  <select id="add-form" v-model="form.dosageForm" :class="inputCls">
                    <option v-for="f in medicineForms" :key="f" :value="f">{{ f }}</option>
                  </select>
                </div>
              </div>
              <div>
                <label for="add-med-category" class="mb-1.5 block text-sm font-medium text-meridian-700 dark:text-meridian-200">Category</label>
                <input
                  id="add-med-category"
                  v-model="form.category"
                  type="text"
                  list="medicine-categories"
                  placeholder="Choose or type a category"
                  :class="inputCls"
                />
                <datalist id="medicine-categories">
                  <option v-for="c in medicineCategories" :key="c" :value="c" />
                </datalist>
              </div>
            </template>

            <div v-else>
              <label for="add-dis-category" class="mb-1.5 block text-sm font-medium text-meridian-700 dark:text-meridian-200">Category</label>
              <select id="add-dis-category" v-model="form.category" :class="inputCls">
                <option v-for="c in diseaseCategories" :key="c" :value="c">{{ c }}</option>
              </select>
            </div>

            <p v-if="formError" class="text-sm text-red-600" role="alert">{{ formError }}</p>
          </div>

          <div class="mt-6 flex justify-end gap-2">
            <button
              type="button"
              @click="closeAdd"
              class="rounded-lg border border-meridian-200 dark:border-white/10 px-4 py-2 text-sm font-medium text-meridian-600 dark:text-meridian-200 transition hover:bg-meridian-50 dark:hover:bg-white/10"
            >
              Cancel
            </button>
            <button
              type="submit"
              class="rounded-lg bg-meridian-600 px-4 py-2 text-sm font-medium text-white transition hover:bg-meridian-700 focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500"
            >
              {{ addTitle }}
            </button>
          </div>
        </form>
      </div>
    </Teleport>
  </DashboardLayout>
</template>