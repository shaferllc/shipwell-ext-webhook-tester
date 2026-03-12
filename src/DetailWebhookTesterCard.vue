<template>
  <ExtensionLayout tab-id="webhook-tester" content-class="detail-webhook-tester">
    <template #toolbar-start>
      <p class="text-sm text-rm-muted m-0">
        Capture and inspect incoming HTTP requests — like webhook.site, built in.
      </p>
    </template>
    <template #toolbar-end>
      <div class="flex items-center gap-2">
        <Button severity="secondary" size="small" :disabled="loading" @click="refresh">
          <svg class="w-3.5 h-3.5 mr-1" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M23 4v6h-6"/><path d="M1 20v-6h6"/><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"/></svg>
          Refresh
        </Button>
        <Button severity="primary" size="small" @click="createEndpoint">
          New endpoint
        </Button>
      </div>
    </template>

    <!-- Loading -->
    <div v-if="loading && !endpoints.length" class="flex items-center justify-center py-10">
      <ProgressSpinner style="width: 32px; height: 32px" strokeWidth="3" />
    </div>

    <!-- Error -->
    <div v-else-if="error && !endpoints.length" class="py-6 px-4 text-center">
      <p class="text-sm text-rm-danger m-0">{{ error }}</p>
      <Button severity="secondary" size="small" class="mt-3" @click="refresh">Try again</Button>
    </div>

    <!-- No active endpoint selected — show endpoint list -->
    <div v-else-if="!activeEndpoint">
      <!-- Empty state -->
      <div v-if="!endpoints.length" class="empty-state py-12 px-4">
        <div class="empty-state-icon">
          <svg class="w-10 h-10 text-rm-muted opacity-60" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
            <path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/>
            <path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/>
          </svg>
        </div>
        <h4 class="empty-state-title">No webhook endpoints</h4>
        <p class="empty-state-body text-sm text-rm-muted m-0">Create an endpoint to get a unique URL that captures incoming HTTP requests.</p>
        <div class="empty-state-actions mt-3">
          <Button severity="primary" size="small" @click="createEndpoint">Create endpoint</Button>
        </div>
      </div>

      <!-- Endpoint list -->
      <Panel v-else class="wht-endpoints-panel">
        <template #header>
          <div class="flex items-center justify-between gap-3 w-full">
            <h3 class="text-sm font-semibold text-rm-text m-0 tracking-tight">Endpoints</h3>
            <span class="text-xs text-rm-muted">{{ endpoints.length }} endpoint{{ endpoints.length === 1 ? '' : 's' }}</span>
          </div>
        </template>
        <ul class="wht-list list-none m-0 p-0">
          <li
            v-for="ep in endpoints"
            :key="ep.id"
            class="wht-row flex items-start gap-3 px-4 py-3 border-b border-rm-border last:border-b-0 cursor-pointer hover:bg-rm-surface-hover/50"
            @click="selectEndpoint(ep)"
          >
            <div class="min-w-0 flex-1">
              <div class="flex items-center gap-2 mb-0.5">
                <span class="font-medium text-sm text-rm-text">{{ ep.name || 'Unnamed endpoint' }}</span>
                <Tag :severity="ep.is_active ? 'success' : 'secondary'" :value="ep.is_active ? 'Active' : 'Inactive'" class="text-xs" />
                <span class="text-xs text-rm-muted">{{ ep.requests_count ?? 0 }} req{{ (ep.requests_count ?? 0) === 1 ? '' : 's' }}</span>
              </div>
              <p class="text-xs text-rm-accent font-mono truncate m-0">{{ getEndpointUrl(ep) }}</p>
            </div>
            <svg class="w-4 h-4 text-rm-muted shrink-0 mt-1" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="9 18 15 12 9 6"/></svg>
          </li>
        </ul>
      </Panel>
    </div>

    <!-- Active endpoint detail view -->
    <div v-else>
      <!-- Back + endpoint header -->
      <div class="flex items-center gap-2 mb-4">
        <Button variant="text" size="small" class="p-1 min-w-0" @click="activeEndpoint = null; stopPolling()">
          <svg class="w-4 h-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="15 18 9 12 15 6"/></svg>
        </Button>
        <span class="font-medium text-sm text-rm-text">{{ activeEndpoint.name || 'Unnamed endpoint' }}</span>
        <Tag :severity="activeEndpoint.is_active ? 'success' : 'secondary'" :value="activeEndpoint.is_active ? 'Active' : 'Inactive'" class="text-xs" />
        <span class="text-xs text-rm-muted ml-auto">{{ requests.length }} request{{ requests.length === 1 ? '' : 's' }}</span>
      </div>

      <!-- URL box -->
      <div class="wht-url-box rounded-rm-lg border border-rm-accent/20 bg-rm-accent/5 p-4 mb-4">
        <div class="flex items-center gap-3">
          <div class="min-w-0 flex-1">
            <p class="text-xs text-rm-muted m-0 mb-1">Webhook URL</p>
            <code class="text-sm text-rm-accent font-mono break-all">{{ getEndpointUrl(activeEndpoint) }}</code>
          </div>
          <Button severity="secondary" size="small" @click="copyUrl(activeEndpoint)">
            {{ justCopied ? '✓ Copied' : 'Copy' }}
          </Button>
        </div>
      </div>

      <!-- Config summary -->
      <div class="grid grid-cols-3 gap-3 mb-4 text-xs">
        <div class="bg-rm-surface rounded-rm p-2.5">
          <span class="text-rm-muted block mb-0.5">Status code</span>
          <span class="text-rm-text font-mono">{{ activeEndpoint.default_status_code ?? 200 }}</span>
        </div>
        <div class="bg-rm-surface rounded-rm p-2.5">
          <span class="text-rm-muted block mb-0.5">Content-Type</span>
          <span class="text-rm-text font-mono truncate block">{{ activeEndpoint.default_content_type ?? 'application/json' }}</span>
        </div>
        <div class="bg-rm-surface rounded-rm p-2.5">
          <span class="text-rm-muted block mb-0.5">Delay</span>
          <span class="text-rm-text font-mono">{{ activeEndpoint.timeout_seconds ?? 0 }}s</span>
        </div>
      </div>

      <!-- Action bar -->
      <div class="flex items-center gap-2 mb-4">
        <Button v-if="requests.length" severity="danger" variant="outlined" size="small" @click="clearAllRequests">Clear all</Button>
        <Button severity="danger" variant="text" size="small" class="ml-auto" @click="deleteEndpoint">Delete endpoint</Button>
      </div>

      <!-- Requests panel -->
      <Panel class="wht-requests-panel">
        <template #header>
          <div class="flex items-center justify-between gap-3 w-full">
            <h3 class="text-sm font-semibold text-rm-text m-0 tracking-tight">Captured Requests</h3>
            <span v-if="polling" class="flex items-center gap-1.5 text-xs text-rm-muted">
              <span class="w-1.5 h-1.5 rounded-full bg-green-400 animate-pulse"></span>
              Live
            </span>
          </div>
        </template>

        <div v-if="loadingRequests && !requests.length" class="flex items-center justify-center py-8">
          <ProgressSpinner style="width: 24px; height: 24px" strokeWidth="3" />
        </div>

        <div v-else-if="!requests.length" class="py-10 px-4 text-center">
          <svg class="w-8 h-8 mx-auto text-rm-muted opacity-40 mb-3 animate-pulse" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5"><path d="M8.288 15.038a5.25 5.25 0 017.424 0M5.106 11.856c3.807-3.808 9.98-3.808 13.788 0M1.924 8.674c5.565-5.565 14.587-5.565 20.152 0M12 19.5v.75"/></svg>
          <p class="text-sm text-rm-text font-medium m-0">Waiting for requests…</p>
          <p class="text-xs text-rm-muted mt-1 m-0">Send a request to the URL above to see it here.</p>
        </div>

        <ul v-else class="wht-list list-none m-0 p-0">
          <li
            v-for="req in requests"
            :key="req.id"
            class="wht-row border-b border-rm-border last:border-b-0"
          >
            <div
              class="flex items-center gap-3 px-4 py-2.5 cursor-pointer hover:bg-rm-surface-hover/50"
              @click="expandedId = expandedId === req.id ? null : req.id"
            >
              <span
                class="text-[10px] px-1.5 py-0.5 rounded font-mono font-medium border shrink-0 w-14 text-center"
                :class="methodColor(req.method)"
              >{{ req.method }}</span>
              <span class="text-xs text-rm-text whitespace-nowrap">{{ formatTime(req.created_at) }}</span>
              <span class="text-xs text-rm-muted font-mono hidden sm:inline">{{ req.source_ip }}</span>
              <span v-if="req.content_type" class="text-[10px] text-rm-muted font-mono hidden md:inline truncate max-w-[180px]">{{ req.content_type }}</span>
              <span v-if="req.size_bytes" class="text-[10px] text-rm-muted hidden lg:inline">{{ req.size_bytes }} B</span>
              <svg class="w-3.5 h-3.5 text-rm-muted ml-auto shrink-0 transition-transform" :class="expandedId === req.id ? 'rotate-180' : ''" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M19 9l-7 7-7-7"/></svg>
            </div>

            <!-- Expanded detail -->
            <div v-if="expandedId === req.id" class="border-t border-rm-border px-4 py-3 space-y-3 bg-rm-surface/30">
              <div class="text-[10px] text-rm-muted">
                {{ req.created_at }} · {{ req.source_ip }}
                <span v-if="req.size_bytes"> · {{ req.size_bytes }} bytes</span>
              </div>

              <!-- Query params -->
              <div v-if="req.query_params && Object.keys(req.query_params).length">
                <h4 class="text-xs text-rm-muted mb-1.5 font-medium">Query Parameters</h4>
                <div class="bg-rm-surface rounded-rm p-2.5 text-xs font-mono">
                  <div v-for="(val, key) in req.query_params" :key="key">
                    <span class="text-rm-accent">{{ key }}:</span>
                    <span class="text-rm-text ml-1">{{ typeof val === 'object' ? JSON.stringify(val) : val }}</span>
                  </div>
                </div>
              </div>

              <!-- Headers -->
              <div v-if="req.headers && Object.keys(req.headers).length">
                <h4 class="text-xs text-rm-muted mb-1.5 font-medium">Headers</h4>
                <div class="bg-rm-surface rounded-rm p-2.5 max-h-48 overflow-auto text-xs font-mono">
                  <div v-for="(val, key) in req.headers" :key="key">
                    <span class="text-rm-accent">{{ key }}:</span>
                    <span class="text-rm-text ml-1">{{ Array.isArray(val) ? val.join(', ') : val }}</span>
                  </div>
                </div>
              </div>

              <!-- Body -->
              <div v-if="req.body">
                <div class="flex items-center justify-between mb-1.5">
                  <h4 class="text-xs text-rm-muted font-medium m-0">Body</h4>
                  <Button variant="text" size="small" class="text-xs p-0" @click="copyText(req.body)">Copy</Button>
                </div>
                <pre class="bg-rm-surface rounded-rm p-2.5 text-xs font-mono text-rm-text whitespace-pre-wrap break-all max-h-64 overflow-auto m-0">{{ formatBody(req.body, req.content_type) }}</pre>
              </div>
            </div>
          </li>
        </ul>
      </Panel>
    </div>

    <!-- Create endpoint dialog -->
    <Dialog
      v-model:visible="createVisible"
      header="New webhook endpoint"
      :style="{ width: '28rem' }"
      :modal="true"
      :dismissableMask="true"
      class="wht-create-dialog max-w-[95vw]"
    >
      <div class="space-y-3">
        <label class="block">
          <span class="text-xs font-medium text-rm-muted block mb-1">Name (optional)</span>
          <InputText v-model="createForm.name" class="text-sm w-full" placeholder="e.g. Stripe webhooks" />
        </label>
        <label class="block">
          <span class="text-xs font-medium text-rm-muted block mb-1">Description (optional)</span>
          <Textarea v-model="createForm.description" class="text-sm w-full" rows="2" placeholder="What is this endpoint for?" />
        </label>
      </div>
      <template #footer>
        <Button variant="text" size="small" @click="createVisible = false">Cancel</Button>
        <Button severity="primary" size="small" :disabled="creating" :loading="creating" @click="submitCreate">Create</Button>
      </template>
    </Dialog>
  </ExtensionLayout>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue';

