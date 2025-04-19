<template>
  <div class="card">
    <div class="card-header">
      Aktív Task - <small>TaskInfo Component</small>
    </div>
    <div class="card-body">
      <!-- Betöltési állapot -->
      <div v-if="isLoading" class="text-center">
        <div class="spinner-border" role="status">
          <span class="visually-hidden">Betöltés...</span>
        </div>
        <p>Betöltés...</p>
      </div>
      <!-- Hibaüzenet -->
      <div v-else-if="errorMessage" class="alert alert-danger">
        {{ errorMessage }}
      </div>
      <!-- Ha a task adatai betöltődtek -->
      <div v-else-if="task">
        <h5 class="card-title">Task #{{ task.id }}</h5>
        <p class="card-text"><strong>Leírás:</strong> {{ task.description }}</p>
        <ul class="list-group list-group-flush">
          <li class="list-group-item">
            <strong>Létrehozva:</strong> {{ formatDate(task.created_at) }}
          </li>
          <li class="list-group-item">
            <strong>Frissítve:</strong> {{ formatDate(task.updated_at) }}
          </li>
          <li class="list-group-item">
            <strong>Feladó:</strong> {{ task.assigner }}
          </li>
          <li class="list-group-item">
            <strong>Kiadás dátuma:</strong> {{ task.state0date ? formatDate(task.state0date) : "Nincs" }}
          </li>
          <li class="list-group-item">
            <strong>Megkezdés dátuma:</strong> {{ task.state1date ? formatDate(task.state1date) : "Nincs" }}
          </li>
          <li class="list-group-item">
            <strong>Befejezés dátuma:</strong> {{ task.state2date ? formatDate(task.state2date) : "Nincs" }}
          </li>
        </ul>
        <div class="mt-3 text-center">
          <button v-if="!actionLoading" class="btn btn-primary" @click="handleTaskAction">
            {{ task.state1date ? "Folytatás" : "Megkezdés" }}
          </button>
          <div v-else class="progress mt-3">
            <div class="progress-bar progress-bar-striped progress-bar-animated" style="width: 100%"></div>
          </div>
        </div>
      </div>
      <!-- Ha nincs aktív task -->
      <div v-else>
        <p>Nincs aktív task.</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';
import { defineProps, defineEmits } from 'vue';

const props = defineProps({
  token: { type: String, required: true }
});
const emit = defineEmits(['navigate']);

const task = ref(null);
const isLoading = ref(false);
const errorMessage = ref("");
const actionLoading = ref(false);

const formatDate = (dateStr) => {
  if (!dateStr) return "";
  const dateObj = new Date(dateStr);
  return dateObj.toLocaleString();
};

const fetchTask = async () => {
  isLoading.value = true;
  errorMessage.value = "";
  try {
    const response = await fetch("http://127.0.0.1:8000/api/task", {
      method: "GET",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${props.token}`
      }
    });
    if (!response.ok) {
      const data = await response.json();
      throw new Error(data.error || "Hiba történt a task lekérésekor!");
    }
    const data = await response.json();
    if (data.success) {
      task.value = data.task;
    } else {
      throw new Error(data.error || "Nem sikerült lekérni az aktív taskot.");
    }
  } catch (error) {
    errorMessage.value = error.message;
  } finally {
    isLoading.value = false;
  }
};

onMounted(() => {
  if (props.token) fetchTask();
});
watch(() => props.token, (newVal) => {
  if (newVal) fetchTask();
});

const handleTaskAction = async () => {
  if (!task.value) return;
  if (!task.value.state1date) {
    try {
      const response = await fetch(`http://127.0.0.1:8000/api/task/${task.value.id}`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${props.token}`
        },
        body: JSON.stringify({ action: "start" })
      });
      if (!response.ok) {
        const data = await response.json();
        throw new Error(data.error || "Hiba történt a feladat indításakor!");
      }
      const result = await response.json();
      if (result.task && result.task.state1date) {
        task.value.state1date = result.task.state1date;
      }
    } catch (error) {
      console.error("Error starting task:", error);
      errorMessage.value = error.message;
      return;
    }
  }
  emit('navigate', task.value.id);
};
</script>

<style scoped>
.card {
  margin-top: 20px;
}
</style>
