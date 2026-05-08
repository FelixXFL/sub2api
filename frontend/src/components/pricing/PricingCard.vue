<template>
  <div
    class="relative w-full rounded-2xl border bg-white p-6 transition-all duration-300 dark:bg-dark-800"
    :class="[
      selected
        ? 'border-blue-500 shadow-lg ring-2 ring-blue-500/20'
        : 'border-gray-200 shadow-md hover:border-gray-300 hover:shadow-lg dark:border-dark-600',
    ]"
  >
    <div v-if="selected" class="absolute -top-3 left-4 rounded-full bg-blue-500 px-3 py-1 text-xs font-medium text-white">
      已选择
    </div>

    <div class="mb-4 flex items-start justify-between">
      <div>
        <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
          {{ card.name }}
        </h3>
        <span
          v-if="card.badge"
          class="mt-2 inline-block rounded-full border border-orange-200 bg-orange-50 px-3 py-0.5 text-xs font-medium text-orange-600 dark:border-orange-800 dark:bg-orange-900/30 dark:text-orange-400"
        >
          {{ card.badge }}
        </span>
      </div>
    </div>

    <div class="mb-4 rounded-xl bg-gradient-to-r from-blue-50 to-indigo-50 p-4 dark:from-blue-900/20 dark:to-indigo-900/20">
      <div class="mb-1 text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-dark-400">
        倍率
      </div>
      <div class="text-3xl font-bold text-blue-600 dark:text-blue-400">
        {{ card.rate }}
      </div>
    </div>

    <div v-if="card.models && card.models.length > 0" class="mb-4">
      <div class="mb-2 text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-dark-400">
        支持模型
      </div>
      <div class="flex flex-wrap gap-1.5">
        <span
          v-for="model in card.models"
          :key="model"
          class="rounded-md border border-gray-200 bg-gray-50 px-2 py-1 text-xs text-gray-600 dark:border-dark-600 dark:bg-dark-700 dark:text-gray-300"
        >
          {{ model }}
        </span>
      </div>
    </div>

    <p v-if="card.memo" class="text-sm text-gray-500 dark:text-dark-400">
      {{ card.memo }}
    </p>
  </div>
</template>

<script setup lang="ts">
interface PricingCardData {
  name: string
  badge?: string
  models?: string[]
  rate: string
  memo?: string
}

defineProps<{
  card: PricingCardData
  selected?: boolean
}>()
</script>
