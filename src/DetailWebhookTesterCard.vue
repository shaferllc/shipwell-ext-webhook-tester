<template>
  <ExtensionLayout tab-id="webhook-tester" content-class="detail-webhook-tester">
    <template #toolbar-start>
      <p class="text-sm text-rm-muted m-0">
        Send test payloads to your webhook endpoints and inspect the responses.
      </p>
    </template>
    <template #toolbar-end>
      <div class="flex items-center gap-2">
        <Button severity="secondary" size="small" :disabled="loading" @click="loadWebhooks">
          <svg class="w-3.5 h-3.5 mr-1" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M23 4v6h-6"/><path d="M1 20v-6h6"/><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"/></svg>
          Refresh
        </Button>
        <Button severity="primary" size="small" @click="openComposer(null)">
          New request
        </Button>
      </div>
    </template>

    <!-- Webhooks list + quick test -->
    <Panel class="wht-webhooks-panel">
      <template #header>
        <div class="flex items-center justify-between gap-3 w-full">
          <h3 class="text-sm font-semibold text-rm-text m-0 tracking-tight">Registered webhooks</h3>
          <span v-if="webhooks.length" class="text-xs text-rm-muted">{{ webhooks.length }} webhook{{ webhooks.length === 1 ? '' : 's' }}</span>
        </div>
      </template>

      <div v-if="loading" class="flex items-center justify-center py-10">
        <ProgressSpinner style="width: 32px; height: 32px" strokeWidth="3" />
      </div>

      <div v-else-if="error" class="py-6 px-4 text-center">
        <p class="text-sm text-rm-danger m-0">{{ error }}</p>
        <Button severity="secondary" size="small" class="mt-3" @click="loadWebhooks">Try again</Button>
      </div>

      <div v-else-if="webhooks.length === 0" class="empty-state py-12 px-4">
        <div class="empty-state-icon">
          <svg class="w-10 h-10 text-rm-muted opacity-60" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
            <path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/>
            <path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/>
          </svg>
        </div>
        <h4 class="empty-state-title">No webhooks configured</h4>
        <p class="empty-state-body text-sm text-rm-muted m-0">Create a webhook in Settings first, then come back here to test it.</p>
      </div>

      <ul v-else class="wht-list list-none m-0 p-0">
        <li
          v-for="wh in webhooks"
          :key="wh.id"
          class="wht-row flex items-start gap-3 px-4 py-3 border-b border-rm-border last:border-b-0 hover:bg-rm-surface-hover/50"
        >
          <div class="min-w-0 flex-1">
            <div class="flex items-center gap-2 mb-1">
              <span class="font-medium text-sm text-rm-text truncate">{{ wh.description || wh.url }}</span>
              <Tag
                :severity="wh.is_active ? 'success' : 'secondary'"
                :value="wh.is_active ? 'Active' : 'Inactive'"
                class="text-xs"
              />
            </div>
            <p class="text-xs text-rm-muted font-mono truncate m-0">{{ wh.url }}</p>
            <div v-if="wh.events && wh.events.length" class="flex flex-wrap gap-1 mt-1.5">
              <span
                v-for="ev in wh.events"
                :key="ev"
                class="px-1.5 py-0.5 rounded text-xs bg-rm-surface-hover text-rm-muted"
              >{{ ev }}</span>
            </div>
          </div>
          <div class="flex items-center gap-1 shrink-0 pt-0.5">
            <Button
              severity="secondary"
              size="small"
              :disabled="testingId === wh.id"
              :loading="testingId === wh.id"
              @click="quickTest(wh)"
            >
              <svg v-if="testingId !== wh.id" class="w-3.5 h-3.5 mr-1" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polygon points="5 3 19 12 5 21 5 3"/></svg>
              Ping
            </Button>
            <Button
              severity="primary"
              size="small"
              @click="openComposer(wh)"
            >
              Test
            </Button>
          </div>
        </li>
      </ul>
    </Panel>

    <!-- Request history -->
    <Panel v-if="history.length" class="wht-history-panel mt-5">
      <template #header>
        <div class="flex items-center justify-between gap-3 w-full">
          <h3 class="text-sm font-semibold text-rm-text m-0 tracking-tight">History</h3>
          <Button variant="text" size="small" class="text-xs text-rm-muted" @click="clearHistory">Clear</Button>
        </div>
      </template>

      <ul class="wht-list list-none m-0 p-0">
        <li
          v-for="(entry, idx) in history"
          :key="idx"
          class="wht-row flex items-start gap-3 px-4 py-3 border-b border-rm-border last:border-b-0 cursor-pointer hover:bg-rm-surface-hover/50"
          @click="viewHistoryEntry(entry)"
        >
          <div class="min-w-0 flex-1">
            <div class="flex items-center gap-2 mb-0.5">
              <Tag
                :severity="getStatusSeverity(entry.status)"
                :value="entry.status ? String(entry.status) : (entry.error ? 'Error' : '—')"
                class="text-xs font-mono"
              />
              <span class="text-xs font-medium text-rm-text truncate">
                {{ entry.method }} {{ entry.label || entry.url }}
              </span>
            </div>
            <p class="text-xs text-rm-muted m-0">
              {{ formatTime(entry.timestamp) }}
              <span v-if="entry.duration" class="ml-2 text-rm-muted">{{ entry.duration }}ms</span>
            </p>
          </div>
          <svg class="w-4 h-4 text-rm-muted shrink-0 mt-1" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="9 18 15 12 9 6"/></svg>
        </li>
      </ul>
    </Panel>

    <!-- Composer dialog -->
    <Dialog
      v-model:visible="composerVisible"
      header="Webhook request"
      :style="{ width: '40rem' }"
      :modal="true"
      :dismissableMask="true"
      class="wht-composer max-w-[95vw]"
    >
      <div class="space-y-4">
        <!-- Method + URL row -->
        <div class="flex gap-2">
          <Select
            v-model="form.method"
            :options="methods"
            class="shrink-0"
            style="width: 7rem"
          />
          <InputText
            v-model="form.url"
            class="text-sm w-full font-mono"
            placeholder="https://example.com/api/webhook"
          />
        </div>

        <!-- Headers -->
        <div>
          <div class="flex items-center justify-between mb-1.5">
            <span class="text-xs font-medium text-rm-muted">Headers</span>
            <Button variant="text" size="small" class="text-xs" @click="addHeader">+ Add</Button>
          </div>
          <div v-for="(h, i) in form.headers" :key="i" class="flex items-center gap-2 mb-1.5">
            <InputText v-model="h.key" class="text-xs font-mono flex-1" placeholder="Header name" />
            <InputText v-model="h.value" class="text-xs font-mono flex-1" placeholder="Value" />
            <Button
              variant="text"
              severity="danger"
              size="small"
              class="p-1 min-w-0 opacity-70 hover:opacity-100"
              @click="form.headers.splice(i, 1)"
            >×</Button>
          </div>
        </div>

        <!-- Body -->
        <div>
          <div class="flex items-center justify-between mb-1.5">
            <span class="text-xs font-medium text-rm-muted">Body</span>
            <SelectButton
              v-model="form.bodyFormat"
              :options="bodyFormats"
              optionLabel="label"
              optionValue="value"
              :allowEmpty="false"
              class="text-xs"
            />
          </div>
          <Textarea
            v-model="form.body"
            class="text-xs font-mono w-full"
            :rows="8"
            placeholder='{ "event": "test", "data": {} }'
            spellcheck="false"
          />
          <p v-if="bodyError" class="text-xs text-rm-danger mt-1 m-0">{{ bodyError }}</p>
        </div>

        <!-- Payload templates -->
        <div>
          <span class="text-xs font-medium text-rm-muted block mb-1.5">Quick templates</span>
          <div class="flex flex-wrap gap-1.5">
            <Button
              v-for="t in templates"
              :key="t.label"
              variant="outlined"
              size="small"
              class="text-xs"
              @click="applyTemplate(t)"
            >{{ t.label }}</Button>
          </div>
        </div>
      </div>

      <template #footer>
        <Button variant="text" size="small" @click="composerVisible = false">Cancel</Button>
        <Button
          severity="primary"
          size="small"
          :disabled="!form.url?.trim() || sending"
          :loading="sending"
          @click="sendRequest"
        >
          Send request
        </Button>
      </template>
    </Dialog>

    <!-- Response viewer dialog -->
    <Dialog
      v-model:visible="responseVisible"
      header="Response"
      :style="{ width: '40rem' }"
      :modal="true"
      :dismissableMask="true"
      class="wht-response max-w-[95vw]"
    >
      <div v-if="activeResponse" class="space-y-4">
        <!-- Status line -->
        <div class="flex items-center gap-3 pb-3 border-b border-rm-border">
          <Tag
            :severity="getStatusSeverity(activeResponse.status)"
            :value="activeResponse.status ? String(activeResponse.status) : 'Error'"
            class="text-sm font-mono"
          />
          <span class="text-sm text-rm-text font-medium">
            {{ activeResponse.statusText || (activeResponse.error ? 'Request failed' : '') }}
          </span>
          <span v-if="activeResponse.duration" class="text-xs text-rm-muted ml-auto">{{ activeResponse.duration }}ms</span>
        </div>

        <!-- Error -->
        <div v-if="activeResponse.error" class="p-3 rounded-rm bg-red-500/10 border border-red-500/20">
          <p class="text-sm text-rm-danger m-0 font-mono">{{ activeResponse.error }}</p>
        </div>

        <!-- Response headers -->
        <div v-if="activeResponse.responseHeaders && Object.keys(activeResponse.responseHeaders).length">
          <span class="text-xs font-medium text-rm-muted block mb-1.5">Response headers</span>
          <div class="bg-rm-surface rounded-rm p-3 max-h-40 overflow-auto">
            <div
              v-for="(val, key) in activeResponse.responseHeaders"
              :key="key"
              class="text-xs font-mono leading-relaxed"
            >
              <span class="text-rm-accent">{{ key }}:</span>
              <span class="text-rm-text ml-1">{{ val }}</span>
            </div>
          </div>
        </div>

        <!-- Response body -->
        <div v-if="activeResponse.responseBody != null">
          <div class="flex items-center justify-between mb-1.5">
            <span class="text-xs font-medium text-rm-muted">Response body</span>
            <Button
              variant="text"
              size="small"
              class="text-xs"
              @click="copyToClipboard(activeResponse.responseBody)"
            >Copy</Button>
          </div>
          <pre class="bg-rm-surface rounded-rm p-3 text-xs font-mono text-rm-text whitespace-pre-wrap break-all max-h-80 overflow-auto m-0">{{ formatResponseBody(activeResponse.responseBody) }}</pre>
        </div>
      </div>
    </Dialog>
  </ExtensionLayout>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';

