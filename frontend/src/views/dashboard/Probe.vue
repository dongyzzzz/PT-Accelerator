<template>
  <div class="dashboard-redesign">
    <div class="page-header">
      <h2 class="page-title">Cloudflare 探活监控</h2>
      <Teleport to="#mobile-header-actions" :disabled="!isMobile">
        <div class="page-header-actions" v-if="isMobile || true">
          <button class="action-btn action-btn-primary action-btn-compact" @click="refreshProbeData" :disabled="loadingStatus || loadingHistory">
            <span>
              <i class="bx bx-refresh" :class="{ spin: loadingStatus || loadingHistory }"></i>
              <span v-if="!isMobile">刷新</span>
            </span>
          </button>
          <button class="action-btn action-btn-success action-btn-compact" @click="runProbeNow" :disabled="runningProbe">
            <span>
              <i class="bx bx-play-circle"></i>
              <span v-if="!isMobile">立即探活</span>
            </span>
          </button>
        </div>
      </Teleport>
    </div>

    <section class="insight-grid">
      <article class="workspace-card">
        <header class="workspace-card-header">
          <div>
            <h3>探活配置</h3>
            <p>配置探活频率、失败阈值与冷却策略。</p>
          </div>
        </header>

        <form @submit.prevent="saveProbeConfig" class="config-form">
          <div class="toggle-row">
            <label class="form-check me-3 mb-0">
              <input class="form-check-input" type="checkbox" v-model="probeConfig.probe_enable">
            </label>
            <div>
              <strong>启用探活任务</strong>
              <p>关闭后不会执行周期探活，也不会触发阈值重优选。</p>
            </div>
          </div>

          <div class="config-field">
            <label class="form-label">探活 CRON</label>
            <CronInput v-model="probeConfig.probe_cron" />
          </div>

          <div class="config-grid">
            <div class="config-field">
              <label class="form-label">失败阈值</label>
              <input class="form-control" type="number" min="1" v-model.number="probeConfig.probe_fail_threshold" />
            </div>
            <div class="config-field">
              <label class="form-label">冷却时间（分钟）</label>
              <input class="form-control" type="number" min="0" v-model.number="probeConfig.cooldown_minutes" />
            </div>
            <div class="config-field">
              <label class="form-label">探活超时（秒）</label>
              <input class="form-control" type="number" min="1" v-model.number="probeConfig.probe_timeout" />
            </div>
          </div>

          <button type="submit" class="save-config-btn" :disabled="savingConfig">
            <span>
              <span v-if="savingConfig" class="spinner-border spinner-border-sm me-2"></span>
              <i v-else class="bx bx-save me-2"></i>
              保存探活配置
            </span>
          </button>
        </form>
      </article>

      <article class="workspace-card status-card">
        <header class="workspace-card-header compact">
          <div>
            <h3>运行状态</h3>
            <p>当前探活状态、冷却时间和最近一次结果。</p>
          </div>
          <span class="workspace-pill" :class="statusPillClass">
            <span class="workspace-pill-dot"></span>
            {{ statusText }}
          </span>
        </header>

        <div class="status-grid">
          <div class="status-item">
            <span>当前IP</span>
            <strong class="mono-text">{{ runtimeStatus?.current_ip || '--' }}</strong>
          </div>
          <div class="status-item">
            <span>最近可用IP</span>
            <strong class="mono-text">{{ runtimeStatus?.last_good_cloudflare_ip || '--' }}</strong>
          </div>
          <div class="status-item">
            <span>连续失败次数</span>
            <strong>{{ runtimeStatus?.consecutive_failures ?? 0 }}</strong>
          </div>
          <div class="status-item">
            <span>冷却剩余</span>
            <strong>{{ cooldownText }}</strong>
          </div>
        </div>

        <div class="latest-result">
          <h4>最近一次探活</h4>
          <p class="latest-time mono-text">{{ latestEvent?.time || '--' }}</p>
          <p class="latest-message">{{ latestEvent?.message || '暂无记录' }}</p>
        </div>
      </article>
    </section>

    <section class="logs-layout">
      <article class="workspace-card logs-card">
        <header class="workspace-card-header logs-card-header">
          <div class="logs-card-heading">
            <div class="logs-card-title-row">
              <h3>探活历史 <span class="log-count">({{ history.length }}条)</span></h3>
            </div>
            <p>记录每次探活结果、阈值触发、冷却与回滚行为。</p>
          </div>
          <button class="action-btn action-btn-danger action-btn-compact" @click="clearHistory" :disabled="clearingHistory || history.length === 0">
            <span>
              <i class="bx bx-trash"></i>
              清空历史
            </span>
          </button>
        </header>

        <div class="logs-split">
          <div class="probe-log-viewer">
            <div v-if="history.length === 0" class="workspace-empty logs-empty">
              <i class="bx bx-history"></i>
              <strong>暂无探活记录</strong>
              <span>等待定时探活执行或手动触发后，这里会显示结果。</span>
            </div>
            <div v-else class="probe-log-list">
              <div class="probe-log-row" v-for="item in history" :key="`${item.timestamp}-${item.status}-${item.message}`">
                <div class="probe-log-meta">
                  <span class="probe-badge" :class="`probe-badge-${item.status}`">{{ formatStatus(item.status) }}</span>
                  <span class="mono-text">{{ item.time || formatTime(item.timestamp) }}</span>
                </div>
                <div class="probe-log-message">{{ item.message }}</div>
              </div>
            </div>
          </div>

          <div class="uptime-panel">
            <div class="uptime-panel-header">
              <h4>稳定IP维持记录 <span class="log-count">({{ uptimeHistory.length }}条)</span></h4>
              <p>展示探活判定为正常时，同一IP的连续维持区间（最近20条）。</p>
            </div>
            <div v-if="uptimeHistory.length === 0" class="workspace-empty logs-empty uptime-empty">
              <i class="bx bx-timer"></i>
              <strong>暂无稳定维持记录</strong>
            </div>
            <div v-else class="uptime-table-wrap">
              <table class="uptime-table">
                <thead>
                  <tr>
                    <th>IP</th>
                    <th>开始时间</th>
                    <th>结束时间</th>
                    <th>持续时长</th>
                    <th>结束原因</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="item in uptimeHistory" :key="`${item.ip}-${item.start_timestamp}-${item.end_timestamp}`">
                    <td class="mono-text">{{ item.ip || '--' }}</td>
                    <td class="mono-text">{{ item.start_time || formatTime(item.start_timestamp) }}</td>
                    <td class="mono-text">
                      {{ item.ongoing ? '进行中' : (item.end_time || formatTime(item.end_timestamp)) }}
                    </td>
                    <td>{{ formatDuration(item.duration_seconds, item.ongoing) }}</td>
                    <td>{{ formatEndReason(item.end_reason, item.ongoing) }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </article>
    </section>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, onUnmounted, reactive, ref } from 'vue';
