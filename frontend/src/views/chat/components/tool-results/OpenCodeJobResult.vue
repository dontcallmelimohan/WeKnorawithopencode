<template>
  <div class="opencode-job-result">
    <div class="ocw-head">
      <span class="ocw-status" :class="`is-${statusKind}`">
        <span class="ocw-dot" aria-hidden="true"></span>
        {{ statusLabel }}
      </span>
      <span v-if="jobId" class="ocw-jobid" :title="jobId">{{ jobId }}</span>
    </div>

    <div v-if="metaParts.length" class="ocw-meta">
      <span v-for="(part, i) in metaParts" :key="i" class="ocw-meta-item">{{ part }}</span>
    </div>

    <div v-if="timeline.length" class="ocw-timeline">
      <div v-for="(item, index) in timeline" :key="index" class="ocw-step">
        <span class="ocw-step-mark" :class="{ 'is-error': item.isError, 'is-last': index === timeline.length - 1 && isRunning }">
          {{ item.isError ? '✗' : (index === timeline.length - 1 && isRunning ? '●' : '✓') }}
        </span>
        <span v-if="item.step" class="ocw-step-no">{{ item.step }}</span>
        <span v-if="item.tool" class="ocw-step-tool">{{ item.tool }}</span>
        <span class="ocw-step-target">{{ item.target }}</span>
      </div>
    </div>

    <div v-if="summary" class="ocw-summary">
      <div class="ocw-summary-label">{{ $t('agentStream.opencodeJob.summary') }}</div>
      <pre class="ocw-summary-body">{{ summary }}</pre>
    </div>

    <div v-if="rawOutput" class="ocw-raw">
      <button type="button" class="ocw-raw-toggle" @click="showRaw = !showRaw">
        <span class="ocw-caret">{{ showRaw ? '▾' : '▸' }}</span>
        {{ $t('agentStream.opencodeJob.rawOutput') }}
      </button>
      <pre v-if="showRaw" class="ocw-raw-body">{{ rawOutput }}</pre>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue'
import { useI18n } from 'vue-i18n'

const props = defineProps<{
  data?: Record<string, unknown>
  output?: string
}>()

const { t } = useI18n()

const showRaw = ref(false)

interface StepItem {
  step: number | null
  tool: string
  target: string
  isError: boolean
}

// execute_skill_script 把 ocw.py 的 stdout 放在 data.stdout；工具的直接输出在 output。
const rawOutput = computed(() => {
  const stdout = props.data?.stdout
  if (typeof stdout === 'string' && stdout.trim()) return stdout
  return props.output || ''
})

// 只在确实是 opencode 任务时渲染：stdout 里含 job_id 字段。
const parsed = computed<Record<string, any> | null>(() => {
  const text = rawOutput.value.trim()
  if (!text.startsWith('{')) return null
  try {
    const obj = JSON.parse(text)
    if (obj && typeof obj === 'object' && typeof obj.job_id === 'string') return obj
  } catch {
    return null
  }
  return null
})

const isOpencodeJob = computed(() => parsed.value !== null)

const jobId = computed(() => (parsed.value?.job_id as string) || '')

const status = computed(() => String(parsed.value?.status || 'unknown'))

const isRunning = computed(() => status.value === 'running' || status.value === 'submitted')

const statusKind = computed(() => {
  if (status.value === 'succeeded') return 'ok'
  if (isRunning.value) return 'running'
  return 'error'
})

const statusLabel = computed(() => {
  const key = `agentStream.opencodeJob.status.${status.value}`
  const label = t(key)
  return label === key ? status.value : label
})

const progress = computed<Record<string, any>>(() => (parsed.value?.progress as Record<string, any>) || {})

