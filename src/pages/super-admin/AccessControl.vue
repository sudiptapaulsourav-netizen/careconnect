<script setup>
import { ref, computed, onMounted } from 'vue'
import { onBeforeRouteLeave } from 'vue-router'
import DashboardLayout from '@/layouts/DashboardLayout.vue'
import BaseBadge from '@/components/ui/BaseBadge.vue'
import BaseButton from '@/components/ui/BaseButton.vue'
import BaseModal from '@/components/modals/BaseModal.vue'
import { useUiStore } from '@/stores/ui'
import {
  ShieldCheckIcon,
  UsersIcon,
  LockClosedIcon,
  HeartIcon,
  BeakerIcon,
  UserGroupIcon,
  ExclamationTriangleIcon,
} from '@heroicons/vue/24/outline'

const ui = useUiStore()
const loading = ref(true)
onMounted(() => setTimeout(() => (loading.value = false), 500))

/* ---------- data ---------- */
const roleIcon = { doctor: HeartIcon, pharmacist: BeakerIcon, caretaker: UserGroupIcon }

const initialRoles = [
  {
    id: 'doctor', name: 'Doctor', color: 'info', members: 640,
    description: 'Licensed clinicians who diagnose and treat patients.',
    permissions: [
      { key: 'issue_rx', label: 'Issue prescriptions', description: 'Create and sign prescriptions for patients.', group: 'Clinical work', enabled: true },
      { key: 'manage_appointments', label: 'Manage appointments', description: 'Accept, reschedule and cancel appointments.', group: 'Clinical work', enabled: true },
      { key: 'view_records', label: 'View patient records', description: 'Open the full medical history of a patient.', group: 'Patient data', sensitive: true, enabled: true },
      { key: 'export_reports', label: 'Export reports', description: 'Download reports that contain patient information.', group: 'Patient data', sensitive: true, enabled: false },
    ],
  },
  {
    id: 'pharmacist', name: 'Pharmacist', color: 'success', members: 312,
    description: 'Pharmacy staff who dispense medicines and manage stock.',
    permissions: [
      { key: 'fulfill_rx', label: 'Fulfill prescriptions', description: 'Dispense medicines against a valid prescription.', group: 'Dispensing and stock', enabled: true },
      { key: 'scan_qr', label: 'Scan prescription QR', description: 'Verify a prescription by scanning its QR code.', group: 'Dispensing and stock', enabled: true },
      { key: 'manage_inventory', label: 'Manage inventory', description: 'Add, edit and remove stock items.', group: 'Dispensing and stock', enabled: true },
      { key: 'view_records', label: 'View patient records', description: 'Open the full medical history of a patient.', group: 'Patient data', sensitive: true, enabled: false },
    ],
  },
  {
    id: 'caretaker', name: 'Caretaker', color: 'neutral', members: 2105,
    description: "Family members or carers who support a patient's daily care.",
    permissions: [
      { key: 'view_schedule', label: 'View care recipient schedule', description: "See the recipient's medicine and appointment schedule.", group: 'Daily care', enabled: true },
      { key: 'log_doses', label: 'Log doses on behalf of recipient', description: 'Mark doses as taken for the person they care for.', group: 'Daily care', enabled: true },
      { key: 'book_appointments', label: 'Book appointments', description: 'Book visits with doctors for the recipient.', group: 'Bookings and records', enabled: true },
      { key: 'view_prescriptions', label: 'View prescriptions', description: "Read the recipient's active prescriptions.", group: 'Bookings and records', sensitive: true, enabled: false },
    ],
  },
]

/* `enabled` = what the admin is editing, `saved` = what is live right now */
const roles = ref(
  initialRoles.map((r) => ({ ...r, permissions: r.permissions.map((p) => ({ ...p, saved: p.enabled })) }))
)

/* ---------- helpers ---------- */
const enabledCount = (role) => role.permissions.filter((p) => p.enabled).length
const changed = (p) => p.enabled !== p.saved
const roleChanges = (role) => role.permissions.filter(changed)
const plural = (n, one, many) => (n === 1 ? one : many)

/* ---------- selected role ---------- */
const selectedId = ref('doctor')
const selected = computed(() => roles.value.find((r) => r.id === selectedId.value))
const selectedGroups = computed(() => {
  const map = new Map()
  for (const p of selected.value.permissions) {
    if (!map.has(p.group)) map.set(p.group, [])
    map.get(p.group).push(p)
  }
  return [...map].map(([name, items]) => ({ name, items }))
})

