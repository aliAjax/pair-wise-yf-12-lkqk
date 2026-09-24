<script setup lang="ts">
import { computed, reactive, ref } from "vue";

type Field = {
  key: string;
  label: string;
  type?: "number" | "date" | "select";
  options?: readonly string[];
};

const STATUS = {
  OPEN: "营业中",
  SUSPENDED: "暂停营业",
  LOW: "库存紧张",
  RESTOCKING: "补货中"
} as const;

type FlowStatus = (typeof STATUS)[keyof typeof STATUS];
type FlowAction = "low" | "suspend" | "resume" | "restock" | "arrive";

type HistoryEntry = {
  id: string;
  at: string;
  action: FlowAction;
  fromStatus: string;
  toStatus: string;
  operator: string;
  remark: string;
  eta?: string;
  owner?: string;
};

type RecordItem = {
  id: string;
  status: string;
  notes: string;
  createdAt: string;
  history: HistoryEntry[];
  restockEta?: string;
  restockOwner?: string;
  [key: string]: string | number | HistoryEntry[] | undefined;
};

const project = {
  "number": 21,
  "folder": "hxwl/frontend/hxwlfront-21",
  "framework": "vue",
  "title": "油站网点地图管理",
  "subtitle": "维护油站位置、营业状态和库存摘要。",
  "industry": "石油",
  "stack": [
    "Vue3",
    "Vite",
    "TypeScript",
    "Element Plus",
    "Leaflet"
  ],
  "storageKey": "hxwlfront-21-station-map",
  "operatorKey": "hxwlfront-21-operator",
  "formTitle": "新增油站",
  "primaryAction": "保存油站",
  "entityLabel": "油站",
  "statuses": [
    "营业中",
    "暂停营业",
    "库存紧张",
    "补货中"
  ],
  "filters": [
    "全部区域",
    "东区",
    "西区",
    "机场线"
  ],
  "fields": [
    {
      "key": "station",
      "label": "油站名称"
    },
    {
      "key": "area",
      "label": "区域",
      "type": "select",
      "options": [
        "东区",
        "西区",
        "机场线"
      ]
    },
    {
      "key": "stock",
      "label": "库存摘要L",
      "type": "number"
    },
    {
      "key": "manager",
      "label": "负责人"
    }
  ],
  "records": [
    {
      "station": "东区一站",
      "area": "东区",
      "stock": 36000,
      "manager": "刘站长",
      "status": "营业中",
      "notes": "库存正常"
    },
    {
      "station": "机场快线站",
      "area": "机场线",
      "stock": 9000,
      "manager": "王站长",
      "status": "库存紧张",
      "notes": "柴油待补"
    }
  ],
  "metricLabels": [
    "油站数",
    "营业中",
    "库存紧张"
  ]
} as const;

const fields = project.fields as readonly Field[];
const statuses: string[] = [...project.statuses];

// 允许的状态流转：暂停营业不在任何补货入口的来源中，从根上保证“暂停不能补货”
const transitions: Record<FlowAction, { from: readonly FlowStatus[]; to: FlowStatus; label: string }> = {
  low: { from: [STATUS.OPEN], to: STATUS.LOW, label: "标记库存不足" },
  restock: { from: [STATUS.LOW], to: STATUS.RESTOCKING, label: "登记补货" },
  arrive: { from: [STATUS.RESTOCKING], to: STATUS.OPEN, label: "到货确认并恢复营业" },
  suspend: { from: [STATUS.OPEN, STATUS.LOW], to: STATUS.SUSPENDED, label: "暂停营业" },
  resume: { from: [STATUS.SUSPENDED], to: STATUS.OPEN, label: "恢复营业" }
};

const actionButtons: Record<string, { action: FlowAction; label: string; cls?: string }[]> = {
  [STATUS.OPEN]: [
    { action: "low", label: "标记库存不足" },
    { action: "suspend", label: "暂停营业", cls: "danger" }
  ],
  [STATUS.LOW]: [
    { action: "restock", label: "登记补货" },
    { action: "suspend", label: "暂停营业", cls: "danger" }
  ],
  [STATUS.RESTOCKING]: [
    { action: "arrive", label: "到货确认并恢复营业" }
  ],
  [STATUS.SUSPENDED]: [
    { action: "resume", label: "恢复营业" }
  ]
};

