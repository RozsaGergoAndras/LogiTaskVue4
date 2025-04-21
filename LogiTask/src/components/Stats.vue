<template>
  <div class="card shadow-sm mb-4 h-100">
    <!-- Card header in dark style -->
    <div class="card-header bg-dark text-white">
      <h5 class="mb-0">Statisztika</h5>
    </div>

    <!-- Card body with nav + content -->
    <div class="card-body d-flex flex-column">
      <div class="row flex-fill">
        <!-- MENU COLUMN -->
        <div class="col-md-3 border-end pe-3">
          <h6 class="section-title">Feladat statisztika</h6>
          <ul class="nav nav-pills flex-column mb-4">
            <li class="nav-item" v-for="(opt, idx) in feladatOptions" :key="opt.endpoint">
              <button
                class="nav-link text-start"
                :class="{ active: selectedIndex===idx }"
                @click="selectOption(idx)"
              >
                {{ opt.label }}
              </button>
            </li>
          </ul>
          <h6 class="section-title">Rendszer statisztika</h6>
          <ul class="nav nav-pills flex-column">
            <li class="nav-item" v-for="(opt, idx) in systemOptions" :key="opt.endpoint">
              <button
                class="nav-link text-start"
                :class="{ active: selectedIndex===idx+feladatOptions.length }"
                @click="selectOption(idx+feladatOptions.length)"
              >
                {{ opt.label }}
              </button>
            </li>
          </ul>
        </div>

        <!-- CONTENT COLUMN -->
        <div class="col-md-9 d-flex flex-column">
          <!-- Filters -->
          <div class="row g-3 mb-4 flex-shrink-0">
            <div class="col-md-4">
              <label class="form-label"><strong>Kezdő dátum</strong></label>
              <input v-model="startDate" type="date" class="form-control" />
            </div>
            <div class="col-md-4">
              <label class="form-label"><strong>Vég dátum</strong></label>
              <input v-model="endDate" type="date" class="form-control" />
            </div>
            <div class="col-md-4" v-if="current.requiresTaskType">
              <label class="form-label"><strong>Feladattípus</strong></label>
              <select v-model="taskTypeId" class="form-select">
                <option disabled value="">– Válassz típus –</option>
                <option v-for="t in taskTypes" :key="t.id" :value="t.id">
                  {{ t.type_name }}
                </option>
              </select>
            </div>
          </div>

          <!-- Fetch button -->
          <div class="d-grid mb-4 flex-shrink-0">
            <button
              class="btn btn-outline-dark btn-lg"
              @click="fetchStats"
              :disabled="loading"
            >
              {{ loading ? 'Lekérés...' : 'Lekérés' }}
            </button>
          </div>

          <!-- Error -->
          <div v-if="error" class="alert alert-danger mb-4 flex-shrink-0">
            {{ error }}
          </div>

          <!-- Results -->
          <div class="flex-fill overflow-auto">
            <div v-if="statsData===null"></div>
            <div v-else-if="statsData.length===0" class="text-center text-muted">
              Nincs elérhető adat.
            </div>

            <!-- Single‑value display -->
            <div
              v-else-if="isSingleValueEndpoint"
              class="d-flex flex-column align-items-center justify-content-center p-5 bg-light rounded shadow-sm"
            >
              <div class="display-1 text-dark">
                {{ displayText }}
              </div>
              <div class="h5 mt-2">{{ current.label }}</div>
            </div>

            <!-- Chart -->
            <div v-else class="chart-container p-4 bg-light rounded shadow-sm">
              <canvas ref="statsCanvas"></canvas>
            </div>
          </div>
        </div>
      </div>

      <!-- Back button -->
      <div class="mt-4 text-center flex-shrink-0">
        <button class="btn btn-secondary" @click="goBack" :disabled="backLoading">
          {{ backLoading ? 'Vissza...' : 'Vissza a főoldalra' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, onMounted, watch } from 'vue';
import Chart from 'chart.js/auto';
import { defineEmits } from 'vue';

const emit = defineEmits(['backToTaskInfo']);

const allOptions = [
  { label: 'Feladatok teljesítve',             endpoint: '/statistics/worker/compleated-task',     requiresTaskType: false },
  { label: 'Dolgozói átlagmunkaidő',            endpoint: '/statistics/worker/avg-work-time',       requiresTaskType: false },
  { label: 'Feladat átlagos ideje',             endpoint: '/statistics/task/avg-task-time',         requiresTaskType: false },
  { label: 'Feladat átlagideje típus szerint',  endpoint: '/statistics/task/avg-task-time-by-type', requiresTaskType: true  },
  { label: 'Feladatok száma szerepek szerint',  endpoint: '/statistics/task/task-count-by-role',    requiresTaskType: false },
  { label: 'Legaktívabb API felhasználók',      endpoint: '/statistics/api/most-active-api-users',  requiresTaskType: false },
  { label: 'API használat',                     endpoint: '/statistics/api/usage',                  requiresTaskType: false },
  { label: 'Legkért útvonal',                   endpoint: '/statistics/api/requests',               requiresTaskType: false }
];

const feladatOptions = computed(() => allOptions.slice(0,5));
const systemOptions  = computed(() => allOptions.slice(5));

const token        = ref(localStorage.getItem('token')||'');
const selectedIndex= ref(0);

const startDate    = ref('1900-01-01');
const endDate      = ref('2100-01-01');
const taskTypeId   = ref('');
const taskTypes    = ref([]);

const statsData    = ref(null);
const loading      = ref(false);
const error        = ref('');
const backLoading  = ref(false);

const statsCanvas  = ref(null);
let chartInstance  = null;

const current = computed(() => allOptions[selectedIndex.value]);

const unitEndpoints = [
  '/statistics/worker/avg-work-time',
  '/statistics/task/avg-task-time',
  '/statistics/task/avg-task-time-by-type'
];

const isSingleValueEndpoint = computed(() =>
  ['/statistics/task/avg-task-time-by-type','/statistics/api/usage']
    .includes(current.value.endpoint)
);

// raw single number
const singleValue = computed(() => {
  if (!statsData.value?.length) return NaN;
  const map = mapping[current.value.endpoint];
  return map ? map.valueKey(statsData.value[0]) : NaN;
});

// display text: number+unit or fallback
const displayText = computed(() => {
  const v = singleValue.value;
  if (isSingleValueEndpoint.value && (isNaN(v) || v === null)) {
    return 'Nincs lekérhető érték';
  }
  return `${v}${ unitEndpoints.includes(current.value.endpoint) ? ' perc' : '' }`;
});

const mapping = {
  '/statistics/worker/compleated-task':     { labelKey:i=>i.name,              valueKey:i=>i.task_count },
  '/statistics/worker/avg-work-time':       { labelKey:i=>i.name,              valueKey:i=>Math.round(i.total_work_time/60) },
  '/statistics/task/avg-task-time':         { labelKey:i=>i.name,              valueKey:i=>Math.round(parseFloat(i.avg_completion_time)/60) },
  '/statistics/task/avg-task-time-by-type': { labelKey:()=>`Típus ${taskTypeId.value}`, valueKey:i=>Math.round(parseFloat(i.avg_task_completion_time)/60) },
  '/statistics/task/task-count-by-role':    { labelKey:i=>i.role_name,         valueKey:i=>i.task_count },
  '/statistics/api/most-active-api-users':  { labelKey:i=>i.name,              valueKey:i=>i.api_usage_count },
  '/statistics/api/usage':                  { labelKey:()=> 'Összes hívás',     valueKey:i=>i.count },
  '/statistics/api/requests':               { labelKey:i=>i.route,             valueKey:i=>i.request_count }
};

async function fetchTaskTypes() {
  const res = await fetch('http://127.0.0.1:8000/api/tasktype', {
    headers: { 'Authorization': `Bearer ${token.value}` }
  });
  const j   = await res.json();
  taskTypes.value = j.data||[];
}

async function selectOption(idx) {
  selectedIndex.value = idx;
  statsData.value     = null;
  error.value         = '';
  destroyChart();
  if (current.value.requiresTaskType && !taskTypes.value.length) {
    await fetchTaskTypes();
  }
  fetchStats();
}

async function fetchStats() {
  loading.value = true; error.value=''; statsData.value=null;
  try {
    const url = new URL(`http://127.0.0.1:8000/api${current.value.endpoint}`);
    url.searchParams.set('startDate', startDate.value);
    url.searchParams.set('endDate',   endDate.value);
    if (current.value.requiresTaskType) url.searchParams.set('task_type', taskTypeId.value);

    const res = await fetch(url, { headers:{ Authorization:`Bearer ${token.value}` } });
    const j   = await res.json();
    if (!res.ok||!j.success) throw new Error(j.error||'Hiba a lekéréskor');
    statsData.value = j.data;
    await nextTick();
    if (!isSingleValueEndpoint.value) drawChart();
  } catch(e) {
    error.value = e.message;
  } finally {
    loading.value = false;
  }
}

function drawChart() {
  const ctx = statsCanvas.value?.getContext('2d');
  if (!ctx) return;
  destroyChart();
  const { labelKey,valueKey } = mapping[current.value.endpoint];
  const labels = statsData.value.map(labelKey);
  const data   = statsData.value.map(valueKey);

  chartInstance = new Chart(ctx,{
    type:'bar',
    data:{ labels, datasets:[{
      label: current.value.label,
      data,
      backgroundColor:'rgba(13,110,253,0.6)',
      borderColor:'rgba(13,110,253,1)',
      borderWidth:1
    }]},
    options:{
      scales:{ y:{ beginAtZero:true } },
      plugins:{ legend:{ display:false }}
    }
  });
}

function destroyChart(){
  if(chartInstance){ chartInstance.destroy(); chartInstance=null; }
}

function goBack(){
  backLoading.value=true;
  setTimeout(()=>{
    backLoading.value=false;
    emit('backToTaskInfo');
  },500);
}

onMounted(()=> selectOption(0));
watch(()=>selectedIndex.value,destroyChart);
</script>

<style scoped>
.section-title {
  font-size:1.1rem;
  font-weight:600;
  color:#343a40;
  border-bottom:2px solid #343a40;
  padding-bottom:.25rem;
  margin-bottom:.75rem;
}
.nav-pills .nav-link {
  border:none;
  color:#6c757d;
  margin-bottom:.25rem;
}
.nav-pills .nav-link.active {
  background-color:#343a40;
  color:#fff;
  border:1px solid #343a40;
}
.chart-container {
  width:100%;
  max-width:700px;
  margin:auto;
}
</style>