const Button = window.PrimeVue?.['button'] ?? { name: 'Button', template: '<button><slot /></button>' };
const Dialog = window.PrimeVue?.['dialog'] ?? { name: 'Dialog', template: '<div v-if="visible"><slot /><slot name="footer" /></div>', props: ['visible','header','style','modal','dismissableMask'], emits: ['update:visible'] };
const InputText = window.PrimeVue?.['inputtext'] ?? { name: 'InputText', template: '<input />' };
const Panel = window.PrimeVue?.['panel'] ?? { name: 'Panel', template: '<div><slot /><slot name="header" /></div>' };
const ProgressSpinner = window.PrimeVue?.['progressspinner'] ?? { name: 'ProgressSpinner', template: '<span>Loading…</span>' };
const Select = window.PrimeVue?.['select'] ?? { name: 'Select', template: '<select><slot /></select>' };
const SelectButton = window.PrimeVue?.['selectbutton'] ?? { name: 'SelectButton', template: '<div><slot /></div>' };
const Tag = window.PrimeVue?.['tag'] ?? { name: 'Tag', template: '<span><slot /></span>' };
const Textarea = window.PrimeVue?.['textarea'] ?? { name: 'Textarea', template: '<textarea></textarea>' };
const ExtensionLayout = { name: 'ExtensionLayout', props: { tabId: String, contentClass: String }, template: '<div class="detail-tab-panel" :data-detail-tab="tabId" :class="contentClass"><slot /><slot name="toolbar-start" /><slot name="toolbar-end" /></div>' };

