<script setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'
import DashboardLayout from '@/layouts/DashboardLayout.vue'
import BaseBadge from '@/components/ui/BaseBadge.vue'
import BaseAvatar from '@/components/ui/BaseAvatar.vue'
import usersData from '@/data/users.json'
import {
  ArrowLeftIcon,
  EnvelopeIcon,
  PhoneIcon,
  UserCircleIcon,
} from '@heroicons/vue/24/outline'

const route = useRoute()

// Users.vue te jei key (id ?? email) diye link banano hoyeche, ekhane o sei key diye khuje nibo
const user = computed(() => {
  const key = decodeURIComponent(String(route.params.id))
  const found = usersData.find((u) => String(u.id ?? u.email) === key && u.role !== 'super-admin')
  if (!found) return null
  const { password, ...safe } = found
  return safe
})

const statusTone = { active: 'success', suspended: 'danger', pending: 'warning' }
const formatRole = (r) => String(r).replace('-', ' ')
const whatsappLink = (phone) => `https://wa.me/${String(phone).replace(/\D/g, '')}`

const prettyKey = (k) => k.replace(/([A-Z])/g, ' $1').replace(/[_-]/g, ' ')

// Header e ja already dekhano hocche, seigulo niche repeat korbo na
const HIDDEN_KEYS = ['avatar', 'name', 'id', 'email', 'phone', 'role', 'status']

const extraEntries = computed(() =>
  user.value
    ? Object.entries(user.value)
        .filter(([k]) => !HIDDEN_KEYS.includes(k))
        .map(([k, v]) => [
          prettyKey(k),
          v === null || v === undefined || v === ''
            ? '—'
            : typeof v === 'object'
              ? JSON.stringify(v)
              : String(v),
        ])
    : []
)
</script>