function setAll(role, value) {
  role.permissions.forEach((p) => (p.enabled = value))
}

/* ---------- summary ---------- */
const allPerms = computed(() => roles.value.flatMap((r) => r.permissions))
const totalOn = computed(() => allPerms.value.filter((p) => p.enabled).length)
const sensitiveOn = computed(() => allPerms.value.filter((p) => p.sensitive && p.enabled).length)

const tiles = computed(() => [
  { label: 'Roles', value: roles.value.length, icon: UsersIcon },
  { label: 'Permissions on', value: `${totalOn.value} of ${allPerms.value.length}`, icon: ShieldCheckIcon },
  { label: 'Sensitive access granted', value: sensitiveOn.value, icon: LockClosedIcon, accent: true },
])

/* ---------- unsaved changes ---------- */
const changes = computed(() => roles.value.flatMap((r) => roleChanges(r).map((p) => ({ role: r, perm: p }))))
const changeCount = computed(() => changes.value.length)
const changedRoles = computed(() => roles.value.filter((r) => roleChanges(r).length))
const affectedMembers = computed(() => changedRoles.value.reduce((s, r) => s + r.members, 0))
const grantsSensitive = computed(() => changes.value.some((c) => c.perm.sensitive && c.perm.enabled))

function discard() {
  roles.value.forEach((r) => r.permissions.forEach((p) => (p.enabled = p.saved)))
  ui.toast({ type: 'info', title: 'Changes discarded', message: 'Permissions are back to their saved state.' })
}

const reviewOpen = ref(false)
const saving = ref(false)
async function save() {
  const n = changeCount.value
  saving.value = true
  await new Promise((r) => setTimeout(r, 600)) // replace with real API call
  roles.value.forEach((r) => r.permissions.forEach((p) => (p.saved = p.enabled)))
  saving.value = false
  reviewOpen.value = false
  ui.toast({ type: 'success', title: 'Permissions updated', message: `${n} ${plural(n, 'change', 'changes')} applied.` })
}

/* warn before leaving the page with unsaved edits */
onBeforeRouteLeave(() =>
  changeCount.value ? window.confirm('You have unsaved permission changes. Leave without saving?') : true
)
</script>

