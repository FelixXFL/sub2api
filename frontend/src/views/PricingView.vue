<template>
  <div class="min-h-screen bg-gray-50 dark:bg-dark-900">
    <NavBar />

    <main class="mx-auto max-w-6xl px-4 py-12 sm:px-6 lg:px-8">
      <div class="mb-12 text-center">
        <h1 class="mb-4 text-4xl font-bold text-gray-900 dark:text-white">
          选择您的套餐
        </h1>
        <p class="text-lg text-gray-600 dark:text-dark-400">
          透明定价，灵活选择
        </p>
      </div>

      <div class="grid gap-8 lg:grid-cols-2">
        <!-- Left: Pricing Cards -->
        <div class="space-y-6">
          <PricingCard
            v-for="plan in plans"
            :key="plan.name"
            :card="plan"
            :selected="selectedPlan === plan.name"
            class="cursor-pointer"
            @click="selectedPlan = plan.name"
          />
        </div>

        <!-- Right: Calculator -->
        <div class="space-y-6">
          <PricingCalculator :rate="selectedPlanRate" />

          <div class="rounded-2xl border border-gray-200 bg-white p-6 dark:border-dark-600 dark:bg-dark-800">
            <h3 class="mb-4 text-lg font-semibold text-gray-900 dark:text-white">
              已选套餐
            </h3>
            <div class="space-y-3">
              <div class="flex justify-between text-sm">
                <span class="text-gray-600 dark:text-dark-400">套餐名称</span>
                <span class="font-medium text-gray-900 dark:text-white">{{ selectedPlanName }}</span>
              </div>
              <div class="flex justify-between text-sm">
                <span class="text-gray-600 dark:text-dark-400">倍率</span>
                <span class="font-medium text-gray-900 dark:text-white">{{ selectedPlanRate }}</span>
              </div>
            </div>
            <button
              class="btn btn-primary mt-6 w-full"
              @click="handlePurchase"
            >
              立即购买
            </button>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import NavBar from '@/components/layout/NavBar.vue'
import PricingCalculator from '@/components/pricing/PricingCalculator.vue'
import PricingCard from '@/components/pricing/PricingCard.vue'

interface PricingPlan {
  name: string
  badge?: string
  models?: string[]
  rate: string
  memo?: string
}

const plans: PricingPlan[] = [
  {
    name: '基础版',
    badge: '推荐',
    models: ['GPT-4', 'Claude-3'],
    rate: '1:1.5',
    memo: '适合个人用户'
  },
  {
    name: '专业版',
    models: ['GPT-4', 'GPT-4o', 'Claude-3', 'Claude-3.5'],
    rate: '1:1.8',
    memo: '适合团队使用'
  },
  {
    name: '企业版',
    badge: '最优惠',
    models: ['全模型'],
    rate: '1:2.0',
    memo: '适合企业用户'
  }
]

const selectedPlan = ref(plans[0].name)

const selectedPlanData = computed(() => plans.find(p => p.name === selectedPlan.value))
const selectedPlanName = computed(() => selectedPlanData.value?.name || '')
const selectedPlanRate = computed(() => selectedPlanData.value?.rate || null)

function handlePurchase() {
  // TODO: Redirect to payment flow
  console.log('Purchase:', selectedPlan.value)
}
</script>
