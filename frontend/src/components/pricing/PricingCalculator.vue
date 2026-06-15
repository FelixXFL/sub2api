<template>
  <div class="rounded-2xl border border-gray-200 bg-gradient-to-br from-gray-50 to-white p-6 dark:border-dark-600 dark:from-dark-800 dark:to-dark-800">
    <div class="mb-4">
      <label class="mb-2 block text-sm font-medium text-gray-700 dark:text-gray-300">
        输入充值金额
      </label>
      <div class="relative">
        <input
          v-model.number="amount"
          type="number"
          min="1"
          class="block w-full rounded-lg border border-gray-300 bg-white px-4 py-3 text-lg text-gray-900 focus:border-blue-500 focus:outline-none focus:ring-2 focus:ring-blue-500/20 dark:border-dark-600 dark:bg-dark-700 dark:text-white sm:text-xl"
          placeholder="输入金额"
        >
        <span class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-500 dark:text-dark-400">
          元
        </span>
      </div>
    </div>

    <div class="rounded-xl bg-white p-4 dark:bg-dark-700">
      <div v-if="rate" class="text-center">
        <div class="mb-2 text-sm text-gray-500 dark:text-dark-400">
          预计可用额度
        </div>
        <div class="text-3xl font-bold text-emerald-600 dark:text-emerald-400">
          {{ amount }} 元 ≈ {{ calculatedUsd.toFixed(2) }} 美元额度
        </div>
        <div class="mt-2 text-sm text-gray-500 dark:text-dark-400">
          当前倍率：1:{{ rateValue }}
        </div>
      </div>
      <div v-else class="py-4 text-center text-gray-500 dark:text-dark-400">
        请先选择一个套餐
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'

const props = defineProps<{
  rate: string | null
}>()

const amount = ref(100)

const rateValue = computed(() => {
  if (!props.rate) return 0
  const match = props.rate.match(/1:([\d.]+)/)
  return match ? parseFloat(match[1]) : 0
})

const calculatedUsd = computed(() => {
  if (!rateValue.value) return 0
  return amount.value / rateValue.value
})
</script>
