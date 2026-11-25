<script setup>
import { ref, computed, onMounted } from 'vue'
import DashboardLayout from '@/layouts/DashboardLayout.vue'
import DataTable from '@/components/tables/DataTable.vue'
import BaseAvatar from '@/components/ui/BaseAvatar.vue'
import BaseBadge from '@/components/ui/BaseBadge.vue'
import { MagnifyingGlassIcon, XMarkIcon } from '@heroicons/vue/24/outline'

const loading = ref(true)
const searchQuery = ref('')
const activeCategory = ref('all')

onMounted(() => setTimeout(() => (loading.value = false), 500))

const patients = ref([
  {
    id: 'usr_004',
    name: 'Elena Whitfield',
    image: '', // Add the patient's image URL or path here
    gender: 'female',
    age: 38,
    condition: 'Diabetes, Hypercholesterolemia',
    lastVisit: '2026-07-28',
    risk: 'moderate',
  },
  {
    id: 'usr_010',
    name: 'Jonah Petrov',
    image: '',
    gender: 'male',
    age: 52,
    condition: 'Hypertension',
    lastVisit: '2026-08-02',
    risk: 'high',
  },
  {
    id: 'usr_011',
    name: 'Naomi Reyes',
    image: '',
    gender: 'female',
    age: 29,
    condition: 'Asthma',
    lastVisit: '2026-08-04',
    risk: 'low',
  },
  {
    id: 'usr_012',
    name: 'Marcus Webb',
    image: '',
    gender: 'male',
    age: 61,
    condition: 'Post-op cardiac review',
    lastVisit: '2026-08-05',
    risk: 'high',
  },
])

// Category tabs (by gender)
const categories = [
  { key: 'all', label: 'All' },
  { key: 'male', label: 'Male' },
  { key: 'female', label: 'Female' },
  { key: 'others', label: 'Others' },
]

// Patient count per category
const categoryCounts = computed(() => {
  const counts = { all: patients.value.length, male: 0, female: 0, others: 0 }
  patients.value.forEach((p) => {
    if (counts[p.gender] !== undefined) counts[p.gender]++
  })
  return counts
})

// Category filter first, then search by name or patient ID
const filteredPatients = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()

  return patients.value.filter((patient) => {
    const matchesCategory =
      activeCategory.value === 'all' || patient.gender === activeCategory.value

    const matchesSearch =
      !query ||
      patient.name.toLowerCase().includes(query) ||
      patient.id.toLowerCase().includes(query)

    return matchesCategory && matchesSearch
  })
})

const isFiltering = computed(
  () => searchQuery.value.trim() !== '' || activeCategory.value !== 'all'
)

const columns = [
  { key: 'name', label: 'Patient', sortable: true },
  { key: 'age', label: 'Age', sortable: true },
  { key: 'condition', label: 'Primary condition' },
  { key: 'lastVisit', label: 'Last visit', sortable: true },
  { key: 'risk', label: 'Risk level' },
]

const riskTone = {
  high: 'danger',
  moderate: 'warning',
  low: 'success',
}

// Track images that fail to load
const failedImages = ref([])

function handleImageError(patientId) {
  if (!failedImages.value.includes(patientId)) {
    failedImages.value.push(patientId)
  }
}

function clearSearch() {
  searchQuery.value = ''
}

function setCategory(key) {
  activeCategory.value = key
}
</script>

<template>
  <DashboardLayout>
    <div class="mb-6">
      <h1 class="font-display text-2xl font-semibold text-meridian-900 dark:text-white mb-1">
        My Patients
      </h1>

      <p class="text-sm text-meridian-500 dark:text-meridian-400">
        Click a patient to view their full medical history and billing summary.
      </p>
    </div>

    <div class="mb-5">
      <!-- Category tabs (above search) -->
      <div
        class="mb-4 flex flex-wrap items-center gap-1 border-b border-meridian-200 dark:border-meridian-700"
        role="tablist"
        aria-label="Filter patients by gender"
      >
        <button
          v-for="category in categories"
          :key="category.key"
          type="button"
          role="tab"
          :aria-selected="activeCategory === category.key"
          @click="setCategory(category.key)"
          class="-mb-px inline-flex items-center gap-2 border-b-2 px-4 py-2.5 text-sm font-medium transition focus:outline-none"
          :class="
            activeCategory === category.key
              ? 'border-meridian-800 text-meridian-900 dark:border-white dark:text-white'
              : 'border-transparent text-meridian-500 hover:border-meridian-300 hover:text-meridian-800 dark:text-meridian-400 dark:hover:border-meridian-600 dark:hover:text-white'
          "
        >
          {{ category.label }}
          <span
            class="rounded-full px-2 py-0.5 text-xs"
            :class="
              activeCategory === category.key
                ? 'bg-meridian-800 text-white dark:bg-white dark:text-meridian-900'
                : 'bg-meridian-100 text-meridian-500 dark:bg-meridian-800 dark:text-meridian-400'
            "
          >
            {{ categoryCounts[category.key] }}
          </span>
        </button>
      </div>

      <!-- Search -->
      <div class="relative w-full sm:max-w-md">
        <MagnifyingGlassIcon
          class="pointer-events-none absolute left-3.5 top-1/2 h-5 w-5 -translate-y-1/2 text-meridian-400"
        />

        <input
          v-model="searchQuery"
          type="text"
          placeholder="Search by patient name or ID..."
          aria-label="Search patients by name or ID"
          class="w-full rounded-xl border border-meridian-200 bg-white py-3 pl-11 pr-11 text-sm text-meridian-800 placeholder:text-meridian-400 outline-none transition focus:border-meridian-400 focus:ring-2 focus:ring-meridian-100 dark:border-meridian-700 dark:bg-meridian-900 dark:text-white dark:placeholder:text-meridian-500 dark:focus:border-meridian-500 dark:focus:ring-meridian-800"
        />

        <button
          v-if="searchQuery"
          type="button"
          aria-label="Clear search"
          @click="clearSearch"
          class="absolute right-3 top-1/2 -translate-y-1/2 rounded-lg p-1 text-meridian-400 transition hover:bg-meridian-100 hover:text-meridian-700 dark:hover:bg-meridian-800 dark:hover:text-white"
        >
          <XMarkIcon class="h-4 w-4" />
        </button>
      </div>

      <p
        v-if="isFiltering && !loading"
        class="mt-2 text-xs text-meridian-500 dark:text-meridian-400"
      >
        {{ filteredPatients.length }}
        {{ filteredPatients.length === 1 ? 'patient' : 'patients' }} found
      </p>
    </div>

    <!-- Patients table -->
    <DataTable
      :columns="columns"
      :rows="filteredPatients"
      :loading="loading"
      empty-title="No patients found"
    >
      <template #cell-name="{ row }">
        <router-link
          :to="`/doctor/patients/${row.id}`"
          class="group flex w-fit items-center gap-3"
        >
          <!-- Patient image -->
          <BaseAvatar
            :name="row.name"
            :image="row.image"
            size="sm"
          />

          <div class="min-w-0">
            <p
              class="font-medium text-meridian-800 transition group-hover:text-meridian-600 group-hover:underline dark:text-meridian-100 dark:group-hover:text-white"
            >
              {{ row.name }}
            </p>

            <p class="mt-0.5 text-xs text-meridian-400 dark:text-meridian-500">
              ID: {{ row.id }}
            </p>
          </div>
        </router-link>
      </template>

      <template #cell-risk="{ value }">
        <BaseBadge :tone="riskTone[value]" class="capitalize">
          {{ value }}
        </BaseBadge>
      </template>
    </DataTable>
  </DashboardLayout>
</template>