import axios from '../../api/axios';
import CronInput from '../../components/CronInput.vue';
import { useToast } from 'vue-toastification';
import { useConfirm } from '../../composables/useConfirm';
import { useMobile } from '../../composables/useMobile';
import { useTrackerStore } from '../../stores/trackers';

interface ProbeRuntimeStatus {
  current_ip?: string;
  last_good_cloudflare_ip?: string;
  consecutive_failures?: number;
  cooldown_remaining_seconds?: number;
  last_probe_status?: {
    status?: string;
    message?: string;
    timestamp?: number;
  };
}

interface ProbeHistoryItem {
  timestamp: number;
  time?: string;
  status: string;
  message: string;
}

interface ProbeIpUptimeItem {
  ip: string;
  start_timestamp: number;
  start_time?: string;
  end_timestamp: number;
  end_time?: string;
  duration_seconds: number;
  ongoing: boolean;
  end_reason?: string;
}

const toast = useToast();
const { confirm } = useConfirm();
const { isMobile } = useMobile();
const trackerStore = useTrackerStore();

const loadingStatus = ref(false);
const loadingHistory = ref(false);
const runningProbe = ref(false);
const clearingHistory = ref(false);
const savingConfig = ref(false);
const runtimeStatus = ref<ProbeRuntimeStatus | null>(null);
const history = ref<ProbeHistoryItem[]>([]);
const uptimeHistory = ref<ProbeIpUptimeItem[]>([]);
let pollTimer: number | null = null;

const probeConfig = reactive({
  probe_enable: true,
  probe_cron: '*/5 * * * *',
  probe_fail_threshold: 3,
  cooldown_minutes: 15,
  probe_timeout: 2,
});

const latestEvent = computed(() => history.value[0] || null);

const statusText = computed(() => {
  const status = runtimeStatus.value?.last_probe_status?.status || 'unknown';
  return formatStatus(status);
});

const statusPillClass = computed(() => {
  const status = runtimeStatus.value?.last_probe_status?.status || 'unknown';
  if (status === 'healthy' || status === 'refreshed') return 'success';
  if (status === 'degraded' || status === 'cooldown') return 'warning';
  if (status === 'rolled_back') return 'info';
  if (status === 'disabled') return 'secondary';
  if (status === 'unknown' || status === 'skipped') return 'secondary';
  return 'danger';
});

