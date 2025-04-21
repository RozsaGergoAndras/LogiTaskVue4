<template>
  <div class="card shadow-sm mt-4">
    <!-- Card header -->
    <div class="card-header bg-dark text-white">
      <h5 class="mb-0">Feladat kezelése</h5>
    </div>

    <!-- Card body -->
    <div class="card-body">
      <!-- Saját alert -->
      <div
        v-if="alert.show"
        :class="['alert', `alert-${alert.type}`, 'alert-dismissible', 'fade', 'show']"
        role="alert"
      >
        {{ alert.message }}
        <button
          type="button"
          class="btn-close"
          @click="alert.show = false"
        ></button>
      </div>

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
        <p class="task-description">{{ task.description }}</p>

        <ul class="list-group list-group-flush mb-4">
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

        <!-- Fájl feltöltés -->
        <div>
          <h5 class="file-section-title">Fájl feltöltése (opcionális)</h5>
          <form @submit.prevent="uploadFile" class="d-flex gap-2 mb-3">
            <input
              type="file"
              @change="handleFileChange"
              class="form-control"
            />
            <button type="submit" class="btn btn-outline-dark">
              Feltöltés
            </button>
          </form>
        </div>

        <!-- Fájlok listája -->
        <div>
          <h5 class="file-section-title">Fájlok</h5>
          <div v-if="files.length === 0" class="text-muted mb-3">
            Nincs feltöltött fájl.
          </div>
          <ul v-else class="list-group list-group-flush mb-4">
            <li
              v-for="(file, index) in files"
              :key="index"
              class="list-group-item d-flex justify-content-between align-items-center"
            >
              <div>
                <strong>{{ file.link }}</strong>
                <div><small>{{ formatDate(file.created_at) }}</small></div>
              </div>
              <button
                class="btn btn-sm btn-outline-dark"
                @click="openFile(file.link)"
              >
                Megnyitás
              </button>
            </li>
          </ul>
        </div>

        <!-- Feladat befejezése -->
        <div class="text-center">
          <button
            class="btn btn-outline-dark px-4"
            @click="showFinishModal = true"
          >
            Feladat befejezése
          </button>
        </div>
      </div>

      <!-- Nincs task -->
      <div v-else class="text-center text-muted my-5">
        <p>Nincs ilyen task.</p>
      </div>
    </div>

    <!-- Upload megerősítő modal -->
    <div
      v-if="showUploadConfirm"
      class="modal fade show"
      style="display: block;"
      tabindex="-1"
    >
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header bg-dark text-white">
            <h5 class="modal-title">Megerősítés</h5>
            <button
              type="button"
              class="btn-close btn-close-white"
              @click="showUploadConfirm = false"
            ></button>
          </div>
          <div class="modal-body">
            <p>Biztosan feltöltöd a kiválasztott fájlt?</p>
          </div>
          <div class="modal-footer">
            <button class="btn btn-outline-dark" @click="showUploadConfirm = false">
              Mégse
            </button>
            <button class="btn btn-dark" @click="doUpload">
              Igen, feltöltöm
            </button>
          </div>
        </div>
      </div>
    </div>
    <div v-if="showUploadConfirm" class="modal-backdrop fade show"></div>

    <!-- Befejezés megerősítése modal -->
    <div
      v-if="showFinishModal"
      class="modal fade show"
      style="display: block;"
      tabindex="-1"
    >
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header bg-dark text-white">
            <h5 class="modal-title">Megerősítés</h5>
            <button
              type="button"
              class="btn-close btn-close-white"
              @click="showFinishModal = false"
            ></button>
          </div>
          <div class="modal-body">
            <p>Biztosan be akarod fejezni a feladatot?</p>
          </div>
          <div class="modal-footer">
            <button class="btn btn-outline-dark" @click="showFinishModal = false">
              Mégse
            </button>
            <button class="btn btn-dark" @click="confirmFinish">
              Igen, befejezem
            </button>
          </div>
        </div>
      </div>
    </div>
    <div v-if="showFinishModal" class="modal-backdrop fade show"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';