function createBlank() {
  return Object.fromEntries(fields.map((field) => [field.key, field.type === "number" ? 0 : ""]));
}

function normalize(raw: Partial<RecordItem>): RecordItem {
  return {
    ...(raw as RecordItem),
    status: statuses.includes(raw.status ?? "") ? String(raw.status) : STATUS.OPEN,
    history: Array.isArray(raw.history) ? raw.history : []
  };
}

function loadRecords(): RecordItem[] {
  const raw = localStorage.getItem(project.storageKey);
  if (!raw) {
    return project.records.map((record, index) => ({
      ...record,
      id: `seed-${index + 1}`,
      history: [],
      createdAt: new Date(Date.now() - index * 86400000).toISOString()
    })) as RecordItem[];
  }
  try {
    const parsed = JSON.parse(raw) as Partial<RecordItem>[];
    return Array.isArray(parsed) ? parsed.map(normalize) : [];
  } catch {
    return [];
  }
}

const records = ref<RecordItem[]>(loadRecords());
const form = reactive<Record<string, string | number>>(createBlank());
const note = ref("");
const filter = ref(project.filters[0]);
const operator = ref(localStorage.getItem(project.operatorKey) ?? "");

// 每个油站展开的补货登记表
const restockForms = reactive<Record<string, { eta: string; owner: string }>>({});
// 幂等：同一油站同一动作 1 秒内的重复点击直接忽略
const lastInvoke = new Map<string, number>();

const toast = ref<{ message: string; type: "error" | "success" } | null>(null);
let toastTimer: ReturnType<typeof setTimeout> | undefined;

function showToast(message: string, type: "error" | "success" = "error") {
  toast.value = { message, type };
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => {
    toast.value = null;
  }, 2600);
}

function saveOperator() {
  localStorage.setItem(project.operatorKey, operator.value.trim());
}

const filteredRecords = computed(() => {
  if (filter.value.startsWith("全部")) return records.value;
  return records.value.filter((record) => Object.values(record).includes(filter.value));
});

const metrics = computed(() => {
  const total = records.value.length;
  const openCount = records.value.filter((record) => record.status === STATUS.OPEN).length;
  const lowCount = records.value.filter((record) => record.status === STATUS.LOW).length;
  return [total, openCount, lowCount];
});

const chartRows = computed(() => statuses.map((status) => ({
  status,
  value: records.value.filter((record) => record.status === status).length
})));

const maxChart = computed(() => Math.max(1, ...chartRows.value.map((row) => row.value)));

function persist() {
  localStorage.setItem(project.storageKey, JSON.stringify(records.value));
}

function pad(num: number) {
  return String(num).padStart(2, "0");
}

function formatTime(iso: string) {
  const d = new Date(iso);
  if (Number.isNaN(d.getTime())) return iso;
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())} ${pad(d.getHours())}:${pad(d.getMinutes())}`;
}

function toLocalInput(d: Date) {
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}T${pad(d.getHours())}:${pad(d.getMinutes())}`;
}

function defaultEta() {
  return toLocalInput(new Date(Date.now() + 4 * 3600_000));
}

function primaryText(record: RecordItem) {
  const first = fields[0];
  const second = fields[1];
  return [record[first.key], record[second.key]].filter(Boolean).join(" / ") || project.entityLabel;
}

function buttonsOf(record: RecordItem) {
  return actionButtons[record.status] ?? [];
}

function historyOf(record: RecordItem) {
  return record.history.slice(0, 5);
}

function etaOverdue(record: RecordItem) {
  return Boolean(record.restockEta && new Date(record.restockEta).getTime() < Date.now());
}

const statusClassMap: Record<string, string> = {
  [STATUS.OPEN]: "st-open",
  [STATUS.SUSPENDED]: "st-suspend",
  [STATUS.LOW]: "st-low",
  [STATUS.RESTOCKING]: "st-restock"
};

function statusClass(status: string) {
  return statusClassMap[status] ?? "";
}

function submit() {
  records.value = [
    {
      ...form,
      id: crypto.randomUUID(),
      status: STATUS.OPEN,
      notes: note.value || "暂无备注",
      history: [],
      createdAt: new Date().toISOString()
    } as RecordItem,
    ...records.value
  ];
  Object.assign(form, createBlank());
  note.value = "";
  persist();
  showToast("油站已保存", "success");
}