const Button = window.PrimeVue?.['button'] ?? { name: 'Button', template: '<button><slot /></button>' };
const Dialog = window.PrimeVue?.['dialog'] ?? { name: 'Dialog', template: '<div v-if="visible"><slot /><slot name="footer" /></div>', props: ['visible','header','style','modal','dismissableMask'], emits: ['update:visible'] };
const InputText = window.PrimeVue?.['inputtext'] ?? { name: 'InputText', template: '<input />' };
const Panel = window.PrimeVue?.['panel'] ?? { name: 'Panel', template: '<div><slot /><slot name="header" /></div>' };
const ProgressSpinner = window.PrimeVue?.['progressspinner'] ?? { name: 'ProgressSpinner', template: '<span>Loading…</span>' };
const Tag = window.PrimeVue?.['tag'] ?? { name: 'Tag', template: '<span><slot /></span>' };
const Textarea = window.PrimeVue?.['textarea'] ?? { name: 'Textarea', template: '<textarea></textarea>' };
const ExtensionLayout = { name: 'ExtensionLayout', props: { tabId: String, contentClass: String }, template: '<div class="detail-tab-panel" :data-detail-tab="tabId" :class="contentClass"><slot /><slot name="toolbar-start" /><slot name="toolbar-end" /></div>' };

