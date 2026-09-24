<script setup lang="ts">
import { computed, reactive, ref, watch } from "vue";

type Field = {
  key: string;
  label: string;
  type?: "number" | "date" | "select";
  options?: readonly string[];
};

type RecordItem = {
  id: string;
  status: string;
  notes: string;
  createdAt: string;
  expectedArrival?: string;
  restockManager?: string;
  [key: string]: string | number | undefined;
};

type Transition = {
  key: string;
  from: string;
  to: string;
  label: string;
  needsForm?: boolean;
};

type FlowLog = {
  id: string;
  recordId: string;
  station: string;
  action: string;
  from: string;
  to: string;
  operator: string;
  time: string;
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
  "formTitle": "新增油站",
  "primaryAction": "保存油站",
  "entityLabel": "油站",
  "statuses": [
    "营业中",
    "库存不足",
    "补货中",
    "暂停营业"
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
      "status": "库存不足",
      "notes": "柴油待补"
    }
  ],
  "metricLabels": [
    "油站数",
    "营业中",
    "库存不足"
  ]
} as const;

const fields = project.fields as readonly Field[];
const statuses = [...project.statuses];

// 状态机：只允许以下流转。
// 补货必须登记预计到货时间和负责人；补货中须经到货确认才能恢复营业；暂停营业不能开始补货。
const transitions: readonly Transition[] = [
  { key: "low", from: "营业中", to: "库存不足", label: "登记库存不足" },
  { key: "suspend", from: "营业中", to: "暂停营业", label: "暂停营业" },
  { key: "restock", from: "库存不足", to: "补货中", label: "登记补货", needsForm: true },
  { key: "arrive", from: "补货中", to: "营业中", label: "到货确认" },
  { key: "resume", from: "暂停营业", to: "营业中", label: "恢复营业" }
];

const logStorageKey = `${project.storageKey}-flow-logs`;
const operatorStorageKey = `${project.storageKey}-operator`;

// 历史数据里的旧状态名映射到现行状态
const legacyStatus: Record<string, string> = { 库存紧张: "库存不足" };

function normalizeStatus(status: string) {
  if (statuses.includes(status)) return status;
  return legacyStatus[status] ?? statuses[0];
}

function createBlank() {
  return Object.fromEntries(fields.map((field) => [field.key, field.type === "number" ? 0 : ""]));
}

function loadRecords(): RecordItem[] {
  const raw = localStorage.getItem(project.storageKey);
  if (!raw) {
    return project.records.map((record, index) => ({
      ...record,
      id: `seed-${index + 1}`,
      createdAt: new Date(Date.now() - index * 86400000).toISOString()
    })) as RecordItem[];
  }
  try {
    const stored = JSON.parse(raw) as RecordItem[];
    return stored.map((record) => ({ ...record, status: normalizeStatus(record.status) }));
  } catch {
    return [];
  }
}

function loadLogs(): FlowLog[] {
  try {
    return JSON.parse(localStorage.getItem(logStorageKey) ?? "[]") as FlowLog[];
  } catch {
    return [];
  }
}

const records = ref<RecordItem[]>(loadRecords());
const logs = ref<FlowLog[]>(loadLogs());
const form = reactive<Record<string, string | number>>(createBlank());
const note = ref("");
const filter = ref(project.filters[0]);
const operator = ref(localStorage.getItem(operatorStorageKey) || "值班员");
const restockTarget = ref<RecordItem | null>(null);
const restockForm = reactive({ expectedArrival: "", manager: "" });

watch(operator, (value) => localStorage.setItem(operatorStorageKey, value));

const filteredRecords = computed(() => {
  if (filter.value.startsWith("全部")) return records.value;
  return records.value.filter((record) => Object.values(record).includes(filter.value));
});

const metrics = computed(() => [
  records.value.length,
  records.value.filter((record) => record.status === "营业中").length,
  records.value.filter((record) => record.status === "库存不足").length
]);

const chartRows = computed(() => statuses.map((status) => ({
  status,
  value: records.value.filter((record) => record.status === status).length
})));

const maxChart = computed(() => Math.max(1, ...chartRows.value.map((row) => row.value)));

const recentLogs = computed(() => logs.value.slice(0, 5));

function persist() {
  localStorage.setItem(project.storageKey, JSON.stringify(records.value));
}

function persistLogs() {
  localStorage.setItem(logStorageKey, JSON.stringify(logs.value));
}

function primaryText(record: RecordItem) {
  const first = fields[0];
  const second = fields[1];
  return [record[first.key], record[second.key]].filter(Boolean).join(" / ") || project.entityLabel;
}

function formatTime(value?: string | number) {
  if (!value) return "";
  const date = new Date(value);
  return Number.isNaN(date.getTime()) ? String(value) : date.toLocaleString("zh-CN", { hour12: false });
}

function statusClass(status: string) {
  return {
    营业中: "is-open",
    库存不足: "is-low",
    补货中: "is-restocking",
    暂停营业: "is-closed"
  }[status] ?? "";
}