const metaParts = computed(() => {
  const parts: string[] = []
  const seconds = progress.value.elapsed_seconds ?? parsed.value?.duration_seconds
  if (typeof seconds === 'number') parts.push(`${Math.round(seconds)}s`)

  const steps = progress.value.steps ?? parsed.value?.steps
  if (typeof steps === 'number') parts.push(t('agentStream.opencodeJob.steps', { count: steps }))

  const calls = progress.value.tool_calls ?? parsed.value?.tool_calls
  if (typeof calls === 'number') parts.push(t('agentStream.opencodeJob.toolCalls', { count: calls }))

  const ctx = progress.value.ctx_tokens
  if (typeof ctx === 'number') parts.push(`ctx ${formatTokens(ctx)}`)

  const waited = parsed.value?.waited_seconds
  if (typeof waited === 'number' && isRunning.value) {
    parts.push(t('agentStream.opencodeJob.waited', { count: Math.round(waited) }))
  }
  return parts.filter(Boolean)
})

const timeline = computed<StepItem[]>(() => {
  const rich = progress.value.timeline
  if (Array.isArray(rich) && rich.length) {
    return rich.map((entry: any) => ({
      step: typeof entry?.n === 'number' ? entry.n : null,
      tool: typeof entry?.tool === 'string' ? entry.tool : '',
      target: String(entry?.target ?? ''),
      isError: entry?.ok === false || entry?.tool === 'invalid',
    }))
  }

  // 兼容现有 ocw.py：recent_activity 是字符串数组（工具标题 / 文件路径 / 命令）。
  const activity = parsed.value?.recent_activity
  if (!Array.isArray(activity)) return []
  return activity.map((entry: unknown, index: number) => {
    const text = String(entry ?? '')
    const match = /^(invalid|[a-z_]+)\s*[·:]\s*(.+)$/i.exec(text)
    return {
      step: index + 1,
      tool: match ? match[1] : '',
      target: match ? match[2] : text,
      isError: !match && /invalid/i.test(text),
    }
  })
})

const summary = computed(() => {
  const value = parsed.value?.summary
  return typeof value === 'string' ? value.trim() : ''
})

function formatTokens(n: number): string {
  if (n < 1000) return String(n)
  if (n < 1000000) return `${(n / 1000).toFixed(1)}K`
  return `${(n / 1000000).toFixed(1)}M`
}

defineExpose({ isOpencodeJob })
</script>

<style lang="less" scoped>
.opencode-job-result {
  display: flex;
  flex-direction: column;
  gap: 10px;
  min-width: 0;
  font-size: 12px;
}

.ocw-head {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.ocw-status {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-weight: 600;

  .ocw-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: currentColor;
  }

  &.is-ok { color: var(--td-success-color); }
  &.is-running { color: var(--td-brand-color); }
  &.is-error { color: var(--td-error-color); }
}

.ocw-jobid {
  font-family: var(--td-font-family-mono, monospace);
  color: var(--td-text-color-placeholder);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  max-width: 100%;
}

.ocw-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  color: var(--td-text-color-secondary);
}

.ocw-timeline {
  display: flex;
  flex-direction: column;
  gap: 3px;
  border-left: 2px solid var(--td-component-stroke);
  padding-left: 10px;
  max-height: 220px;
  overflow-y: auto;
}

.ocw-step {
  display: flex;
  align-items: baseline;
  gap: 6px;
  min-width: 0;
  line-height: 1.6;
}

.ocw-step-mark {
  color: var(--td-success-color);
  flex: none;

  &.is-error { color: var(--td-error-color); }
  &.is-last { color: var(--td-brand-color); }
}

.ocw-step-no {
  color: var(--td-text-color-placeholder);
  flex: none;
  min-width: 18px;
}

.ocw-step-tool {
  flex: none;
  font-weight: 600;
  color: var(--td-text-color-secondary);
}

.ocw-step-target {
  color: var(--td-text-color-primary);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.ocw-summary-label,
.ocw-raw-toggle {
  color: var(--td-text-color-secondary);
}

.ocw-summary-body,
.ocw-raw-body {
  margin: 4px 0 0;
  padding: 8px;
  border-radius: 4px;
  background: var(--td-bg-color-container-hover);
  white-space: pre-wrap;
  word-break: break-word;
  max-height: 260px;
  overflow: auto;
  font-family: var(--td-font-family-mono, monospace);
  font-size: 11px;
  line-height: 1.6;
}

.ocw-raw-toggle {
  background: none;
  border: none;
  padding: 0;
  cursor: pointer;
  font-size: 12px;

  .ocw-caret { margin-right: 4px; }
}
</style>