<template>
  <DashboardLayout>
    <!-- Header -->
    <div class="mb-6">
      <h1 class="font-display text-2xl font-semibold text-meridian-900 dark:text-white">Access Control</h1>
      <p class="mt-1 text-sm text-meridian-500 dark:text-meridian-300">
        Pick a role, then choose what it can do. Nothing goes live until you review and save.
      </p>
    </div>

    <!-- Loading -->
    <template v-if="loading">
      <div class="mb-6 grid grid-cols-1 gap-4 sm:grid-cols-3">
        <div v-for="i in 3" :key="i" class="h-20 animate-pulse rounded-2xl bg-meridian-100 dark:bg-white/5" />
      </div>
      <div class="grid gap-6 lg:grid-cols-[320px_1fr]">
        <div class="h-72 animate-pulse rounded-2xl bg-meridian-100 dark:bg-white/5" />
        <div class="h-96 animate-pulse rounded-2xl bg-meridian-100 dark:bg-white/5" />
      </div>
    </template>

    <template v-else>
      <!-- Summary -->
      <div class="mb-6 grid grid-cols-1 gap-4 sm:grid-cols-3">
        <div
          v-for="t in tiles"
          :key="t.label"
          class="flex items-center gap-4 rounded-2xl border border-meridian-100 bg-white p-4 dark:border-white/10 dark:bg-white/5"
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
        </div>
      </div>

      <div class="grid gap-6 lg:grid-cols-[320px_1fr]">
        <!-- Role list -->
        <nav aria-label="Roles" class="grid h-fit grid-cols-1 gap-3 sm:grid-cols-3 lg:grid-cols-1">
          <button
            v-for="r in roles"
            :key="r.id"
            type="button"
            :aria-current="selectedId === r.id"
            @click="selectedId = r.id"
            :class="[
              'w-full rounded-xl border p-4 text-left transition focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500',
              selectedId === r.id
                ? 'border-meridian-500 bg-meridian-50 shadow-sm dark:bg-white/10'
                : 'border-meridian-100 bg-white hover:border-meridian-300 dark:border-white/10 dark:bg-white/5 dark:hover:border-white/25',
            ]"
          >
            <div class="flex items-center gap-3">
              <span
                :class="[
                  'flex h-10 w-10 shrink-0 items-center justify-center rounded-xl transition',
                  selectedId === r.id ? 'bg-meridian-600 text-white' : 'bg-meridian-50 text-meridian-600 dark:bg-white/10',
                ]"
              >
                <component :is="roleIcon[r.id]" class="h-5 w-5" />
              </span>
              <div class="min-w-0 flex-1">
                <div class="flex items-center justify-between gap-2">
                  <p class="truncate font-medium text-meridian-900 dark:text-white">{{ r.name }}</p>
                  <span
                    v-if="roleChanges(r).length"
                    class="shrink-0 rounded-full bg-pulse-400/10 px-2 py-0.5 text-xs font-medium text-pulse-600"
                  >Unsaved</span>
                </div>
                <p class="text-xs text-meridian-500 dark:text-meridian-300">{{ r.members.toLocaleString() }} members</p>
              </div>
            </div>

            <div class="mt-3">
              <div class="flex justify-between text-xs text-meridian-500 dark:text-meridian-300">
                <span>Access</span>
                <span class="tabular-nums">{{ enabledCount(r) }} of {{ r.permissions.length }}</span>
              </div>
              <div class="mt-1.5 h-1.5 overflow-hidden rounded-full bg-meridian-100 dark:bg-white/10">
                <div
                  class="h-full rounded-full bg-meridian-600 transition-all duration-300"
                  :style="{ width: (enabledCount(r) / r.permissions.length) * 100 + '%' }"
                />
              </div>
            </div>
          </button>
        </nav>

        <!-- Permission editor -->
        <section class="rounded-2xl border border-meridian-100 bg-white dark:border-white/10 dark:bg-white/5">
          <header class="flex flex-wrap items-start justify-between gap-4 border-b border-meridian-100 p-5 dark:border-white/10 sm:p-6">
            <div class="flex items-start gap-3">
              <span class="flex h-11 w-11 shrink-0 items-center justify-center rounded-xl bg-meridian-50 text-meridian-600 dark:bg-white/10">
                <component :is="roleIcon[selected.id]" class="h-5 w-5" />
              </span>
              <div>
                <div class="flex items-center gap-2">
                  <h2 class="font-display text-lg font-semibold text-meridian-900 dark:text-white">{{ selected.name }}</h2>
                  <BaseBadge :tone="selected.color">{{ enabledCount(selected) }}/{{ selected.permissions.length }}</BaseBadge>
                </div>
                <p class="mt-0.5 max-w-md text-sm text-meridian-500 dark:text-meridian-300">{{ selected.description }}</p>
              </div>
            </div>

            <div class="flex items-center gap-4 text-sm font-medium">
              <button
                type="button"
                :disabled="enabledCount(selected) === selected.permissions.length"
                @click="setAll(selected, true)"
                class="text-meridian-600 hover:underline disabled:cursor-not-allowed disabled:opacity-40 disabled:no-underline"
              >Enable all</button>
              <button
                type="button"
                :disabled="enabledCount(selected) === 0"
                @click="setAll(selected, false)"
                class="text-meridian-600 hover:underline disabled:cursor-not-allowed disabled:opacity-40 disabled:no-underline"
              >Disable all</button>
            </div>
          </header>

          <div class="space-y-6 p-5 sm:p-6">
            <div v-for="g in selectedGroups" :key="g.name">
              <h3 class="text-sm font-semibold text-meridian-700 dark:text-meridian-200">{{ g.name }}</h3>

              <ul class="mt-2 divide-y divide-meridian-100 rounded-xl border border-meridian-100 dark:divide-white/10 dark:border-white/10">
                <li
                  v-for="p in g.items"
                  :key="p.key"
                  :class="[
                    'flex items-center justify-between gap-4 px-4 py-3.5 transition-colors',
                    changed(p) ? 'bg-meridian-50/70 dark:bg-white/5' : '',
                  ]"
                >
                  <div class="min-w-0">
                    <div class="flex flex-wrap items-center gap-2">
                      <p class="text-sm font-medium text-meridian-900 dark:text-meridian-50">{{ p.label }}</p>
                      <span
                        v-if="p.sensitive"
                        class="inline-flex items-center gap-1 rounded-md bg-pulse-400/10 px-1.5 py-0.5 text-xs font-medium text-pulse-600"
                      >
                        <LockClosedIcon class="h-3 w-3" />Sensitive
                      </span>
                      <span v-if="changed(p)" class="text-xs font-medium text-meridian-500">Changed</span>
                    </div>
                    <p class="mt-0.5 text-sm text-meridian-500 dark:text-meridian-300">{{ p.description }}</p>
                  </div>

                  <button
                    type="button"
                    role="switch"
                    :aria-checked="p.enabled"
                    :aria-label="`${p.label} for ${selected.name}`"
                    @click="p.enabled = !p.enabled"
                    :class="[
                      'relative inline-flex h-6 w-11 shrink-0 items-center rounded-full transition focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500 focus-visible:ring-offset-2 dark:focus-visible:ring-offset-meridian-900',
                      p.enabled ? 'bg-meridian-600' : 'bg-meridian-200 dark:bg-white/15',
                    ]"
                  >
                    <span
                      :class="[
                        'inline-block h-5 w-5 transform rounded-full bg-white shadow transition',
                        p.enabled ? 'translate-x-5' : 'translate-x-0.5',
                      ]"
                    />
                  </button>
                </li>
              </ul>
            </div>
          </div>
        </section>
      </div>

      <!-- Unsaved changes bar -->
      <div
        v-if="changeCount"
        role="status"
        class="sticky bottom-4 z-30 mt-6 flex flex-wrap items-center justify-between gap-3 rounded-2xl border border-meridian-200 bg-white px-5 py-3 shadow-lg dark:border-white/15 dark:bg-meridian-900"
      >
        <p class="text-sm text-meridian-700 dark:text-meridian-100">
          <span class="font-semibold tabular-nums">{{ changeCount }}</span>
          unsaved {{ plural(changeCount, 'change', 'changes') }} in
          {{ changedRoles.length }} {{ plural(changedRoles.length, 'role', 'roles') }}
        </p>
        <div class="flex gap-2">
          <BaseButton variant="outline" @click="discard">Discard</BaseButton>
          <BaseButton @click="reviewOpen = true">Review &amp; save</BaseButton>
        </div>
      </div>
    </template>

    <!-- Review changes -->
    <BaseModal v-model="reviewOpen" title="Review changes" size="sm">
      <div>
        <p class="text-sm text-meridian-600 dark:text-meridian-300">
          These changes apply immediately to
          <span class="font-semibold text-meridian-900 dark:text-white">{{ affectedMembers.toLocaleString() }} members</span>.
        </p>

        <div
          v-if="grantsSensitive"
          class="mt-4 flex gap-2.5 rounded-xl bg-pulse-400/10 px-3.5 py-3 text-sm text-pulse-700 dark:text-white/90"
        >
          <ExclamationTriangleIcon class="mt-0.5 h-5 w-5 shrink-0" />
          <p>You are granting access to sensitive patient data. Make sure the role really needs it.</p>
        </div>

        <div class="mt-4 max-h-72 space-y-4 overflow-y-auto">
          <div v-for="r in changedRoles" :key="r.id">
            <p class="text-sm font-semibold text-meridian-900 dark:text-white">{{ r.name }}</p>
            <ul class="mt-1 divide-y divide-meridian-100 dark:divide-white/10">
              <li v-for="p in roleChanges(r)" :key="p.key" class="flex items-center justify-between gap-3 py-2 text-sm">
                <span class="text-meridian-700 dark:text-meridian-100">{{ p.label }}</span>
                <BaseBadge :tone="p.enabled ? 'success' : 'neutral'">{{ p.enabled ? 'Enabled' : 'Disabled' }}</BaseBadge>
              </li>
            </ul>
          </div>
        </div>

        <div class="mt-6 flex justify-end gap-3">
          <BaseButton variant="outline" @click="reviewOpen = false">Keep editing</BaseButton>
          <BaseButton :disabled="saving" @click="save">{{ saving ? 'Applying…' : 'Apply changes' }}</BaseButton>
        </div>
      </div>
    </BaseModal>
  </DashboardLayout>
</template>