const cooldownText = computed(() => {
  const remain = runtimeStatus.value?.cooldown_remaining_seconds || 0;
  if (remain <= 0) return '0s';
  if (remain < 60) return `${remain}s`;
  const minute = Math.floor(remain / 60);
  const second = remain % 60;
  return `${minute}m ${second}s`;
});

const normalizeInt = (value: unknown, fallback: number, minValue: number) => {
  const parsed = Number(value);
  if (!Number.isFinite(parsed)) return fallback;
  return Math.max(minValue, Math.floor(parsed));
};

const applyProbeConfigFromStore = () => {
  const cf = trackerStore.cloudflare || {};
  probeConfig.probe_enable = Boolean(cf.probe_enable ?? true);
  probeConfig.probe_cron = String(cf.probe_cron || '*/5 * * * *');
  probeConfig.probe_fail_threshold = normalizeInt(cf.probe_fail_threshold, 3, 1);
  probeConfig.cooldown_minutes = normalizeInt(cf.cooldown_minutes, 15, 0);
  probeConfig.probe_timeout = normalizeInt(cf.probe_timeout, 2, 1);
};

const loadCloudflareConfig = async () => {
  await trackerStore.fetchConfig();
  applyProbeConfigFromStore();
};

const loadProbeStatus = async () => {
  loadingStatus.value = true;
  try {
    const response = await axios.get('/cloudflare-probe/status');
    runtimeStatus.value = response.data || {};
  } catch (e) {
    console.error('Failed to fetch probe status', e);
  } finally {
    loadingStatus.value = false;
  }
};

const loadProbeHistory = async (limit: number = 200) => {
  loadingHistory.value = true;
  try {
    const response = await axios.get(`/cloudflare-probe/history?limit=${limit}`);
    history.value = Array.isArray(response.data?.items) ? response.data.items : [];
  } catch (e) {
    console.error('Failed to fetch probe history', e);
    history.value = [];
  } finally {
    loadingHistory.value = false;
  }
};

const loadProbeIpUptime = async (limit: number = 20) => {
  try {
    const response = await axios.get(`/cloudflare-probe/ip-uptime?limit=${limit}`);
    uptimeHistory.value = Array.isArray(response.data?.items) ? response.data.items : [];
  } catch (e) {
    console.error('Failed to fetch probe ip uptime history', e);
    uptimeHistory.value = [];
  }
};

const refreshProbeData = async () => {
  await Promise.all([loadProbeStatus(), loadProbeHistory(), loadProbeIpUptime()]);
};

const saveProbeConfig = async () => {
  savingConfig.value = true;
  try {
    const mergedCloudflareConfig = {
      ...trackerStore.cloudflare,
      probe_enable: Boolean(probeConfig.probe_enable),
      probe_cron: String(probeConfig.probe_cron || '*/5 * * * *').trim(),
      probe_fail_threshold: normalizeInt(probeConfig.probe_fail_threshold, 3, 1),
      cooldown_minutes: normalizeInt(probeConfig.cooldown_minutes, 15, 0),
      probe_timeout: normalizeInt(probeConfig.probe_timeout, 2, 1),
    };
    await trackerStore.saveCloudflareConfig(mergedCloudflareConfig);
    applyProbeConfigFromStore();
    toast.success('探活配置已保存');
    await loadProbeStatus();
  } catch (e) {
    toast.error('保存失败，请检查参数');
  } finally {
    savingConfig.value = false;
  }
};

const runProbeNow = async () => {
  runningProbe.value = true;
  try {
    await axios.post('/cloudflare-probe/run');
    toast.success('探活任务已启动');
    setTimeout(() => {
      refreshProbeData();
    }, 1200);
  } catch (e) {
    toast.error('启动探活失败');
  } finally {
    runningProbe.value = false;
  }
};

const clearHistory = async () => {
  if (!await confirm('确定要清空探活历史吗？', '清空探活历史')) return;
  clearingHistory.value = true;
  try {
    await axios.post('/cloudflare-probe/history/clear');
    history.value = [];
    uptimeHistory.value = [];
    toast.success('探活历史已清空');
    await loadProbeStatus();
  } catch (e) {
    toast.error('清空失败');
  } finally {
    clearingHistory.value = false;
  }
};

const formatTime = (timestamp: number) => {
  if (!timestamp) return '--';
  const date = new Date(timestamp * 1000);
  return date.toLocaleString();
};