import { defineProps, defineEmits } from 'vue';

const props = defineProps({
  token: { type: String, required: true },
  taskId: { type: [Number, String], required: true }
});
const emit = defineEmits(['navigateToEndTask']);

const task = ref(null);
const files = ref([]);
const isLoading = ref(false);
const errorMessage = ref("");
const fileToUpload = ref(null);

// saját alert state
const alert = ref({ show: false, type: 'warning', message: '' });

const showUploadConfirm = ref(false);
const showFinishModal = ref(false);

const formatDate = s => new Date(s).toLocaleString();

async function fetchTask() {
  isLoading.value = true;
  errorMessage.value = "";
  try {
    const res = await fetch(`http://127.0.0.1:8000/api/task/${props.taskId}`, {
      headers: { Authorization: `Bearer ${props.token}` }
    });
    const json = await res.json();
    if (!res.ok) throw new Error(json.error || "Hiba a task lekéréskor");
    task.value = json.task || null;
    files.value = Array.isArray(json.taskContents)
      ? json.taskContents
      : Object.values(json.taskContents || {});
  } catch (e) {
    errorMessage.value = e.message;
  } finally {
    isLoading.value = false;
  }
}

function handleFileChange(e) {
  fileToUpload.value = e.target.files[0] || null;
}

function uploadFile() {
  if (!fileToUpload.value) {
    alert.value = { show: true, type: 'warning', message: 'Kérlek válassz egy fájlt!' };
    return;
  }
  showUploadConfirm.value = true;
}

async function doUpload() {
  showUploadConfirm.value = false;
  const formData = new FormData();
  formData.append("file", fileToUpload.value);
  formData.append("taskId", props.taskId);
  try {
    const res = await fetch("http://127.0.0.1:8000/api/file/upload", {
      method: "POST",
      headers: { Authorization: `Bearer ${props.token}` },
      body: formData
    });
    const json = await res.json();
    if (!res.ok) throw new Error(json.error || "Feltöltési hiba");
    alert.value = { show: true, type: 'success', message: 'Fájl sikeresen feltöltve!' };
    await fetchTask();
  } catch (e) {
    alert.value = { show: true, type: 'danger', message: 'Hiba: ' + e.message };
  }
}

async function openFile(link) {
  try {
    const res = await fetch(
      `http://127.0.0.1:8000/api/file/signed-url/${link}`,
      { headers: { Authorization: `Bearer ${props.token}` } }
    );
    const json = await res.json();
    window.open(json.url, "_blank");
  } catch (e) {
    alert.value = { show: true, type: 'danger', message: 'Hiba a fájl megnyitásakor: ' + e.message };
  }
}

async function confirmFinish() {
  showFinishModal.value = false;
  try {
    const res = await fetch(`http://127.0.0.1:8000/api/task/${props.taskId}`, {
      method: "POST",
      headers: { Authorization: `Bearer ${props.token}` }
    });
    const json = await res.json();
    if (!res.ok) throw new Error(json.error || "Befejezési hiba");
    await fetchTask();
    let timeTaken = 0;
    if (task.value.state1date && task.value.state2date) {
      timeTaken = Math.floor( (new Date(task.value.state2date) - new Date(task.value.state1date)) / 1000 );
    }
    emit("navigateToEndTask", { taskId: props.taskId, timeTaken });
  } catch (e) {
    alert.value = { show: true, type: 'danger', message: 'Hiba: ' + e.message };
  }
}

onMounted(fetchTask);
watch(() => props.taskId, fetchTask);
</script>

<style scoped>
.card-header {
  background-color: #343a40;
  color: #fff;
}
.task-description {
  font-size: 1.25rem;
  font-weight: 500;
  margin-bottom: 1.5rem;
}
.file-section-title {
  font-size: 1.2rem;
  font-weight: 600;
  margin-top: 1rem;
  margin-bottom: 0.5rem;
}
.btn-outline-dark {
  border-width: 2px;
}
.modal {
  background: rgba(0, 0, 0, 0.5);
}
</style>