function availableActions(record: RecordItem) {
  return transitions.filter((transition) => transition.from === record.status);
}

function addLog(record: RecordItem, transition: Transition) {
  const now = Date.now();
  const latest = logs.value[0];
  // 幂等：同一处理的重复点击（同油站、同操作、同前后状态且间隔极短）只保留一条
  if (
    latest &&
    latest.recordId === record.id &&
    latest.action === transition.label &&
    latest.from === transition.from &&
    latest.to === transition.to &&
    now - new Date(latest.time).getTime() < 2000
  ) {
    return;
  }
  logs.value = [
    {
      id: crypto.randomUUID(),
      recordId: record.id,
      station: String(record[fields[0].key] || project.entityLabel),
      action: transition.label,
      from: transition.from,
      to: transition.to,
      operator: operator.value.trim() || "值班员",
      time: new Date(now).toISOString()
    },
    ...logs.value
  ].slice(0, 100);
  persistLogs();
}

function applyTransition(record: RecordItem, transition: Transition, extra?: () => void) {
  // 幂等守卫：状态已变化（如重复点击、重复提交）时直接忽略，不重复变更也不重复记录
  if (record.status !== transition.from) return;
  extra?.();
  record.status = transition.to;
  addLog(record, transition);
  persist();
}

function runAction(record: RecordItem, transition: Transition) {
  if (transition.needsForm) {
    openRestock(record);
    return;
  }
  applyTransition(record, transition);
}

function openRestock(record: RecordItem) {
  // 仅库存不足可登记补货，暂停营业不能开始补货
  if (record.status !== "库存不足") return;
  restockTarget.value = record;
  restockForm.expectedArrival = "";
  restockForm.manager = String(record.manager || operator.value);
}

function cancelRestock() {
  restockTarget.value = null;
}

function confirmRestock() {
  const record = restockTarget.value;
  if (!record || !restockForm.expectedArrival || !restockForm.manager.trim()) return;
  const transition = transitions.find((item) => item.key === "restock");
  if (!transition) return;
  applyTransition(record, transition, () => {
    record.expectedArrival = restockForm.expectedArrival;
    record.restockManager = restockForm.manager.trim();
  });
  restockTarget.value = null;
}

function submit() {
  records.value = [
    {
      ...form,
      id: crypto.randomUUID(),
      status: statuses[0],
      notes: note.value || "暂无备注",
      createdAt: new Date().toISOString()
    } as RecordItem,
    ...records.value
  ];
  Object.assign(form, createBlank());
  note.value = "";
  persist();
}

function copySummary(record: RecordItem) {
  navigator.clipboard?.writeText(primaryText(record));
}

function remove(id: string) {
  records.value = records.value.filter((record) => record.id !== id);
  persist();
}
</script>

<template>
  <main class="app">
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
            <label class="operator-field">
              操作人
              <input v-model="operator" type="text" placeholder="值班员姓名" />
            </label>
            <select v-model="filter">
              <option v-for="item in project.filters" :key="item">{{ item }}</option>
            </select>
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
              <p v-if="record.status === '补货中' && record.expectedArrival" class="restock-info">
                预计到货：{{ formatTime(record.expectedArrival) }} ・ 补货负责人：{{ record.restockManager || "未填写" }}
              </p>
              <div class="actions">
                <button
                  v-for="action in availableActions(record)"
                  :key="action.key"
                  type="button"
                  @click="runAction(record, action)"
                >
                  {{ action.label }}
                </button>
                <button class="secondary" type="button" @click="copySummary(record)">复制摘要</button>
                <button class="danger" type="button" @click="remove(record.id)">删除</button>
              </div>
              <div v-if="restockTarget?.id === record.id" class="restock-form">
                <label>
                  预计到货时间
                  <input v-model="restockForm.expectedArrival" type="datetime-local" />
                </label>
                <label>
                  负责人
                  <input v-model="restockForm.manager" type="text" placeholder="补货负责人" />
                </label>
                <div class="actions">
                  <button
                    type="button"
                    :disabled="!restockForm.expectedArrival || !restockForm.manager.trim()"
                    @click="confirmRestock"
                  >
                    确认登记
                  </button>
                  <button class="secondary" type="button" @click="cancelRestock">取消</button>
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

          <div class="flow-log">
            <h3>变更记录（最近5条）</h3>
            <p v-if="recentLogs.length === 0" class="empty">暂无变更记录</p>
            <ul v-else>
              <li v-for="log in recentLogs" :key="log.id">
                <span class="log-time">{{ formatTime(log.time) }}</span>
                <span class="log-station">{{ log.station }}</span>
                <span>{{ log.action }}</span>
                <span class="log-status">{{ log.from }} → {{ log.to }}</span>
                <span>操作人：{{ log.operator }}</span>
              </li>
            </ul>
          </div>
        </section>
      </section>
    </div>
  </main>
</template>