const formatDuration = (seconds: number, ongoing: boolean = false) => {
  const safeSeconds = Math.max(0, Number(seconds) || 0);
  const hour = Math.floor(safeSeconds / 3600);
  const minute = Math.floor((safeSeconds % 3600) / 60);
  const second = safeSeconds % 60;

  const base = hour > 0
    ? `${hour}h ${minute}m ${second}s`
    : (minute > 0 ? `${minute}m ${second}s` : `${second}s`);

  return ongoing ? `${base} (持续中)` : base;
};

const formatEndReason = (reason?: string, ongoing: boolean = false) => {
  if (ongoing) return '进行中';
  const normalized = String(reason || '').trim().toLowerCase();
  const map: Record<string, string> = {
    ip_switched: 'IP切换',
    status_degraded: '探活降级',
    status_cooldown: '进入冷却',
    status_rolled_back: '已回滚',
    status_error: '异常',
    status_disabled: '探活禁用',
    status_unknown: '状态未知',
    ongoing: '进行中',
  };
  return map[normalized] || (normalized ? normalized : '--');
};

const formatStatus = (status: string) => {
  const map: Record<string, string> = {
    healthy: '正常',
    degraded: '降级',
    cooldown: '冷却中',
    refreshed: '已重优选',
    rolled_back: '已回滚',
    error: '异常',
    skipped: '已跳过',
    disabled: '已禁用',
    unknown: '未知',
  };
  return map[status] || status;
};

onMounted(async () => {
  await loadCloudflareConfig();
  await refreshProbeData();
  pollTimer = window.setInterval(() => {
    loadProbeStatus();
    loadProbeHistory(60);
    loadProbeIpUptime(20);
  }, 10000);
});

onUnmounted(() => {
  if (pollTimer !== null) {
    window.clearInterval(pollTimer);
    pollTimer = null;
  }
});
</script>

<style scoped>
.dashboard-redesign {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
}

.page-header-actions {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.insight-grid {
  display: grid;
  grid-template-columns: minmax(0, 1.2fr) minmax(320px, 0.8fr);
  gap: 1.5rem;
}

.workspace-card {
  border-radius: 1.4rem;
  background: var(--bg-surface);
  border: 1px solid rgba(161, 172, 184, 0.14);
  box-shadow: var(--shadow-sm);
  padding: 1.5rem;
}

.workspace-card-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 1rem;
  margin-bottom: 1.2rem;
}

.workspace-card-header h3 {
  margin: 0;
  font-size: 1.2rem;
  font-weight: 700;
}

.workspace-card-header p {
  margin: 0.45rem 0 0;
  color: var(--text-muted);
}

.config-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.toggle-row {
  display: flex;
  align-items: center;
  gap: 0.85rem;
}

.toggle-row p {
  margin: 0.3rem 0 0;
  color: var(--text-muted);
  font-size: 0.88rem;
}

.config-grid {
  display: grid;
  gap: 0.9rem;
  grid-template-columns: repeat(3, minmax(0, 1fr));
}

.config-field {
  display: flex;
  flex-direction: column;
  gap: 0.45rem;
}

.status-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0.9rem;
  margin-bottom: 1rem;
}

.status-item {
  padding: 0.85rem;
  border-radius: 0.8rem;
  border: 1px solid rgba(161, 172, 184, 0.22);
  background: rgba(161, 172, 184, 0.06);
}

.status-item span {
  display: block;
  font-size: 0.78rem;
  color: var(--text-muted);
  margin-bottom: 0.3rem;
}

.status-item strong {
  font-size: 0.98rem;
  color: var(--text-heading);
}

.latest-result {
  padding: 0.9rem;
  border-radius: 0.9rem;
  border: 1px solid rgba(161, 172, 184, 0.2);
}

.latest-result h4 {
  margin: 0;
  font-size: 0.94rem;
}

.latest-time {
  margin: 0.35rem 0;
  font-size: 0.8rem;
  color: var(--text-muted);
}

.latest-message {
  margin: 0;
  font-size: 0.9rem;
  color: var(--text-main);
}

.logs-layout {
  display: flex;
}

.logs-card {
  width: 100%;
  min-height: 420px;
}

.logs-card-header {
  align-items: center;
}

.logs-card-heading {
  min-width: 0;
}

.logs-card-title-row {
  display: flex;
  align-items: center;
  gap: 0.65rem;
}

.log-count {
  color: var(--text-muted);
  font-size: 0.9rem;
}

.probe-log-viewer {
  border-radius: 0.95rem;
  border: 1px solid rgba(161, 172, 184, 0.16);
  background: rgba(161, 172, 184, 0.04);
  overflow: hidden;
  height: 100%;
}

