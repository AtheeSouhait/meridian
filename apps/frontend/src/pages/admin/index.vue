<script lang="ts" setup>
definePageMeta({ layout: 'admin' });

const { data: sources, error: sourcesError } = await useFetch('/api/admin/sources');
if (sourcesError.value) {
  console.error(sourcesError.value);

  if (sourcesError.value.statusCode === 401) {
    await navigateTo('/admin/login');
  } else {
    throw createError({ statusCode: 500, statusMessage: 'Failed to fetch sources' });
  }
}

type Source = NonNullable<typeof sources.value>[number];

const sortKey = ref<keyof Source | ''>('');
const sortOrder = ref<'asc' | 'desc'>('asc');
const selectedFrequency = ref<string>('all');
const selectedCategory = ref<string>('all');
const showPaywallOnly = ref(false);
const showErrorsOnly = ref(false);
const errorThreshold = ref(5);
const staleHours = ref(24);
const enableTimeFilter = ref(false);
const articleCountFilter = ref<'more' | 'less'>('more');
const articleCountThreshold = ref(100);
const enableArticleFilter = ref(false);

const FREQUENCIES = ['Hourly', '4 Hours', '6 Hours', 'Daily'] as const;

const totalArticles = computed(() => sources.value?.reduce((sum, source) => sum + source.totalArticles, 0) ?? 0);
const filteredArticles = computed(() => filteredSources.value.reduce((sum, source) => sum + source.totalArticles, 0));
const stats = computed(() => {
  if (!filteredSources.value.length) return null;

  return {
    avgProcessSuccess: Math.round(
      filteredSources.value.reduce((sum, s) => sum + (s.processSuccessRate ?? 0), 0) / filteredSources.value.length
    ),
    avgErrorRate: Math.round(
      filteredSources.value.reduce((sum, s) => sum + (s.errorRate ?? 0), 0) / filteredSources.value.length
    ),
    avgLowQuality: Math.round(
      filteredSources.value.reduce((sum, s) => sum + (s.lowQualityRate ?? 0), 0) / filteredSources.value.length
    ),
    avgArticlesPerDay: Math.round(
      filteredSources.value.reduce((sum, s) => sum + (s.avgPerDay || 0), 0) / filteredSources.value.length
    ),
  };
});
const filteredSources = computed(() => {
  if (!sources.value) return [];

  let filtered = sources.value;

  // frequency filter
  if (selectedFrequency.value !== 'all') {
    filtered = filtered.filter(source => source.frequency === selectedFrequency.value);
  }

  // category filter
  if (selectedCategory.value !== 'all') {
    filtered = filtered.filter(source => source.category === selectedCategory.value);
  }

  // paywall filter
  if (showPaywallOnly.value) {
    filtered = filtered.filter(source => source.paywall);
  }

  // errors filter
  if (showErrorsOnly.value) {
    filtered = filtered.filter(source => (source.errorRate ?? 0) > errorThreshold.value);
  }

  // stale sources filter
  if (enableTimeFilter.value) {
    const cutoffTime = new Date();
    cutoffTime.setHours(cutoffTime.getHours() - staleHours.value);

    filtered = filtered.filter(source => {
      const lastCheck = new Date(source.lastChecked ?? '');
      return lastCheck > cutoffTime;
    });
  }

  // article count filter
  if (enableArticleFilter.value) {
    filtered = filtered.filter(source => {
      if (articleCountFilter.value === 'more') {
        return source.totalArticles > articleCountThreshold.value;
      }
      return source.totalArticles < articleCountThreshold.value;
    });
  }

  return filtered;
});
const sortedSources = computed(() => {
  if (!sortKey.value) return filteredSources.value;

  const key = sortKey.value as keyof Source;

  return [...filteredSources.value].sort((a, b) => {
    if (key === 'lastChecked') {
      const aTime = new Date(a[key] ?? '').getTime();
      const bTime = new Date(b[key] ?? '').getTime();
      return sortOrder.value === 'asc' ? aTime - bTime : bTime - aTime;
    }

    const aVal = a[key];
    const bVal = b[key];
    return sortOrder.value === 'asc' ? Number(aVal) - Number(bVal) : Number(bVal) - Number(aVal);
  });
});

const toggleSort = (key: keyof Source) => {
  if (sortKey.value === key) {
    sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc';
  } else {
    sortKey.value = key;
    sortOrder.value = 'asc';
  }
};
const formatDate = (dateStr: string) => {
  const date = new Date(dateStr);
  const Y = date.getFullYear();
  const M = String(date.getMonth() + 1).padStart(2, '0');
  const D = String(date.getDate()).padStart(2, '0');
  const h = String(date.getHours()).padStart(2, '0');
  const m = String(date.getMinutes()).padStart(2, '0');
  const s = String(date.getSeconds()).padStart(2, '0');
  return `${Y}-${M}-${D} ${h}:${m}:${s}`;
};
</script>