function useApi() { return window.releaseManager ?? {}; }
const props = defineProps({ info: { type: Object, default: null } });
const api = useApi();

const loading = ref(false);
const error = ref('');
const endpoints = ref([]);
const activeEndpoint = ref(null);
const requests = ref([]);
const loadingRequests = ref(false);
const expandedId = ref(null);
const polling = ref(false);
const justCopied = ref(false);

const createVisible = ref(false);
const creating = ref(false);
const createForm = ref({ name: '', description: '' });

let pollTimer = null;
const POLL_INTERVAL = 3000;

function getServerUrl() {
  const cfg = api.getLicenseServerUrl?.() || '';
  return typeof cfg === 'string' ? cfg.replace(/\/+$/, '') : '';
}

function getEndpointUrl(ep) {
  const base = getServerUrl();
  return base ? `${base}/webhook-test/${ep.uuid}` : `/webhook-test/${ep.uuid}`;
}

async function loadEndpoints() {
  loading.value = true;
  error.value = '';
  try {
    const result = await api.getWebhookTestEndpoints?.();
    if (result?.ok) {
      endpoints.value = Array.isArray(result.data) ? result.data : [];
    } else {
      error.value = result?.error || 'Failed to load endpoints';
    }
  } catch (e) {
    error.value = e?.message || 'Failed';
  } finally {
    loading.value = false;
  }
}

