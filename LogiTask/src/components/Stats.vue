<template>
    <div class="card">
      <div class="card-header">
        Statisztika - <small>Stats Component</small>
      </div>
      <div class="card-body">
        <div class="row">
          <!-- Bal oldali menüsáv -->
          <div class="col-md-3">
            <div class="list-group">
              <button
                v-for="(option, index) in options"
                :key="index"
                class="list-group-item list-group-item-action"
                :class="{ active: selectedOptionIndex === index }"
                @click="selectedOptionIndex = index"
              >
                {{ option.label }}
              </button>
            </div>
          </div>
          <!-- Jobb oldali tartalom -->
          <div class="col-md-9">
            <div class="mb-3">
              <label class="form-label">Start Date</label>
              <input v-model="startDate" type="date" class="form-control" />
            </div>
            <div class="mb-3">
              <label class="form-label">End Date</label>
              <input v-model="endDate" type="date" class="form-control" />
            </div>
            <!-- Extra input, ha az aktuális opció igényli a task_type értéket -->
            <div class="mb-3" v-if="currentOption.requiresTaskType">
              <label class="form-label">Task Type</label>
              <input
                v-model="taskType"
                type="number"
                class="form-control"
                placeholder="Enter task type"
              />
            </div>
            <button class="btn btn-primary" @click="fetchStats">
              Lekérés
            </button>
            <div class="mt-3" v-if="statsData">
              <div v-if="statsData.length === 0">
                <p>Nincs elérhető adat.</p>
              </div>
              <div v-else>
                <canvas id="statsChart"></canvas>
              </div>
            </div>
          </div>
        </div>
        <div class="mt-3 text-center">
          <button v-if="!backLoading" class="btn btn-secondary" @click="backToMain">
            Vissza a főoldalra
          </button>
          <div v-else class="progress mt-3">
            <div class="progress-bar progress-bar-striped progress-bar-animated" style="width:100%"></div>
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, computed, onMounted, watch, nextTick } from 'vue';
  import { defineProps, defineEmits } from 'vue';
  import Chart from 'chart.js/auto';
  
  const props = defineProps({
    token: { type: String, required: true }
  });
  const emit = defineEmits(['backToTaskInfo']);
  
  // Statikus opciók (8 endpoint)
  const options = [
    { label: 'Worker Completed Task', endpoint: '/statistics/worker/compleated-task', requiresTaskType: false },
    { label: 'Worker Avg Work Time', endpoint: '/statistics/worker/avg-work-time', requiresTaskType: false },
    { label: 'Task Avg Task Time', endpoint: '/statistics/task/avg-task-time', requiresTaskType: false },
    { label: 'Task Avg Task Time By Type', endpoint: '/statistics/task/avg-task-time-by-type', requiresTaskType: true },
    { label: 'Task Count By Role', endpoint: '/statistics/task/task-count-by-role', requiresTaskType: false },
    { label: 'Most Active API Users', endpoint: '/statistics/api/most-active-api-users', requiresTaskType: false },
    { label: 'API Usage', endpoint: '/statistics/api/usage', requiresTaskType: false },
    { label: 'Most Requested Route', endpoint: '/statistics/api/requests', requiresTaskType: false }
  ];
  
  const selectedOptionIndex = ref(0);
  const currentOption = computed(() => options[selectedOptionIndex.value]);
  
  // Input mezők
  const startDate = ref("1900-01-01");
  const endDate = ref("2100-01-01");
  const taskType = ref("");
  
  // Állapotok
  const statsData = ref(null);
  const isLoading = ref(false);
  const backLoading = ref(false);
  
  // Mapping objektum módosítva a backend kulcsokra
  const mapping = {
    '/statistics/worker/compleated-task': { 
      labelKey: (item) => item.worker || item.name || "", 
      valueKey: (item) => item.task_count || 0 
    },
    '/statistics/worker/avg-work-time': { 
      labelKey: (item) => item.worker || item.name || "", 
      valueKey: (item) => item.total_work_time || 0 
    },
    '/statistics/task/avg-task-time': { 
      labelKey: (item) => item.task || item.name || "", 
      valueKey: (item) => item.avg_completion_time || 0 
    },
    '/statistics/task/avg-task-time-by-type': { 
      labelKey: (item) => "Task Type " + taskType.value, 
      valueKey: (item) => parseFloat(item.avg_task_completion_time) || 0 
    },
    '/statistics/task/task-count-by-role': { 
      labelKey: (item) => item.role || "", 
      valueKey: (item) => item.task_count || item.count || 0 
    },
    '/statistics/api/most-active-api-users': { 
      labelKey: (item) => item.user || item.name || "", 
      valueKey: (item) => item.api_usage_count || 0 
    },
    '/statistics/api/usage': { 
      labelKey: (item) => item.usage || item.name || "", 
      valueKey: (item) => item.count || 0 
    },
    '/statistics/api/requests': { 
      labelKey: (item) => item.route || item.name || "", 
      valueKey: (item) => item.request_count || 0 
    },
  };
  
  async function fetchStats() {
    isLoading.value = true;
    statsData.value = null;
    try {
      const baseUrl = "http://127.0.0.1:8000/api" + currentOption.value.endpoint;
      const url = new URL(baseUrl);
      url.searchParams.append("startDate", startDate.value);
      url.searchParams.append("endDate", endDate.value);
      if (currentOption.value.requiresTaskType) {
        url.searchParams.append("task_type", taskType.value);
      }
      console.log("Fetching stats from:", url.toString());
      const response = await fetch(url.toString(), {
        method: "GET",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${props.token}`
        }
      });
      const data = await response.json();
      console.log("Stats response:", data);
      if (!response.ok) {
        throw new Error(data.error || "Statisztika lekérés hiba");
      }
      statsData.value = data.data;
      await nextTick();
      if (statsData.value && statsData.value.length > 0) {
        drawChart();
      } else {
        console.log("statsData is empty");
        // Ha nincs adat, töröljük az esetleges előző diagramot is:
        if (myChart) {
          myChart.destroy();
          myChart = null;
        }
      }
    } catch (error) {
      console.error("Error fetching stats:", error);
      statsData.value = [];
      if (myChart) {
        myChart.destroy();
        myChart = null;
      }
    } finally {
      isLoading.value = false;
    }
  }
  
  let myChart = null;
  function drawChart() {
    const canvas = document.getElementById('statsChart');
    if (!canvas) {
      console.error("Canvas element not found!");
      return;
    }
    const ctx = canvas.getContext('2d');
    if (myChart) {
      myChart.destroy();
    }
    const mapFn = mapping[currentOption.value.endpoint] || { 
      labelKey: (item) => item.label || "", 
      valueKey: (item) => item.value || 0 
    };
    const labels = statsData.value.map(item => mapFn.labelKey(item));
    const values = statsData.value.map(item => mapFn.valueKey(item));
    console.log("Labels:", labels, "Values:", values);
    myChart = new Chart(ctx, {
      type: 'bar',
      data: {
        labels: labels,
        datasets: [{
          label: currentOption.value.label,
          data: values,
          backgroundColor: 'rgba(54, 162, 235, 0.6)',
          borderColor: 'rgba(54, 162, 235, 1)',
          borderWidth: 1
        }]
      },
      options: {
        scales: {
          y: { beginAtZero: true }
        }
      }
    });
  }
  
  // Amikor a menü opció változik, töröljük az előző diagramot és az adatokat
  watch(selectedOptionIndex, () => {
    if (myChart) {
      myChart.destroy();
      myChart = null;
    }
    statsData.value = null;
  });
  
  function backToMain() {
    backLoading.value = true;
    setTimeout(() => {
      backLoading.value = false;
      emit('backToTaskInfo');
    }, 1000);
  }
  </script>
  
  <style scoped>
  .card {
    margin-top: 20px;
  }
  .list-group-item {
    cursor: pointer;
  }
  .list-group-item.active {
    background-color: #0d6efd;
    border-color: #0d6efd;
    color: white;
  }
  </style>
  