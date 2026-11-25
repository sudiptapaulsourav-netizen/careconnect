```vue
<script setup>
import { ref, computed, onMounted } from 'vue'
import DashboardLayout from '@/layouts/DashboardLayout.vue'
import EmptyState from '@/components/ui/EmptyState.vue'
import SkeletonLine from '@/components/skeletons/SkeletonLine.vue'
import { useNotificationStore } from '@/stores/notifications'
import {
  BellAlertIcon,
  ClockIcon,
  CalendarDaysIcon,
  CheckCircleIcon,
  PlusIcon,
  TrashIcon,
  ClipboardDocumentCheckIcon,
  MapPinIcon,
  UserCircleIcon,
  CheckIcon,
  XMarkIcon,
} from '@heroicons/vue/24/outline'

const notif = useNotificationStore()

const filter = ref('all')
const taskInput = ref('')
const tasks = ref([])
const taskError = ref('')

onMounted(() => {
  notif.fetch()
})

/* ---------------- Filters ---------------- */

const filters = [
  { key: 'all', label: 'All activity' },
  { key: 'appointment', label: 'Appointments' },
  { key: 'reminder', label: 'Reminders' },
  { key: 'todo', label: 'To-do list' },
]

/* ---------------- Notifications ---------------- */

// Only doctor appointment and reminder notifications are displayed.
const notifications = computed(() =>
  (notif.items || []).filter((n) =>
    ['appointment', 'reminder'].includes(n.type)
  )
)

const unreadCount = computed(() =>
  notifications.value.filter((n) => !n.read).length
)

const appointmentCount = computed(() =>
  notifications.value.filter((n) => n.type === 'appointment').length
)

const filteredNotifications = computed(() => {
  if (filter.value === 'all') return notifications.value

  return notifications.value.filter(
    (n) => n.type === filter.value
  )
})

function markNotificationRead(notification) {
  if (!notification.read) {
    notif.markRead(notification.id)
  }
}

function markAllNotificationsRead() {
  if (unreadCount.value) {
    notif.markAllRead()
  }
}

/* ---------------- To-do List ---------------- */

// Tasks are currently kept in page state.
// Connect these handlers to your backend if tasks should persist.
function addTask() {
  const title = taskInput.value.trim()

  if (!title) {
    taskError.value = 'Please enter a task first.'
    return
  }

  tasks.value.unshift({
    id: `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
    title,
    completed: false,
    createdAt: new Date().toISOString(),
  })

  taskInput.value = ''
  taskError.value = ''
}

function toggleTask(id) {
  tasks.value = tasks.value.map((task) =>
    task.id === id
      ? { ...task, completed: !task.completed }
      : task
  )
}

function removeTask(id) {
  tasks.value = tasks.value.filter((task) => task.id !== id)
}

const pendingTasks = computed(() =>
  tasks.value.filter((task) => !task.completed)
)

const completedTasks = computed(() =>
  tasks.value.filter((task) => task.completed)
)

const visibleTasks = computed(() => {
  if (filter.value === 'todo') return tasks.value
  return tasks.value
})

/* ---------------- Helpers ---------------- */

function getNotificationIcon(type) {
  return type === 'appointment' ? CalendarDaysIcon : ClockIcon
}

function getAppointmentDate(notification) {
  return (
    notification.appointmentDate ||
    notification.date ||
    notification.scheduledDate ||
    ''
  )
}

function getAppointmentTime(notification) {
  return (
    notification.appointmentTime ||
    notification.appointmentTimeSlot ||
    notification.scheduledTime ||
    ''
  )
}

function getPatientName(notification) {
  return (
    notification.patientName ||
    notification.patient?.name ||
    notification.patient?.fullName ||
    ''
  )
}

function getChamber(notification) {
  return (
    notification.chamberName ||
    notification.chamber?.name ||
    notification.hospitalName ||
    notification.location ||
    ''
  )
}

function getAppointmentStatus(notification) {
  return notification.status || notification.appointmentStatus || ''
}