/**
 * 执行一次状态流转。返回是否真正产生了变更。
 * 三道防线保证重复点击不会多出变更记录：
 * 1. 操作人必填；2. 当前状态必须是该动作允许的来源；3. 同记录同动作 1 秒内只受理一次。
 */
function applyFlow(record: RecordItem, action: FlowAction, payload?: { eta?: string; owner?: string }): boolean {
  const who = operator.value.trim();
  if (!who) {
    showToast("请先在列表上方填写操作人");
    return false;
  }

  const rule = transitions[action];
  if (!rule.from.includes(record.status as FlowStatus)) {
    if (action === "restock") {
      showToast("暂停营业期间不能登记补货，请先恢复营业");
    } else {
      showToast(`当前状态为「${record.status}」，不能执行「${rule.label}」`);
    }
    return false;
  }

  const invokeKey = `${record.id}:${action}`;
  const now = Date.now();
  const last = lastInvoke.get(invokeKey) ?? 0;
  if (now - last < 1000) {
    return false;
  }
  lastInvoke.set(invokeKey, now);

  const fromStatus = record.status;
  const toStatus = rule.to;
  let remark = rule.label;
  if (action === "restock" && payload?.eta && payload?.owner) {
    record.restockEta = payload.eta;
    record.restockOwner = payload.owner;
    remark = `登记补货，预计到货 ${formatTime(payload.eta)}，负责人 ${payload.owner}`;
  }
  if (action === "arrive") {
    remark = record.restockOwner
      ? `确认到货并恢复营业，补货负责人 ${record.restockOwner}`
      : "确认到货并恢复营业";
  }

  const entry: HistoryEntry = {
    id: crypto.randomUUID(),
    at: new Date().toISOString(),
    action,
    fromStatus,
    toStatus,
    operator: who,
    remark,
    eta: action === "restock" ? payload?.eta : record.restockEta,
    owner: action === "restock" ? payload?.owner : record.restockOwner
  };
  record.history.unshift(entry);
  record.status = toStatus;
  if (action === "arrive") {
    record.restockEta = undefined;
    record.restockOwner = undefined;
  }
  persist();
  showToast(`已${rule.label}：${fromStatus} → ${toStatus}`, "success");
  return true;
}

function openRestock(record: RecordItem) {
  if (record.status !== STATUS.LOW) {
    showToast("只有库存不足的油站才能登记补货，暂停营业请先恢复");
    return;
  }
  if (!restockForms[record.id]) {
    restockForms[record.id] = {
      eta: defaultEta(),
      owner: String(record.manager ?? operator.value ?? "")
    };
  }
}

function submitRestock(record: RecordItem) {
  const draft = restockForms[record.id];
  if (!draft) return;
  const etaDate = new Date(draft.eta);
  if (!draft.eta || Number.isNaN(etaDate.getTime())) {
    showToast("请选择预计到货时间");
    return;
  }
  if (etaDate.getTime() < Date.now()) {
    showToast("预计到货时间不能早于当前时间");
    return;
  }
  const owner = draft.owner.trim();
  if (!owner) {
    showToast("请填写补货负责人");
    return;
  }
  if (applyFlow(record, "restock", { eta: draft.eta, owner })) {
    delete restockForms[record.id];
  }
}

function cancelRestock(record: RecordItem) {
  delete restockForms[record.id];
}

function remove(id: string) {
  records.value = records.value.filter((record) => record.id !== id);
  delete restockForms[id];
  persist();
}
</script>