async function loadRequests() {
  if (!activeEndpoint.value) return;
  loadingRequests.value = true;
  try {
    const result = await api.getWebhookTestRequests?.(activeEndpoint.value.uuid || activeEndpoint.value.id);
    if (result?.ok) {
      requests.value = Array.isArray(result.data) ? result.data : [];
    }
  } catch (_) {}
  loadingRequests.value = false;
}

function selectEndpoint(ep) {
  activeEndpoint.value = ep;
  requests.value = [];
  expandedId.value = null;
  loadRequests();
  startPolling();
}

function startPolling() {
  stopPolling();
  polling.value = true;
  pollTimer = setInterval(() => {
    if (activeEndpoint.value) loadRequests();
  }, POLL_INTERVAL);
}

function stopPolling() {
  polling.value = false;
  if (pollTimer) { clearInterval(pollTimer); pollTimer = null; }
}

function refresh() {
  if (activeEndpoint.value) {
    loadRequests();
  } else {
    loadEndpoints();
  }
}

function createEndpoint() {
  createForm.value = { name: '', description: '' };
  createVisible.value = true;
}

async function submitCreate() {
  creating.value = true;
  try {
    const result = await api.createWebhookTestEndpoint?.({
      name: createForm.value.name?.trim() || null,
      description: createForm.value.description?.trim() || null,
    });
    if (result?.ok && result.data) {
      createVisible.value = false;
      await loadEndpoints();
      selectEndpoint(result.data);
    }
  } catch (_) {}
  creating.value = false;
}