function getStatusClass(status) {
  const normalized = String(status).toLowerCase()

  if (['confirmed', 'accepted', 'approved'].includes(normalized)) {
    return 'bg-emerald-50 text-emerald-700 dark:bg-emerald-500/10 dark:text-emerald-400'
  }

  if (['cancelled', 'canceled', 'rejected'].includes(normalized)) {
    return 'bg-red-50 text-red-600 dark:bg-red-500/10 dark:text-red-400'
  }

  if (['completed'].includes(normalized)) {
    return 'bg-blue-50 text-blue-700 dark:bg-blue-500/10 dark:text-blue-400'
  }

  return 'bg-amber-50 text-amber-700 dark:bg-amber-500/10 dark:text-amber-400'
}
</script>

<template>
  <DashboardLayout>
    <div class="space-y-6">

      <!-- Page Header -->
      <div class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
        <div>
          <div class="flex items-center gap-2">
            <h1 class="font-display text-2xl font-semibold text-meridian-900 dark:text-white">
              Notifications
            </h1>

            <span
              v-if="unreadCount"
              class="inline-flex min-w-6 items-center justify-center rounded-full bg-pulse-500/10 px-2 py-1 text-xs font-semibold text-pulse-500"
            >
              {{ unreadCount }} new
            </span>
          </div>

          <p class="mt-1.5 text-sm text-meridian-500 dark:text-meridian-400">
            Keep track of your appointments, reminders and daily tasks.
          </p>
        </div>

        <button
          v-if="unreadCount"
          type="button"
          @click="markAllNotificationsRead"
          class="inline-flex items-center justify-center gap-2 self-start rounded-lg border border-meridian-200 dark:border-white/10 px-3.5 py-2 text-sm font-medium text-meridian-600 dark:text-meridian-300 transition-colors hover:bg-meridian-50 dark:hover:bg-white/5 sm:self-auto"
        >
          <CheckCircleIcon class="h-4 w-4" />
          Mark all as read
        </button>
      </div>

      <!-- Overview Cards -->
      <div class="grid grid-cols-1 gap-4 sm:grid-cols-3">

        <button
          type="button"
          @click="filter = 'all'"
          class="card-base p-4 text-left transition-all hover:border-meridian-300 dark:hover:border-white/20"
          :class="filter === 'all' ? 'ring-1 ring-meridian-500/20 border-meridian-300 dark:border-white/20' : ''"
        >
          <div class="flex items-center justify-between">
            <div class="flex h-10 w-10 items-center justify-center rounded-xl bg-meridian-50 dark:bg-white/5">
              <BellAlertIcon class="h-5 w-5 text-meridian-600 dark:text-meridian-300" />
            </div>

            <span class="text-xs text-meridian-400">Updates</span>
          </div>

          <p class="mt-4 text-2xl font-semibold text-meridian-900 dark:text-white">
            {{ notifications.length }}
          </p>

          <p class="mt-1 text-sm text-meridian-500 dark:text-meridian-400">
            Total notifications
          </p>
        </button>

        <button
          type="button"
          @click="filter = 'appointment'"
          class="card-base p-4 text-left transition-all hover:border-meridian-300 dark:hover:border-white/20"
          :class="filter === 'appointment' ? 'ring-1 ring-meridian-500/20 border-meridian-300 dark:border-white/20' : ''"
        >
          <div class="flex items-center justify-between">
            <div class="flex h-10 w-10 items-center justify-center rounded-xl bg-blue-50 dark:bg-blue-500/10">
              <CalendarDaysIcon class="h-5 w-5 text-blue-600 dark:text-blue-400" />
            </div>

            <span class="text-xs text-meridian-400">Schedule</span>
          </div>

          <p class="mt-4 text-2xl font-semibold text-meridian-900 dark:text-white">
            {{ appointmentCount }}
          </p>

          <p class="mt-1 text-sm text-meridian-500 dark:text-meridian-400">
            Appointment updates
          </p>
        </button>

        <button
          type="button"
          @click="filter = 'todo'"
          class="card-base p-4 text-left transition-all hover:border-meridian-300 dark:hover:border-white/20"
          :class="filter === 'todo' ? 'ring-1 ring-meridian-500/20 border-meridian-300 dark:border-white/20' : ''"
        >
          <div class="flex items-center justify-between">
            <div class="flex h-10 w-10 items-center justify-center rounded-xl bg-emerald-50 dark:bg-emerald-500/10">
              <ClipboardDocumentCheckIcon class="h-5 w-5 text-emerald-600 dark:text-emerald-400" />
            </div>

            <span class="text-xs text-meridian-400">Your tasks</span>
          </div>

          <p class="mt-4 text-2xl font-semibold text-meridian-900 dark:text-white">
            {{ pendingTasks.length }}
          </p>

          <p class="mt-1 text-sm text-meridian-500 dark:text-meridian-400">
            Tasks remaining
          </p>
        </button>
      </div>

      <!-- Filter Navigation -->
      <div class="flex flex-wrap items-center gap-2">
        <button
          v-for="f in filters"
          :key="f.key"
          type="button"
          @click="filter = f.key"
          class="rounded-full px-4 py-2 text-xs font-medium transition-colors btn-focus-ring"
          :class="filter === f.key
            ? 'bg-meridian-600 text-white shadow-sm'
            : 'bg-meridian-50 dark:bg-white/5 text-meridian-600 dark:text-meridian-300 hover:bg-meridian-100 dark:hover:bg-white/10'"
        >
          {{ f.label }}
          <span
            v-if="f.key === 'todo' && pendingTasks.length"
            class="ml-1.5"
          >
            {{ pendingTasks.length }}
          </span>
        </button>
      </div>

      <!-- To-do List -->
      <section
        v-if="filter === 'todo' || filter === 'all'"
        class="card-base overflow-hidden"
      >
        <div class="border-b border-meridian-100 dark:border-white/10 p-5 sm:p-6">
          <div class="flex items-start justify-between gap-3">
            <div>
              <h2 class="font-display text-lg font-semibold text-meridian-900 dark:text-white">
                My To-do List
              </h2>

              <p class="mt-1 text-sm text-meridian-500 dark:text-meridian-400">
                Organize your work and keep your day on track.
              </p>
            </div>

            <div class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl bg-meridian-50 dark:bg-white/5">
              <ClipboardDocumentCheckIcon class="h-5 w-5 text-meridian-600 dark:text-meridian-300" />
            </div>
          </div>

          <!-- Add Task -->
          <form @submit.prevent="addTask" class="mt-5">
            <div class="flex flex-col gap-2 sm:flex-row">
              <div class="relative min-w-0 flex-1">
                <input
                  v-model="taskInput"
                  type="text"
                  maxlength="200"
                  placeholder="Add a task, e.g. Review patient reports"
                  aria-label="New task"
                  @input="taskError = ''"
                  class="w-full rounded-xl border border-meridian-200 bg-white px-4 py-3 pr-10 text-sm text-meridian-900 outline-none transition-colors placeholder:text-meridian-400 focus:border-meridian-500 dark:border-white/10 dark:bg-white/5 dark:text-white"
                />

                <XMarkIcon
                  v-if="taskInput"
                  class="absolute right-3 top-1/2 h-4 w-4 -translate-y-1/2 cursor-pointer text-meridian-400"
                  @click="taskInput = ''; taskError = ''"
                />
              </div>

              <button
                type="submit"
                class="inline-flex items-center justify-center gap-2 rounded-xl bg-meridian-600 px-5 py-3 text-sm font-semibold text-white transition-colors hover:bg-meridian-700"
              >
                <PlusIcon class="h-4 w-4" />
                Add task
              </button>
            </div>

            <p v-if="taskError" class="mt-2 text-xs text-red-500">
              {{ taskError }}
            </p>
          </form>
        </div>

        <!-- Task Progress -->
        <div
          v-if="tasks.length"
          class="border-b border-meridian-100 dark:border-white/10 px-5 py-3 sm:px-6"
        >
          <div class="flex items-center justify-between text-xs">
            <span class="text-meridian-500 dark:text-meridian-400">
              {{ completedTasks.length }} of {{ tasks.length }} completed
            </span>

            <span class="font-semibold text-meridian-600 dark:text-meridian-300">
              {{ Math.round((completedTasks.length / tasks.length) * 100) }}%
            </span>
          </div>

          <div class="mt-2 h-1.5 overflow-hidden rounded-full bg-meridian-100 dark:bg-white/10">
            <div
              class="h-full rounded-full bg-emerald-500 transition-all duration-300"
              :style="{ width: `${(completedTasks.length / tasks.length) * 100}%` }"
            ></div>
          </div>
        </div>

        <!-- Task Items -->
        <div v-if="visibleTasks.length" class="divide-y divide-meridian-100 dark:divide-white/5">
          <div
            v-for="task in visibleTasks"
            :key="task.id"
            class="flex items-start gap-3 px-5 py-4 transition-colors hover:bg-meridian-50/50 dark:hover:bg-white/[0.02] sm:px-6"
          >
            <button
              type="button"
              @click="toggleTask(task.id)"
              :aria-label="task.completed ? 'Mark task incomplete' : 'Complete task'"
              class="mt-0.5 flex h-5 w-5 shrink-0 items-center justify-center rounded-md border transition-colors"
              :class="task.completed
                ? 'border-emerald-500 bg-emerald-500 text-white'
                : 'border-meridian-300 dark:border-white/20 hover:border-meridian-500'"
            >
              <CheckIcon v-if="task.completed" class="h-3.5 w-3.5" />
            </button>

            <div class="min-w-0 flex-1">
              <p
                class="break-words text-sm font-medium"
                :class="task.completed
                  ? 'text-meridian-400 line-through'
                  : 'text-meridian-900 dark:text-white'"
              >
                {{ task.title }}
              </p>

              <p class="mt-1 text-xs text-meridian-400">
                {{ task.completed ? 'Completed' : 'Pending task' }}
              </p>
            </div>

            <button
              type="button"
              @click="removeTask(task.id)"
              aria-label="Delete task"
              class="flex h-8 w-8 shrink-0 items-center justify-center rounded-lg text-meridian-400 transition-colors hover:bg-red-50 hover:text-red-500 dark:hover:bg-red-500/10"
            >
              <TrashIcon class="h-4 w-4" />
            </button>
          </div>
        </div>

        <div v-else class="px-5 py-8 text-center sm:px-6">
          <div class="mx-auto flex h-12 w-12 items-center justify-center rounded-2xl bg-meridian-50 dark:bg-white/5">
            <CheckCircleIcon class="h-6 w-6 text-meridian-500" />
          </div>

          <p class="mt-3 text-sm font-semibold text-meridian-900 dark:text-white">
            You're all caught up
          </p>

          <p class="mt-1 text-xs text-meridian-500 dark:text-meridian-400">
            Add a task above to plan your day.
          </p>
        </div>
      </section>

      <!-- Notifications List -->
      <section
        v-if="filter !== 'todo'"
        class="card-base overflow-hidden"
      >
        <div class="flex items-center justify-between border-b border-meridian-100 dark:border-white/10 px-5 py-4 sm:px-6">
          <div>
            <h2 class="font-display text-lg font-semibold text-meridian-900 dark:text-white">
              {{ filter === 'appointment'
                ? 'Appointment Updates'
                : filter === 'reminder'
                  ? 'Reminders'
                  : 'Recent Activity' }}
            </h2>

            <p class="mt-1 text-xs text-meridian-500 dark:text-meridian-400">
              Stay informed about your patient schedule.
            </p>
          </div>

          <span class="text-xs text-meridian-400">
            {{ filteredNotifications.length }} items
          </span>
        </div>

        <!-- Loading -->
        <div v-if="notif.loading" class="space-y-4 p-5 sm:p-6">
          <SkeletonLine v-for="i in 4" :key="i" height="4rem" />
        </div>

        <!-- Notification Items -->
        <div
          v-else-if="filteredNotifications.length"
          class="divide-y divide-meridian-100 dark:divide-white/5"
        >
          <button
            v-for="n in filteredNotifications"
            :key="n.id"
            type="button"
            @click="markNotificationRead(n)"
            class="group flex w-full items-start gap-3 p-5 text-left transition-colors hover:bg-meridian-50/60 dark:hover:bg-white/[0.03] sm:gap-4 sm:px-6"
          >
            <!-- Icon -->
            <span
              class="flex h-11 w-11 shrink-0 items-center justify-center rounded-xl"
              :class="n.type === 'appointment'
                ? 'bg-blue-50 text-blue-600 dark:bg-blue-500/10 dark:text-blue-400'
                : 'bg-meridian-50 text-meridian-600 dark:bg-white/5 dark:text-meridian-300'"
            >
              <component
                :is="getNotificationIcon(n.type)"
                class="h-5 w-5"
              />
            </span>

            <!-- Content -->
            <div class="min-w-0 flex-1">
              <div class="flex flex-wrap items-center gap-2">
                <p class="text-sm font-semibold text-meridian-900 dark:text-white">
                  {{ n.title }}
                </p>

                <span
                  v-if="!n.read"
                  class="h-1.5 w-1.5 shrink-0 rounded-full bg-pulse-500"
                  aria-label="Unread"
                ></span>

                <span
                  v-if="n.type === 'appointment'"
                  class="rounded-full bg-blue-50 px-2 py-0.5 text-[10px] font-semibold uppercase tracking-wide text-blue-700 dark:bg-blue-500/10 dark:text-blue-400"
                >
                  Appointment
                </span>
              </div>

              <p class="mt-1 text-sm leading-5 text-meridian-500 dark:text-meridian-400">
                {{ n.body }}
              </p>

              <!-- Appointment Details -->
              <div
                v-if="n.type === 'appointment' && (getPatientName(n) || getAppointmentDate(n) || getAppointmentTime(n) || getChamber(n) || getAppointmentStatus(n))"
                class="mt-3 rounded-xl border border-meridian-100 bg-white p-3 dark:border-white/10 dark:bg-white/[0.02]"
              >
                <div class="grid grid-cols-1 gap-3 sm:grid-cols-2">

                  <div v-if="getPatientName(n)" class="flex items-start gap-2">
                    <UserCircleIcon class="mt-0.5 h-4 w-4 shrink-0 text-meridian-400" />
                    <div class="min-w-0">
                      <p class="text-[10px] font-semibold uppercase tracking-wider text-meridian-400">
                        Patient
                      </p>
                      <p class="mt-0.5 break-words text-xs font-medium text-meridian-800 dark:text-meridian-200">
                        {{ getPatientName(n) }}
                      </p>
                    </div>
                  </div>

                  <div v-if="getAppointmentDate(n) || getAppointmentTime(n)" class="flex items-start gap-2">
                    <ClockIcon class="mt-0.5 h-4 w-4 shrink-0 text-meridian-400" />
                    <div class="min-w-0">
                      <p class="text-[10px] font-semibold uppercase tracking-wider text-meridian-400">
                        Date & time
                      </p>
                      <p class="mt-0.5 break-words text-xs font-medium text-meridian-800 dark:text-meridian-200">
                        {{ getAppointmentDate(n) }}
                        <span v-if="getAppointmentDate(n) && getAppointmentTime(n)"> · </span>
                        {{ getAppointmentTime(n) }}
                      </p>
                    </div>
                  </div>

                  <div v-if="getChamber(n)" class="flex items-start gap-2 sm:col-span-2">
                    <MapPinIcon class="mt-0.5 h-4 w-4 shrink-0 text-meridian-400" />
                    <div class="min-w-0">
                      <p class="text-[10px] font-semibold uppercase tracking-wider text-meridian-400">
                        Chamber / Hospital
                      </p>
                      <p class="mt-0.5 break-words text-xs font-medium text-meridian-800 dark:text-meridian-200">
                        {{ getChamber(n) }}
                      </p>
                    </div>
                  </div>
                </div>

                <div
                  v-if="getAppointmentStatus(n)"
                  class="mt-3 border-t border-meridian-100 pt-3 dark:border-white/10"
                >
                  <span
                    class="inline-flex rounded-full px-2.5 py-1 text-[10px] font-semibold capitalize"
                    :class="getStatusClass(getAppointmentStatus(n))"
                  >
                    {{ getAppointmentStatus(n) }}
                  </span>
                </div>
              </div>

              <p class="mt-2 text-xs text-meridian-400">
                {{ n.time }}
              </p>
            </div>

            <!-- Read State -->
            <span
              v-if="!n.read"
              class="mt-1 h-2 w-2 shrink-0 rounded-full bg-pulse-500"
            ></span>
          </button>
        </div>

        <!-- Empty State -->
        <div v-else class="p-5 sm:p-6">
          <EmptyState
            :title="filter === 'appointment'
              ? 'No appointment updates'
              : filter === 'reminder'
                ? 'No reminders right now'
                : 'No recent notifications'"
            :message="filter === 'appointment'
              ? 'New appointment updates will appear here.'
              : filter === 'reminder'
                ? 'You will see your reminders here when they arrive.'
                : 'You are all caught up. New activity will appear here.'"
          >
            <template #icon>
              <BellAlertIcon class="h-8 w-8 text-meridian-400" />
            </template>
          </EmptyState>
        </div>
      </section>

    </div>
  </DashboardLayout>
</template>
```