function useApi() { return window.releaseManager ?? {}; }
const props = defineProps({ info: { type: Object, default: null } });

const api = useApi();

const PREF_KEY = 'ext.webhookTester.history';
const MAX_HISTORY = 50;

const loading = ref(false);
const error = ref('');
const webhooks = ref([]);
const availableEvents = ref([]);
const testingId = ref(null);
const history = ref([]);

const composerVisible = ref(false);
const sending = ref(false);
const bodyError = ref('');

const responseVisible = ref(false);
const activeResponse = ref(null);

const methods = ['POST', 'GET', 'PUT', 'PATCH', 'DELETE'];
const bodyFormats = [
  { label: 'JSON', value: 'json' },
  { label: 'Form', value: 'form' },
  { label: 'Raw', value: 'raw' },
];

const defaultForm = () => ({
  method: 'POST',
  url: '',
  headers: [
    { key: 'Content-Type', value: 'application/json' },
  ],
  body: '{\n  "event": "test.ping",\n  "timestamp": "' + new Date().toISOString() + '",\n  "data": {\n    "message": "Hello from Shipwell webhook tester"\n  }\n}',
  bodyFormat: 'json',
});

const form = ref(defaultForm());

const templates = [
  {
    label: 'Ping',
    body: () => JSON.stringify({ event: 'test.ping', timestamp: new Date().toISOString(), data: { message: 'Ping!' } }, null, 2),
  },
  {
    label: 'New release',
    body: () => JSON.stringify({
      event: 'release.created',
      timestamp: new Date().toISOString(),
      data: {
        project: props.info?.name || 'my-project',
        version: props.info?.version || '1.0.0',
        tag: props.info?.latestTag || 'v1.0.0',
        branch: props.info?.branch || 'main',
      },
    }, null, 2),
  },
  {
    label: 'Deploy',
    body: () => JSON.stringify({
      event: 'deploy.completed',
      timestamp: new Date().toISOString(),
      data: {
        project: props.info?.name || 'my-project',
        environment: 'production',
        version: props.info?.version || '1.0.0',
        status: 'success',
        duration_seconds: 42,
      },
    }, null, 2),
  },
  {
    label: 'Error alert',
    body: () => JSON.stringify({
      event: 'error.reported',
      timestamp: new Date().toISOString(),
      data: {
        project: props.info?.name || 'my-project',
        error: 'Unhandled exception in main process',
        severity: 'critical',
        stack: 'TypeError: Cannot read properties of undefined\n  at processRequest (/app/src/handler.js:42)',
        environment: 'production',
      },
    }, null, 2),
  },
  {
    label: 'Slack',
    body: () => JSON.stringify({
      text: `Webhook test from ${props.info?.name || 'Shipwell'}`,
      blocks: [
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `*Webhook Test*\nProject: ${props.info?.name || 'my-project'}\nVersion: ${props.info?.version || '1.0.0'}`,
          },
        },
      ],
    }, null, 2),
  },
  {
    label: 'Discord',
    body: () => JSON.stringify({
      content: `Webhook test from ${props.info?.name || 'Shipwell'}`,
      embeds: [{
        title: 'Webhook Test',
        description: `Project: ${props.info?.name || 'my-project'}`,
        color: 5814783,
        fields: [
          { name: 'Version', value: props.info?.version || '1.0.0', inline: true },
          { name: 'Branch', value: props.info?.branch || 'main', inline: true },
        ],
      }],
    }, null, 2),
  },
];