<template>
  <DashboardLayout>
    <!-- Back -->
    <RouterLink
      :to="{ name: 'super-admin-users' }"
      class="mb-5 inline-flex items-center gap-2 rounded-lg px-2 py-1.5 text-sm font-medium text-meridian-500 transition hover:text-meridian-800 focus:outline-none focus-visible:ring-2 focus-visible:ring-pulse-400/60 dark:hover:text-white"
    >
      <ArrowLeftIcon class="h-4 w-4" />
      Back to users
    </RouterLink>

    <!-- Not found -->
    <div
      v-if="!user"
      class="flex flex-col items-center rounded-2xl border border-dashed border-meridian-200 px-6 py-16 text-center dark:border-white/10"
    >
      <UserCircleIcon class="h-12 w-12 text-meridian-400" />
      <h1 class="mt-3 font-display text-xl font-semibold text-meridian-800 dark:text-meridian-100">User not found</h1>
      <p class="mt-1 max-w-xs text-sm text-meridian-500">This user may have been removed. Go back to the list and pick another one.</p>
      <RouterLink
       :to="{ name: 'super-admin-users' }"
        class="mt-5 text-sm font-medium text-pulse-600 hover:text-pulse-700"
      >
        Back to users
      </RouterLink>
    </div>

    <template v-else>
      <!-- Profile header -->
      <section
        class="overflow-hidden rounded-2xl border border-meridian-200 bg-white/70 dark:border-white/10 dark:bg-white/5"
        :class="{ 'opacity-90': user.status === 'suspended' }"
      >
        <!-- Banner -->
        <div class="h-28 bg-meridian-600 sm:h-36" />

        <div class="px-5 pb-6 sm:px-8">
         <div class="flex flex-col gap-5 sm:flex-row sm:items-start sm:justify-between">
          <div class="flex flex-col items-center gap-4 text-center sm:flex-row sm:items-start sm:text-left">
              <div class="-mt-12 rounded-full bg-white p-1.5 shadow-sm dark:bg-meridian-900 sm:-mt-14">
                <BaseAvatar :src="user.avatar" :name="user.name" size="lg" />
              </div>
              <div class="sm:pt-4">
                <h1 class="font-display text-2xl font-semibold text-meridian-900 dark:text-white">{{ user.name }}</h1>
                <div class="mt-2 flex flex-wrap items-center justify-center gap-2 sm:justify-start">
                  <span class="rounded-md bg-meridian-100/70 px-2.5 py-0.5 text-sm font-medium capitalize text-meridian-700 dark:bg-white/10 dark:text-meridian-100">
                    {{ formatRole(user.role) }}
                  </span>
                  <BaseBadge :tone="statusTone[user.status]">{{ user.status }}</BaseBadge>
                  <span v-if="user.id" class="text-xs tabular-nums text-meridian-400">ID: {{ user.id }}</span>
                </div>
              </div>
            </div>

            <!-- Quick actions -->
            <div v-if="user.phone" class="flex items-center justify-center gap-2 sm:pt-4">
              <a
                :href="whatsappLink(user.phone)"
                target="_blank"
                rel="noopener noreferrer"
                class="inline-flex items-center gap-2 rounded-lg border border-meridian-200 px-3.5 py-2 text-sm font-medium text-meridian-600 transition hover:border-meridian-400 hover:text-meridian-800 focus:outline-none focus-visible:ring-2 focus-visible:ring-pulse-400/60 dark:border-white/10 dark:text-meridian-100 dark:hover:border-white/25"
              >
                <svg class="h-4 w-4" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
                  <path d="M17.472 14.382c-.297-.149-1.758-.867-2.03-.967-.273-.099-.471-.148-.67.15-.197.297-.767.966-.94 1.164-.173.199-.347.223-.644.075-.297-.15-1.255-.463-2.39-1.475-.883-.788-1.48-1.761-1.653-2.059-.173-.297-.018-.458.13-.606.134-.133.298-.347.446-.52.149-.174.198-.298.298-.497.099-.198.05-.371-.025-.52-.075-.149-.669-1.612-.916-2.207-.242-.579-.487-.5-.669-.51-.173-.008-.371-.01-.57-.01-.198 0-.52.074-.792.372-.272.297-1.04 1.016-1.04 2.479 0 1.462 1.065 2.875 1.213 3.074.149.198 2.096 3.2 5.077 4.487.709.306 1.262.489 1.694.625.712.227 1.36.195 1.871.118.571-.085 1.758-.719 2.006-1.413.248-.694.248-1.289.173-1.413-.074-.124-.272-.198-.57-.347m-5.421 7.403h-.004a9.87 9.87 0 01-5.031-1.378l-.361-.214-3.741.982.998-3.648-.235-.374a9.86 9.86 0 01-1.51-5.26c.001-5.45 4.436-9.884 9.888-9.884 2.64 0 5.122 1.03 6.988 2.898a9.825 9.825 0 012.893 6.994c-.003 5.45-4.437 9.884-9.885 9.884m8.413-18.297A11.815 11.815 0 0012.05 0C5.495 0 .16 5.335.157 11.892c0 2.096.547 4.142 1.588 5.945L.057 24l6.305-1.654a11.882 11.882 0 005.683 1.448h.005c6.554 0 11.89-5.335 11.893-11.893a11.821 11.821 0 00-3.48-8.413z" />
                </svg>
                WhatsApp
              </a>
              <a
                :href="`tel:${user.phone}`"
                class="inline-flex items-center gap-2 rounded-lg bg-pulse-600 px-3.5 py-2 text-sm font-medium text-white transition hover:bg-pulse-700 focus:outline-none focus-visible:ring-2 focus-visible:ring-pulse-400/60"
              >
                <PhoneIcon class="h-4 w-4" />
                Call
              </a>
            </div>
          </div>
        </div>
      </section>

      <div class="mt-6 grid gap-6 lg:grid-cols-3">
        <!-- Contact -->
        <section class="h-fit rounded-2xl border border-meridian-200 bg-white/70 p-5 dark:border-white/10 dark:bg-white/5">
          <h2 class="font-display text-base font-semibold text-meridian-900 dark:text-white">Contact</h2>

          <ul class="mt-4 space-y-4 text-sm">
            <li class="flex items-start gap-3">
              <span class="rounded-lg bg-pulse-400/10 p-2 text-pulse-600">
                <EnvelopeIcon class="h-4 w-4" />
              </span>
              <div class="min-w-0">
                <p class="text-meridian-500">Email</p>
                <a :href="`mailto:${user.email}`" class="break-all font-medium text-meridian-800 hover:text-pulse-600 dark:text-meridian-100">
                  {{ user.email }}
                </a>
              </div>
            </li>
            <li class="flex items-start gap-3">
              <span class="rounded-lg bg-pulse-400/10 p-2 text-pulse-600">
                <PhoneIcon class="h-4 w-4" />
              </span>
              <div class="min-w-0">
                <p class="text-meridian-500">Phone</p>
                <p class="font-medium text-meridian-800 dark:text-meridian-100">{{ user.phone || '—' }}</p>
              </div>
            </li>
          </ul>
        </section>

        <!-- Account details -->
        <section class="rounded-2xl border border-meridian-200 bg-white/70 p-5 dark:border-white/10 dark:bg-white/5 lg:col-span-2">
          <h2 class="font-display text-base font-semibold text-meridian-900 dark:text-white">Account details</h2>

          <dl v-if="extraEntries.length" class="mt-4 grid gap-3 sm:grid-cols-2">
            <div
              v-for="[label, value] in extraEntries"
              :key="label"
              class="rounded-xl border border-meridian-200 bg-white/60 px-4 py-3 dark:border-white/10 dark:bg-white/5"
            >
              <dt class="text-xs capitalize text-meridian-500">{{ label }}</dt>
              <dd class="mt-1 break-words text-sm font-medium text-meridian-800 dark:text-meridian-100">{{ value }}</dd>
            </div>
          </dl>
          <p v-else class="mt-4 text-sm text-meridian-500">No additional details for this user.</p>
        </section>
      </div>
    </template>
  </DashboardLayout>
</template>