async function deleteEndpoint() {
  if (!activeEndpoint.value) return;
  if (!confirm('Delete this endpoint and all captured requests?')) return;
  try {
    await api.deleteWebhookTestEndpoint?.(activeEndpoint.value.uuid || activeEndpoint.value.id);
    stopPolling();
    activeEndpoint.value = null;
    loadEndpoints();
  } catch (_) {}
}

async function clearAllRequests() {
  if (!activeEndpoint.value) return;
  if (!confirm('Clear all captured requests?')) return;
  try {
    await api.clearWebhookTestRequests?.(activeEndpoint.value.uuid || activeEndpoint.value.id);
    requests.value = [];
  } catch (_) {}
}

function copyUrl(ep) {
  try {
    navigator.clipboard.writeText(getEndpointUrl(ep));
    justCopied.value = true;
    setTimeout(() => { justCopied.value = false; }, 2000);
  } catch (_) {}
}

function copyText(text) {
  try { navigator.clipboard.writeText(text); } catch (_) {}
}

const METHOD_COLORS = {
  GET: 'wht-method-get',
  POST: 'wht-method-post',
  PUT: 'wht-method-put',
  PATCH: 'wht-method-patch',
  DELETE: 'wht-method-delete',
  HEAD: 'wht-method-head',
  OPTIONS: 'wht-method-options',
};

function methodColor(method) {
  return METHOD_COLORS[method] || 'wht-method-default';
}

function formatTime(ts) {
  if (!ts) return '';
  try {
    const d = new Date(ts);
    return d.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
  } catch (_) { return ts; }
}

function formatBody(body, contentType) {
  if (!body) return '';
  if (contentType && contentType.includes('json')) {
    try { return JSON.stringify(JSON.parse(body), null, 2); } catch (_) {}
  }
  return body;
}

onMounted(() => {
  loadEndpoints();
});

onUnmounted(() => {
  stopPolling();
});

watch(() => props.info?.path, () => {
  stopPolling();
  activeEndpoint.value = null;
});
</script>

<style scoped>
.wht-list { list-style: none; padding: 0; margin: 0; }
.wht-endpoints-panel :deep(.p-panel-content-wrapper),
.wht-endpoints-panel :deep(.p-panel-content),
.wht-requests-panel :deep(.p-panel-content-wrapper),
.wht-requests-panel :deep(.p-panel-content) { border-top: none; }
.empty-state-icon { display: flex; justify-content: center; margin-bottom: 0.75rem; }
.empty-state-title { font-size: 0.9375rem; font-weight: 600; color: rgb(var(--rm-text)); margin: 0 0 0.25rem; text-align: center; }
.empty-state-body { margin: 0; text-align: center; }

.wht-method-get    { background: rgba(59,130,246,.12); color: #60a5fa; border-color: rgba(59,130,246,.25); }
.wht-method-post   { background: rgba(34,197,94,.12); color: #4ade80; border-color: rgba(34,197,94,.25); }
.wht-method-put    { background: rgba(249,115,22,.12); color: #fb923c; border-color: rgba(249,115,22,.25); }
.wht-method-patch  { background: rgba(234,179,8,.12); color: #facc15; border-color: rgba(234,179,8,.25); }
.wht-method-delete { background: rgba(239,68,68,.12); color: #f87171; border-color: rgba(239,68,68,.25); }
.wht-method-head   { background: rgba(168,85,247,.12); color: #c084fc; border-color: rgba(168,85,247,.25); }
.wht-method-options, .wht-method-default { background: rgba(148,163,184,.12); color: #94a3b8; border-color: rgba(148,163,184,.25); }
</style>