async function loadWebhooks() {
  loading.value = true;
  error.value = '';
  try {
    const result = await api.getWebhooks?.();
    if (result?.ok) {
      webhooks.value = Array.isArray(result.webhooks) ? result.webhooks : (Array.isArray(result.data) ? result.data : []);
      if (Array.isArray(result.available_events)) availableEvents.value = result.available_events;
    } else {
      error.value = result?.error || 'Failed to load webhooks';
    }
  } catch (e) {
    error.value = e?.message || 'Failed to load webhooks';
  } finally {
    loading.value = false;
  }
}

async function quickTest(wh) {
  testingId.value = wh.id;
  const start = Date.now();
  try {
    const result = await api.testWebhook?.(wh.id);
    const entry = {
      method: 'POST',
      url: wh.url,
      label: wh.description || wh.url,
      timestamp: new Date().toISOString(),
      duration: Date.now() - start,
      status: result?.ok ? (result.status_code || 200) : null,
      statusText: result?.ok ? 'OK' : '',
      error: result?.ok ? null : (result?.error || 'Test failed'),
      responseBody: result?.response_body || result?.body || null,
      responseHeaders: null,
      type: 'quick-test',
    };
    addToHistory(entry);
    activeResponse.value = entry;
    responseVisible.value = true;
  } catch (e) {
    const entry = {
      method: 'POST',
      url: wh.url,
      label: wh.description || wh.url,
      timestamp: new Date().toISOString(),
      duration: Date.now() - start,
      status: null,
      error: e?.message || 'Test failed',
      type: 'quick-test',
    };
    addToHistory(entry);
    activeResponse.value = entry;
    responseVisible.value = true;
  } finally {
    testingId.value = null;
  }
}

function openComposer(wh) {
  const f = defaultForm();
  if (wh) {
    f.url = wh.url || '';
  }
  form.value = f;
  bodyError.value = '';
  composerVisible.value = true;
}

function addHeader() {
  form.value.headers.push({ key: '', value: '' });
}

function applyTemplate(t) {
  form.value.body = typeof t.body === 'function' ? t.body() : t.body;
  form.value.bodyFormat = 'json';
  const ct = form.value.headers.find((h) => h.key.toLowerCase() === 'content-type');
  if (ct) ct.value = 'application/json';
}

