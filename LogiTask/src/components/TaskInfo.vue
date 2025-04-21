<template>
  <div class="card shadow-sm mt-4">
    <!-- Card header -->
    <div class="card-header bg-dark text-white">
      <h5 class="mb-0">Aktív Task</h5>
    </div>

    <!-- Card body -->
    <div class="card-body">
      <!-- Betöltés -->
      <div v-if="isLoading" class="text-center my-5">
        <div class="spinner-border" role="status">
          <span class="visually-hidden">Betöltés...</span>
        </div>
        <p class="mt-2">Betöltés...</p>
      </div>

      <!-- Hiba -->
      <div v-else-if="errorMessage" class="alert alert-danger">
        {{ errorMessage }}
      </div>

      <!-- Task részletek -->
      <div v-else-if="task">
        <!-- Leírás kiemelten -->
        <p class="task-description mb-4">{{ task.description }}</p>

        <ul class="list-group list-group-flush mb-4">
          <li class="list-group-item">
            <strong>Létrehozva:</strong> {{ formatDate(task.created_at) }}
          </li>
          <li class="list-group-item">
            <strong>Frissítve:</strong> {{ formatDate(task.updated_at) }}
          </li>
          <li class="list-group-item">
            <strong>Feladó:</strong> {{ task.assigner.name }}
          </li>
          <li class="list-group-item">
            <strong>Kiadás dátuma:</strong>
            {{ task.state0date ? formatDate(task.state0date) : "Nincs" }}
          </li>
          <li class="list-group-item">
            <strong>Megkezdés dátuma:</strong>
            {{ task.state1date ? formatDate(task.state1date) : "Nincs" }}
          </li>
          <li class="list-group-item">
            <strong>Befejezés dátuma:</strong>
            {{ task.state2date ? formatDate(task.state2date) : "Nincs" }}
          </li>
        </ul>

        <div class="text-center">
          <!-- Kezdés / Folytatás gomb -->
          <button
            v-if="!actionLoading"
            class="btn btn-outline-dark px-4"
            @click="handleTaskAction"
          >
            {{ task.state1date ? "Folytatás" : "Megkezdés" }}
          </button>
          <!-- Fekete alertben lévő spinner -->
          <div v-else class="alert alert-dark d-flex align-items-center justify-content-center">
            <div class="spinner-border text-white me-3" role="status">
              <span class="visually-hidden">Folyamat...</span>
            </div>
            <span class="text-white">Folyamat...</span>
          </div>
        </div>
      </div>

      <!-- Nincs task -->
      <div v-else class="text-center text-muted my-5">
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
  return new Date(dateStr).toLocaleString();
};

const fetchTask = async () => {
  isLoading.value = true;
  errorMessage.value = "";
  try {
    const res = await fetch("http://127.0.0.1:8000/api/task", {
      headers: { Authorization: `Bearer ${props.token}` }
    });
    const json = await res.json();
    console.log('fetchTask response:', json);
    if (!res.ok) {
      throw new Error(json.error || "Hiba a task lekérésekor");
    }
    if (json.success && json.task) {
      task.value = json.task;
    } else {
      task.value = null;
      errorMessage.value = json.error || "Nincs aktív task.";
    }
  } catch (e) {
    errorMessage.value = e.message;
  } finally {
    isLoading.value = false;
  }
};

onMounted(() => {
  if (props.token) fetchTask();
});
watch(() => props.token, (tok) => {
  if (tok) fetchTask();
});

const handleTaskAction = async () => {
  if (!task.value) return;
  actionLoading.value = true;
  try {
    const res = await fetch(`http://127.0.0.1:8000/api/task/${task.value.id}`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${props.token}`
      },
      body: JSON.stringify({ action: task.value.state1date ? "continue" : "start" })
    });
    const json = await res.json();
    console.log('handleTaskAction response:', json);
    if (!res.ok) {
      throw new Error(json.error || "Hiba a task indításakor");
    }
    if (json.task?.state1date) {
      task.value.state1date = json.task.state1date;
    }
    emit('navigate', task.value.id);
  } catch (e) {
    console.error(e);
    errorMessage.value = e.message;
  } finally {
    actionLoading.value = false;
  }
};
</script>

<style scoped>
.card {
  margin-top: 20px;
}
/* A task leírását nagyobbra, kiemeltebbre vesszük */
.task-description {
  font-size: 1.5rem;
  font-weight: 600;
  color: #343a40;
}
/* Fekete körvonalú gomb */
.btn-outline-dark {
  border-width: 2px;
}
</style>
