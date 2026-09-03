<template>
  <article v-if="layout !== 'cards'" class="status-board-row" :style="{ animationDelay: (index * 0.045) + 's' }">
    <div class="status-board-service">
      <div class="status-board-name-line">
        <span class="status-board-dot" :class="statusClass"></span>
        <h4>{{ monitor.name }}</h4>
        <span v-if="monitor.paused" class="status-board-state">暂停</span>
        <span v-else-if="monitor.status === 'DOWN'" class="status-board-state is-down">异常</span>
        <span v-else-if="monitor.status === 'RETRYING'" class="status-board-state is-retrying">重试中</span>
      </div>
      <a v-if="monitor.show_url !== 0" :href="monitor.url" target="_blank" rel="noopener" class="status-board-url" :title="monitor.url">{{ displayUrl }}</a>
      <span v-if="monitor.cert_expiry" class="status-board-cert" :class="getExpiryClass(monitor.cert_expiry)">证书有效：{{ formatExpiry(monitor.cert_expiry) }}</span>
    </div>
    <div class="status-board-uptime">
      <UptimeBar v-if="monitor.daily_stats && monitor.daily_stats.length > 0 && !monitor.paused" :monitor="monitor" compact />
      <div v-else class="status-board-no-history">{{ formatDate(monitor.last_check) }}</div>
    </div>
  </article>
  <article v-else class="glass-card monitor-card rounded-xl px-5 py-3.5 cursor-default group"
    :class="[
      monitor.paused ? 'opacity-50 monitor-status-paused' : '',
      monitor.status === 'UP' && !monitor.paused ? 'monitor-status-up' : '',
      monitor.status === 'DOWN' ? 'monitor-status-down' : '',
      monitor.status === 'RETRYING' ? 'monitor-status-retrying' : '',
    ]">
    <div class="flex flex-col gap-3 md:flex-row md:items-center md:justify-between">
      <div class="min-w-0 flex-1">
        <div class="flex items-center gap-3">
          <span class="status-board-dot" :class="statusClass"></span>
          <h3 class="font-bold text-slate-900 dark:text-white text-base truncate">{{ monitor.name }}</h3>
        </div>
        <div class="mt-2 flex flex-wrap items-center gap-x-3 gap-y-1 pl-5">
          <a v-if="monitor.show_url !== 0" :href="monitor.url" target="_blank" rel="noopener" class="text-[11px] sm:text-[13px] font-mono text-slate-500 dark:text-slate-500 hover:text-emerald-600 dark:hover:text-emerald-400 transition-colors truncate max-w-full sm:max-w-[420px]">{{ monitor.url }}</a>
          <span class="text-[10px] sm:text-[11px] font-mono text-slate-400 dark:text-slate-600">{{ formatDate(monitor.last_check) }}</span>
          <span v-if="monitor.cert_expiry" class="flex items-center gap-1 px-1.5 py-0.5 rounded-md text-[10px] sm:text-[11px] font-mono border" :class="getExpiryClass(monitor.cert_expiry)">SSL {{ formatExpiry(monitor.cert_expiry) }} · {{ formatExpiryDate(monitor.cert_expiry) }}</span>
        </div>
      </div>
      <div class="flex shrink-0 flex-wrap items-center gap-2 md:justify-end">
        <span v-if="monitor.paused" class="inline-flex min-w-[64px] justify-center px-2.5 py-1 rounded-lg text-[11px] font-bold border bg-slate-100 dark:bg-slate-800 text-slate-600 dark:text-slate-400 border-slate-200 dark:border-slate-700">暂停</span>
        <span v-else-if="monitor.status === 'UP'" class="inline-flex min-w-[64px] justify-center px-2.5 py-1 rounded-lg text-[11px] font-bold border bg-emerald-50 dark:bg-emerald-500/10 text-emerald-700 dark:text-emerald-300 border-emerald-200 dark:border-emerald-500/20">在线</span>
        <span v-else-if="monitor.status === 'DOWN'" class="inline-flex min-w-[64px] justify-center px-2.5 py-1 rounded-lg text-[11px] font-bold border bg-red-50 dark:bg-red-500/10 text-red-700 dark:text-red-300 border-red-200 dark:border-red-500/25">异常</span>
        <span v-else class="inline-flex min-w-[64px] justify-center px-2.5 py-1 rounded-lg text-[11px] font-bold border bg-yellow-50 dark:bg-yellow-400/10 text-yellow-700 dark:text-yellow-300 border-yellow-200 dark:border-yellow-400/25">重试中</span>
        <div v-if="monitor.latency != null && !monitor.paused" class="latency-badge px-2.5 py-1 rounded-lg text-xs font-mono font-medium border" :class="latencyClass(monitor.latency)">{{ monitor.latency }}ms</div>
        <div v-if="monitor.uptime_24h != null && !monitor.paused" class="px-2.5 py-1 rounded-lg text-xs font-mono font-bold border" :class="uptimeClass">24h {{ monitor.uptime_24h }}%</div>
      </div>
    </div>
    <UptimeBar v-if="monitor.daily_stats && monitor.daily_stats.length > 0 && !monitor.paused" :monitor="monitor" />
  </article>
</template>

<script setup>
import { computed } from 'vue';
import { formatDate, getExpiryClass, formatExpiry, formatExpiryDate, latencyClass } from '../../utils/format';
import UptimeBar from './UptimeBar.vue';

const props = defineProps({
    monitor: { type: Object, required: true },
    index: { type: Number, required: true },
    layout: { type: String, default: 'board' },
});

const statusClass = computed(() => {
    if (props.monitor.paused) return 'is-paused';
    if (props.monitor.status === 'DOWN') return 'is-down';
    if (props.monitor.status === 'RETRYING') return 'is-retrying';
    return 'is-up';
});

const displayUrl = computed(() => {
    try { return new URL(props.monitor.url).host; }
    catch { return props.monitor.url; }
});

const uptimeClass = computed(() => {
    const value = props.monitor.uptime_24h;
    if (value >= 99.9) return 'text-emerald-600 dark:text-emerald-300 bg-emerald-50 dark:bg-emerald-500/10 border-emerald-200 dark:border-emerald-500/20';
    if (value >= 95) return 'text-yellow-600 dark:text-yellow-300 bg-yellow-50 dark:bg-yellow-400/10 border-yellow-200 dark:border-yellow-400/20';
    return 'text-red-600 dark:text-red-300 bg-red-50 dark:bg-red-500/10 border-red-200 dark:border-red-500/20';
});
</script>