function validateBody() {
  bodyError.value = '';
  if (form.value.bodyFormat === 'json' && form.value.body?.trim()) {
    try {
      JSON.parse(form.value.body);
    } catch (e) {
      bodyError.value = 'Invalid JSON: ' + e.message;
      return false;
    }
  }
  return true;
}

async function sendRequest() {
  if (!form.value.url?.trim()) return;
  if (!validateBody()) return;

  sending.value = true;
  const start = Date.now();
  const headers = {};
  for (const h of form.value.headers) {
    if (h.key?.trim()) headers[h.key.trim()] = h.value || '';
  }

  let body = form.value.body?.trim() || undefined;
  if (form.value.method === 'GET' || form.value.method === 'DELETE') {
    body = undefined;
  }

  try {
    const resp = await fetch(form.value.url.trim(), {
      method: form.value.method,
      headers,
      body,
      signal: AbortSignal.timeout(30000),
    });

    const respHeaders = {};
    resp.headers.forEach((v, k) => { respHeaders[k] = v; });
    let respBody = '';
    try { respBody = await resp.text(); } catch (_) {}

    const entry = {
      method: form.value.method,
      url: form.value.url.trim(),
      label: form.value.url.trim(),
      timestamp: new Date().toISOString(),
      duration: Date.now() - start,
      status: resp.status,
      statusText: resp.statusText,
      error: null,
      responseHeaders: respHeaders,
      responseBody: respBody,
      requestHeaders: headers,
      requestBody: body || null,
      type: 'custom',
    };
    addToHistory(entry);
    activeResponse.value = entry;
    composerVisible.value = false;
    responseVisible.value = true;
  } catch (e) {
    const entry = {
      method: form.value.method,
      url: form.value.url.trim(),
      label: form.value.url.trim(),
      timestamp: new Date().toISOString(),
      duration: Date.now() - start,
      status: null,
      error: e?.message || 'Request failed',
      type: 'custom',
    };
    addToHistory(entry);
    activeResponse.value = entry;
    composerVisible.value = false;
    responseVisible.value = true;
  } finally {
    sending.value = false;
  }
}

function addToHistory(entry) {
  history.value = [entry, ...history.value].slice(0, MAX_HISTORY);
  saveHistory();
}

function clearHistory() {
  history.value = [];
  saveHistory();
}

function viewHistoryEntry(entry) {
  activeResponse.value = entry;
  responseVisible.value = true;
}

function getStatusSeverity(status) {
  if (!status) return 'danger';
  const code = Number(status);
  if (code >= 200 && code < 300) return 'success';
  if (code >= 300 && code < 400) return 'warn';
  return 'danger';
}

function formatTime(ts) {
  if (!ts) return '';
  try {
    const d = new Date(ts);
    return d.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' }) + ' ' + d.toLocaleDateString([], { month: 'short', day: 'numeric' });
  } catch (_) { return ts; }
}

function formatResponseBody(body) {
  if (!body) return '';
  try {
    return JSON.stringify(JSON.parse(body), null, 2);
  } catch (_) {
    return body;
  }
}

function copyToClipboard(text) {
  try { navigator.clipboard.writeText(text); } catch (_) {}
}

async function loadHistory() {
  if (!api.getPreference) return;
  try {
    const raw = await api.getPreference(PREF_KEY);
    const data = raw && typeof raw === 'string' ? JSON.parse(raw) : raw;
    history.value = Array.isArray(data) ? data.slice(0, MAX_HISTORY) : [];
  } catch (_) {
    history.value = [];
  }
}

async function saveHistory() {
  if (!api.setPreference) return;
  try {
    await api.setPreference(PREF_KEY, JSON.stringify(history.value));
  } catch (_) {}
}

onMounted(() => {
  loadWebhooks();
  loadHistory();
});

watch(() => props.info?.path, () => {
  loadWebhooks();
});
</script>

<style scoped>
.wht-list {
  list-style: none;
  padding: 0;
  margin: 0;
}
.wht-webhooks-panel :deep(.p-panel-content-wrapper),
.wht-webhooks-panel :deep(.p-panel-content),
.wht-history-panel :deep(.p-panel-content-wrapper),
.wht-history-panel :deep(.p-panel-content) {
  border-top: none;
}
.empty-state-icon {
  display: flex;
  justify-content: center;
  margin-bottom: 0.75rem;
}
.empty-state-title {
  font-size: 0.9375rem;
  font-weight: 600;
  color: rgb(var(--rm-text));
  margin: 0 0 0.25rem;
  text-align: center;
}
.empty-state-body {
  margin: 0;
  text-align: center;
}
</style>