.probe-log-list {
  max-height: 540px;
  overflow-y: auto;
}

.logs-split {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(360px, 1fr));
  gap: 1rem;
  align-items: start;
}

.probe-log-row {
  padding: 0.85rem 1rem;
  border-bottom: 1px solid rgba(161, 172, 184, 0.14);
  display: flex;
  flex-direction: column;
  gap: 0.45rem;
}

.probe-log-row:last-child {
  border-bottom: none;
}

.probe-log-meta {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.8rem;
}

.probe-log-message {
  font-size: 0.9rem;
  color: var(--text-main);
  word-break: break-word;
}

.uptime-panel {
  border-radius: 0.95rem;
  border: 1px solid rgba(161, 172, 184, 0.16);
  background: rgba(161, 172, 184, 0.04);
  overflow: hidden;
  height: 100%;
}

.uptime-panel-header {
  padding: 0.95rem 1rem 0.65rem;
  border-bottom: 1px solid rgba(161, 172, 184, 0.14);
}

.uptime-panel-header h4 {
  margin: 0;
  font-size: 0.95rem;
}

.uptime-panel-header p {
  margin: 0.35rem 0 0;
  color: var(--text-muted);
  font-size: 0.82rem;
}

.uptime-table-wrap {
  max-height: 540px;
  overflow: auto;
}

.uptime-table {
  width: 100%;
  min-width: 760px;
  border-collapse: collapse;
}

.uptime-table th,
.uptime-table td {
  padding: 0.7rem 0.95rem;
  border-bottom: 1px solid rgba(161, 172, 184, 0.14);
  font-size: 0.86rem;
  text-align: left;
  white-space: nowrap;
}

.uptime-table th {
  color: var(--text-muted);
  font-size: 0.78rem;
  font-weight: 700;
  letter-spacing: 0.02em;
}

.uptime-table tbody tr:last-child td {
  border-bottom: none;
}

.uptime-empty {
  padding-top: 1.4rem;
  padding-bottom: 1.4rem;
}

.probe-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.22rem 0.55rem;
  border-radius: 999px;
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 0.02em;
}

.probe-badge-healthy,
.probe-badge-refreshed {
  background: rgba(40, 199, 111, 0.18);
  color: #1f8b4c;
}

.probe-badge-degraded,
.probe-badge-cooldown {
  background: rgba(255, 171, 0, 0.16);
  color: #8b6400;
}

.probe-badge-rolled_back {
  background: rgba(var(--primary-rgb), 0.16);
  color: var(--primary-color);
}

.probe-badge-error {
  background: rgba(234, 84, 85, 0.14);
  color: #b33435;
}

.probe-badge-disabled,
.probe-badge-skipped,
.probe-badge-unknown {
  background: rgba(133, 146, 163, 0.16);
  color: #637083;
}

.workspace-pill {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.35rem 0.72rem;
  border-radius: 999px;
  font-size: 0.74rem;
  font-weight: 700;
}

.workspace-pill-dot {
  width: 0.45rem;
  height: 0.45rem;
  border-radius: 50%;
  background: currentColor;
}

.workspace-pill.success {
  background: rgba(40, 199, 111, 0.16);
  color: #1f8b4c;
}

.workspace-pill.warning {
  background: rgba(255, 171, 0, 0.16);
  color: #8b6400;
}

.workspace-pill.info {
  background: rgba(var(--primary-rgb), 0.16);
  color: var(--primary-color);
}

.workspace-pill.secondary {
  background: rgba(133, 146, 163, 0.16);
  color: #637083;
}

.workspace-pill.danger {
  background: rgba(234, 84, 85, 0.14);
  color: #b33435;
}

.workspace-empty.logs-empty {
  padding: 2.4rem 1rem;
  text-align: center;
  color: var(--text-muted);
}

.action-btn {
  border: none;
  border-radius: 0.7rem;
  padding: 0.5rem 0.8rem;
  font-size: 0.86rem;
  font-weight: 600;
}

.action-btn span {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
}

.action-btn-primary {
  background: rgba(var(--primary-rgb), 0.14);
  color: var(--primary-color);
}

.action-btn-success {
  background: rgba(40, 199, 111, 0.16);
  color: #1f8b4c;
}

.action-btn-danger {
  background: rgba(234, 84, 85, 0.14);
  color: #b33435;
}

.spin {
  animation: spin 0.9s linear infinite;
}

@keyframes spin {
  100% {
    transform: rotate(360deg);
  }
}

@media (max-width: 1199.98px) {
  .insight-grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 991.98px) {
  .config-grid {
    grid-template-columns: 1fr;
  }
}
</style>