<template>
  <main class="app">
    <div v-if="toast" class="toast" :class="toast.type">{{ toast.message }}</div>
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">{{ project.industry }}行业前端最小闭环</p>
          <h1>{{ project.title }}</h1>
          <p class="subtitle">{{ project.subtitle }}</p>
        </div>
        <div class="stack">
          <span v-for="item in project.stack" :key="item" class="tag">{{ item }}</span>
        </div>
      </header>

      <section class="metrics">
        <article v-for="(label, index) in project.metricLabels" :key="label" class="metric">
          <span>{{ label }}</span>
          <strong>{{ metrics[index] }}</strong>
        </article>
      </section>

      <section class="workspace">
        <form class="panel" @submit.prevent="submit">
          <h2>{{ project.formTitle }}</h2>
          <div class="form-grid">
            <label v-for="field in fields" :key="field.key">
              {{ field.label }}
              <select v-if="field.type === 'select'" v-model="form[field.key]" required>
                <option value="">请选择</option>
                <option v-for="option in field.options" :key="option">{{ option }}</option>
              </select>
              <input v-else v-model="form[field.key]" :type="field.type || 'text'" required />
            </label>
            <label>
              备注
              <textarea v-model="note" placeholder="填写处理说明或现场备注" />
            </label>
            <button type="submit">{{ project.primaryAction }}</button>
          </div>
        </form>

        <section class="list-panel">
          <div class="toolbar">
            <h2>{{ project.entityLabel }}列表</h2>
            <div class="toolbar-tools">
              <label class="operator-box">
                操作人
                <input v-model="operator" @input="saveOperator" placeholder="值班员姓名" />
              </label>
              <select v-model="filter">
                <option v-for="item in project.filters" :key="item">{{ item }}</option>
              </select>
            </div>
          </div>

          <div class="record-grid">
            <div v-if="filteredRecords.length === 0" class="empty">暂无匹配数据</div>
            <article v-for="record in filteredRecords" :key="record.id" class="record">
              <div class="record-head">
                <p class="record-title">{{ primaryText(record) }}</p>
                <span class="status" :class="statusClass(record.status)">{{ record.status }}</span>
              </div>
              <div class="details">
                <span v-for="field in fields" :key="field.key">{{ field.label }}: {{ record[field.key] }}</span>
              </div>
              <p class="note">{{ record.notes }}</p>

              <div v-if="record.status === STATUS.RESTOCKING" class="restock-info">
                <span>预计到货：{{ record.restockEta ? formatTime(record.restockEta) : "—" }}</span>
                <span>补货负责人：{{ record.restockOwner || "—" }}</span>
                <span v-if="etaOverdue(record)" class="overdue">已超过预计到货时间</span>
              </div>

              <p v-if="record.status === STATUS.SUSPENDED" class="hint">暂停营业期间不能登记补货，请先恢复营业。</p>

              <form
                v-if="restockForms[record.id] && record.status === STATUS.LOW"
                class="restock-form"
                @submit.prevent="submitRestock(record)"
              >
                <label>
                  预计到货时间
                  <input v-model="restockForms[record.id].eta" type="datetime-local" required />
                </label>
                <label>
                  补货负责人
                  <input v-model="restockForms[record.id].owner" placeholder="跟进补货的负责人" required />
                </label>
                <div class="restock-actions">
                  <button type="submit">提交补货登记</button>
                  <button type="button" class="secondary" @click="cancelRestock(record)">取消</button>
                </div>
              </form>

              <div class="actions">
                <template v-for="btn in buttonsOf(record)" :key="btn.action">
                  <button
                    v-if="btn.action === 'restock'"
                    type="button"
                    :class="btn.cls"
                    @click="openRestock(record)"
                  >
                    {{ btn.label }}
                  </button>
                  <button
                    v-else
                    type="button"
                    :class="btn.cls"
                    @click="applyFlow(record, btn.action)"
                  >
                    {{ btn.label }}
                  </button>
                </template>
                <button class="secondary" type="button" @click="navigator.clipboard?.writeText(primaryText(record))">复制摘要</button>
                <button class="danger" type="button" @click="remove(record.id)">删除</button>
              </div>

              <div class="history">
                <p class="history-title">最近变更（{{ historyOf(record).length }}/5）</p>
                <p v-if="historyOf(record).length === 0" class="history-empty">暂无变更记录</p>
                <div v-for="item in historyOf(record)" :key="item.id" class="history-item">
                  <div class="history-line">
                    <span>{{ formatTime(item.at) }}</span>
                    <span class="history-flow">{{ item.fromStatus }} → {{ item.toStatus }}</span>
                    <span>操作人：{{ item.operator }}</span>
                  </div>
                  <p class="history-remark">{{ item.remark }}</p>
                </div>
              </div>
            </article>
          </div>

          <div class="mini-chart">
            <div v-for="row in chartRows" :key="row.status" class="bar">
              <span>{{ row.status }}</span>
              <div class="bar-track"><div class="bar-fill" :style="{ width: `${(row.value / maxChart) * 100}%` }" /></div>
              <strong>{{ row.value }}</strong>
            </div>
          </div>
        </section>
      </section>
    </div>
  </main>
</template>