<template>
  <div>
    <div class="flex justify-between items-center">
      <h1 class="text-xl font-medium text-gray-900 mb-6">Source Analytics</h1>
    </div>

    <div class="grid grid-cols-4 gap-4 mb-6">
      <div class="bg-white p-4 rounded border">
        <div class="text-xs text-gray-500 uppercase tracking-wide mb-1">Avg Process Success</div>
        <div class="text-2xl font-medium text-gray-900">{{ stats?.avgProcessSuccess ?? '-' }}%</div>
      </div>
      <div class="bg-white p-4 rounded border">
        <div class="text-xs text-gray-500 uppercase tracking-wide mb-1">Avg Error Rate</div>
        <div
          class="text-2xl font-medium"
          :class="{ 'text-red-600': (stats?.avgErrorRate ?? 0) > 5, 'text-gray-900': (stats?.avgErrorRate ?? 0) <= 5 }"
        >
          {{ stats?.avgErrorRate ?? '-' }}%
        </div>
      </div>
      <div class="bg-white p-4 rounded border">
        <div class="text-xs text-gray-500 uppercase tracking-wide mb-1">Avg Low Quality</div>
        <div
          class="text-2xl font-medium"
          :class="{
            'text-amber-600': (stats?.avgLowQuality ?? 0) > 10,
            'text-gray-900': (stats?.avgLowQuality ?? 0) <= 10,
          }"
        >
          {{ stats?.avgLowQuality ?? '-' }}%
        </div>
      </div>
      <div class="bg-white p-4 rounded border">
        <div class="text-xs text-gray-500 uppercase tracking-wide mb-1">Avg Articles/Day</div>
        <div class="text-2xl font-medium text-gray-900">{{ stats?.avgArticlesPerDay ?? '-' }}</div>
      </div>
    </div>

    <div class="bg-white rounded border p-4 mb-4">
      <div class="flex flex-wrap gap-6 items-center text-sm">
        <div class="flex items-center gap-2">
          <label class="text-gray-600">Frequency:</label>
          <select v-model="selectedFrequency" class="border rounded px-2 py-1.5 text-sm bg-white">
            <option value="all">All</option>
            <option v-for="freq in FREQUENCIES" :key="freq" :value="freq">{{ freq }}</option>
          </select>
        </div>

        <div class="flex items-center gap-2">
          <label class="text-gray-600">Category:</label>
          <select v-model="selectedCategory" class="border rounded px-2 py-1.5 text-sm bg-white">
            <option value="all">All</option>
            <option v-for="cat in [...new Set(sources?.map(s => s.category))]" :key="cat" :value="cat">
              {{ cat }}
            </option>
          </select>
        </div>

        <div class="flex items-center gap-2">
          <label class="inline-flex items-center gap-2">
            <input type="checkbox" v-model="showPaywallOnly" class="rounded border-gray-300" />
            <span class="text-gray-600">Paywall only</span>
          </label>
        </div>

        <div class="flex items-center gap-2">
          <label class="inline-flex items-center gap-2">
            <input type="checkbox" v-model="showErrorsOnly" class="rounded border-gray-300" />
            <span class="text-gray-600">Error rate above:</span>
          </label>
          <input
            v-model="errorThreshold"
            type="number"
            min="0"
            max="100"
            :disabled="!showErrorsOnly"
            class="border rounded px-2 py-1.5 w-16 text-sm disabled:opacity-50 bg-white"
          />
          <span class="text-gray-600">%</span>
        </div>

        <div class="flex items-center gap-2">
          <label class="inline-flex items-center gap-2">
            <input type="checkbox" v-model="enableTimeFilter" class="rounded border-gray-300" />
            <span class="text-gray-600">Sources checked within:</span>
          </label>
          <input
            v-model="staleHours"
            type="number"
            min="1"
            :disabled="!enableTimeFilter"
            class="border rounded px-2 py-1.5 w-16 text-sm disabled:opacity-50 bg-white"
          />
          <span class="text-gray-600">hours</span>
        </div>

        <div class="flex items-center gap-2">
          <label class="inline-flex items-center gap-2">
            <input type="checkbox" v-model="enableArticleFilter" class="rounded border-gray-300" />
            <span class="text-gray-600">Articles:</span>
          </label>
          <select
            v-model="articleCountFilter"
            :disabled="!enableArticleFilter"
            class="border rounded px-2 py-1.5 text-sm disabled:opacity-50 bg-white"
          >
            <option value="more">More than</option>
            <option value="less">Less than</option>
          </select>
          <input
            v-model="articleCountThreshold"
            type="number"
            min="0"
            :disabled="!enableArticleFilter"
            class="border rounded px-2 py-1.5 w-20 text-sm disabled:opacity-50 bg-white"
          />
        </div>
      </div>
    </div>

    <div class="text-sm text-gray-600 mb-4">
      Showing {{ filteredArticles }} / {{ totalArticles }} articles from {{ filteredSources.length }} /
      {{ sources?.length ?? 0 }} sources
    </div>

    <div class="bg-white text-gray-800 rounded border overflow-hidden">
      <table class="min-w-full divide-y divide-gray-200">
        <thead>
          <tr class="bg-gray-50">
            <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Source</th>
            <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Category</th>
            <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Paywall</th>
            <th
              class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
              @click="toggleSort('totalArticles')"
            >
              Total Articles
              <span v-if="sortKey === 'totalArticles'" class="text-gray-400">{{
                sortOrder === 'asc' ? '↑' : '↓'
              }}</span>
            </th>
            <th
              class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
              @click="toggleSort('avgPerDay')"
            >
              Avg/Day
              <span v-if="sortKey === 'avgPerDay'" class="text-gray-400">{{ sortOrder === 'asc' ? '↑' : '↓' }}</span>
            </th>
            <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Frequency</th>
            <th
              class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
              @click="toggleSort('processSuccessRate')"
            >
              Success Rate
              <span v-if="sortKey === 'processSuccessRate'" class="text-gray-400">{{
                sortOrder === 'asc' ? '↑' : '↓'
              }}</span>
            </th>
            <th
              class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
              @click="toggleSort('errorRate')"
            >
              Error Rate
              <span v-if="sortKey === 'errorRate'" class="text-gray-400">{{ sortOrder === 'asc' ? '↑' : '↓' }}</span>
            </th>
            <th
              class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
              @click="toggleSort('lowQualityRate')"
            >
              Low Quality
              <span v-if="sortKey === 'lowQualityRate'" class="text-gray-400">{{
                sortOrder === 'asc' ? '↑' : '↓'
              }}</span>
            </th>
            <th
              class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
              @click="toggleSort('avgProcessingTime')"
            >
              Avg Process Time
              <span v-if="sortKey === 'avgProcessingTime'" class="text-gray-400">{{
                sortOrder === 'asc' ? '↑' : '↓'
              }}</span>
            </th>
            <th
              class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider cursor-pointer hover:bg-gray-100"
              @click="toggleSort('lastChecked')"
            >
              Last Checked
              <span v-if="sortKey === 'lastChecked'" class="text-gray-400">{{ sortOrder === 'asc' ? '↑' : '↓' }}</span>
            </th>
            <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
          </tr>
        </thead>
        <tbody class="divide-y divide-gray-200">
          <tr v-for="source in sortedSources" :key="source.id" class="hover:bg-gray-50">
            <td class="px-4 py-3">
              <NuxtLink :to="source.url" target="_blank" class="text-gray-500 hover:underline">{{
                source.name
              }}</NuxtLink>
            </td>
            <td class="px-4 py-3">{{ source.category }}</td>
            <td class="px-4 py-3">
              <span v-if="source.paywall" class="text-amber-600">Yes</span>
              <span v-else class="text-gray-400">No</span>
            </td>
            <td class="px-4 py-3">{{ source.totalArticles }}</td>
            <td class="px-4 py-3">{{ source.avgPerDay ? source.avgPerDay.toFixed(1) : 'N/A' }}</td>
            <td class="px-4 py-3">{{ source.frequency }}</td>
            <td class="px-4 py-3">{{ source.processSuccessRate?.toFixed(1) ?? 'N/A' }}%</td>
            <td class="px-4 py-3" :class="{ 'text-red-600': (source.errorRate ?? 0) > 5 }">
              {{ source.errorRate?.toFixed(1) ?? 'N/A' }}%
            </td>
            <td class="px-4 py-3" :class="{ 'text-amber-600': (source.lowQualityRate ?? 0) > 10 }">
              {{ source.lowQualityRate?.toFixed(1) ?? 'N/A' }}%
            </td>
            <td class="px-4 py-3">{{ source.avgProcessingTime ? `${source.avgProcessingTime}s` : 'N/A' }}</td>
            <td class="px-4 py-3 text-gray-500">{{ formatDate(source.lastChecked ?? '') }}</td>
            <td class="px-4 py-3">
              <NuxtLink :to="`/admin/feed/${source.id}`"> View Feed </NuxtLink>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>
