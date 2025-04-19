<template>
  <div class="card">
    <div class="card-header">
      Current Task - <small>CurrentTask Component</small>
    </div>
    <div class="card-body">
      <!-- Task adatok -->
      <div v-if="isLoading" class="text-center">
        <div class="spinner-border" role="status">
          <span class="visually-hidden">Betöltés...</span>
        </div>
        <p>Betöltés...</p>
      </div>
      <div v-else-if="errorMessage" class="alert alert-danger">{{ errorMessage }}</div>
      <div v-else-if="task">
        <h5 class="card-title">Task #{{ task.id }}</h5>
        <ul class="list-group list-group-flush">
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

        <!-- Fájl feltöltési rész (opcionális) -->
        <div class="mt-4">
          <h5>Fájl feltöltése (opcionális)</h5>
          <form @submit.prevent="uploadFile">
            <div class="mb-3">
              <input type="file" @change="handleFileChange" class="form-control" />
            </div>
            <button type="submit" class="btn btn-info">Feltöltés</button>
          </form>
          <div v-if="uploadMessage" class="mt-2">
            <p>{{ uploadMessage }}</p>
          </div>
        </div>

        <!-- Fájlok listája -->
        <div class="mt-4">
          <h5>Fájlok</h5>
          <div v-if="files.length === 0">
            <p>Nincs feltöltött fájl.</p>
          </div>
          <ul v-else class="list-group">
            <li v-for="(file, index) in files" :key="index" class="list-group-item d-flex justify-content-between align-items-center">
              <div>
                <div><strong>{{ file.link }}</strong></div>
                <div><small>{{ formatDate(file.created_at) }}</small></div>
              </div>
              <div>
                <!-- Megnyitás gomb -->
                <button class="btn btn-sm btn-outline-primary" @click="openFile(file.link)">
                  Megnyitás
                </button>
                <!-- Letöltés gomb -->
                <button class="btn btn-sm btn-outline-success ms-2" @click="downloadFileAlt(file.link)">
                  Letöltés
                </button>
              </div>
            </li>
          </ul>
        </div>

        <!-- Feladat befejezése gomb (modal megerősítéssel) -->
        <div class="mt-4 text-center">
          <button class="btn btn-success" @click="showConfirmModal = true">Feladat befejezése</button>
        </div>
      </div>
      <div v-else>
        <p>Nincs ilyen task.</p>
      </div>
    </div>

    <!-- Bootstrap modal a feladat befejezés megerősítésére -->
    <div v-if="showConfirmModal" class="modal fade show" style="display: block;" tabindex="-1">
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">Megerősítés</h5>
            <button type="button" class="btn-close" @click="cancelFinish"></button>
          </div>
          <div class="modal-body">
            <p>Biztosan be szeretnéd fejezni a feladatot?</p>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn btn-secondary" @click="cancelFinish">Mégse</button>
            <button type="button" class="btn btn-primary" @click="confirmFinish">Igen, befejezem</button>
          </div>
        </div>
      </div>
    </div>
    <!-- Modal backdrop -->
    <div v-if="showConfirmModal" class="modal-backdrop fade show"></div>
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
const files = ref([]); // A fájlok a backend "taskContents" kulcsában érkeznek
const isLoading = ref(false);
const errorMessage = ref("");

// Fájl feltöltési állapota
const fileToUpload = ref(null);
const uploadMessage = ref("");
const showConfirmModal = ref(false);

// formatDate függvényt arrow function-ként definiáljuk, hogy a template-ben elérhető legyen
const formatDate = (dateStr) => {
  if (!dateStr) return "";
  const dateObj = new Date(dateStr);
  return dateObj.toLocaleString();
};

function handleFileChange(event) {
  const filesArr = event.target.files;
  if (filesArr && filesArr.length > 0) {
    fileToUpload.value = filesArr[0];
    console.log("Selected file:", fileToUpload.value);
  }
}

async function uploadFile() {
  if (!fileToUpload.value) {
    uploadMessage.value = "Kérlek válassz egy fájlt!";
    return;
  }
  const formData = new FormData();
  formData.append("file", fileToUpload.value);
  formData.append("taskId", props.taskId);
  console.log("Uploading file for taskId:", props.taskId);
  try {
    const response = await fetch("http://127.0.0.1:8000/api/file/upload", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${props.token}`
      },
      body: formData
    });
    const data = await response.json();
    console.log("Upload response:", data);
    if (!response.ok) {
      throw new Error(data.error || "Fájl feltöltési hiba");
    }
    uploadMessage.value = "Fájl sikeresen feltöltve!";
    // Feltöltés után újra lekérjük a task adatokat
    await fetchTask();
  } catch (error) {
    console.error("Error uploading file:", error);
    uploadMessage.value = "Hiba: " + error.message;
  }
}

async function fetchTask() {
  isLoading.value = true;
  errorMessage.value = "";
  try {
    const response = await fetch(`http://127.0.0.1:8000/api/task/${props.taskId}`, {
      method: "GET",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${props.token}`
      }
    });
    const data = await response.json();
    console.log("Full fetch response:", data);
    if (!response.ok) {
      throw new Error(data.error || "Hiba történt a task lekérésekor!");
    }
    task.value = data.task || null;
    // A fájlokat a "taskContents" kulcs tartalmazza
    files.value = data.taskContents
      ? (Array.isArray(data.taskContents) ? data.taskContents : Object.values(data.taskContents))
      : [];
    console.log("Task fetched:", task.value);
    console.log("Files fetched:", files.value);
  } catch (error) {
    errorMessage.value = error.message;
  } finally {
    isLoading.value = false;
  }
}

async function openFile(filename) {
  console.log("Opening file with filename (link):", filename);
  try {
    const url = `http://127.0.0.1:8000/api/file/signed-url/${filename}`;
    const response = await fetch(url, {
      method: "GET",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${props.token}`
      }
    });
    const data = await response.json();
    console.log("Open file URL response:", data);
    if (!response.ok) {
      throw new Error(data.error || "Fájl megnyitási hiba");
    }
    const signedUrl = data.url;
    if (signedUrl) {
      window.open(signedUrl, "_blank", "noopener,noreferrer");
    } else {
      throw new Error("Nincs érvényes megnyitási URL");
    }
  } catch (error) {
    console.error("Error opening file:", error);
    alert("Hiba: " + error.message);
  }
}

async function downloadFileAlt(filename) {
  console.log("Download function triggered for filename (link):", filename);
  try {
    const url = `http://127.0.0.1:8000/api/file/signed-url/${filename}`;
    const response = await fetch(url, {
      method: "GET",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${props.token}`
      }
    });
    const data = await response.json();
    console.log("Download URL response:", data);
    if (!response.ok) {
      throw new Error(data.error || "Fájl letöltési hiba");
    }
    const signedUrl = data.url;
    if (signedUrl) {
      // Létrehozunk egy ideiglenes <a> elemet az automatikus letöltéshez
      const link = document.createElement("a");
      link.href = signedUrl;
      link.target = "_blank";
      link.rel = "noopener noreferrer";
      link.download = filename;
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    } else {
      throw new Error("Nincs érvényes letöltési URL");
    }
  } catch (error) {
    console.error("Error downloading file:", error);
    alert("Hiba: " + error.message);
  }
}

function confirmFinishTask() {
  showConfirmModal.value = true;
}

async function confirmFinish() {
  showConfirmModal.value = false;
  await finishTask();
}

function cancelFinish() {
  showConfirmModal.value = false;
}

async function finishTask() {
  try {
    const response = await fetch(`http://127.0.0.1:8000/api/task/${props.taskId}`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${props.token}`
      }
    });
    if (!response.ok) {
      const data = await response.json();
      throw new Error(data.error || "Hiba történt a feladat befejezésekor!");
    }
    const getResponse = await fetch(`http://127.0.0.1:8000/api/task/${props.taskId}`, {
      method: "GET",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${props.token}`
      }
    });
    const getData = await getResponse.json();
    let timeTaken = 0;
    if (getData.success) {
      const updatedTask = getData.task;
      if (updatedTask.state1date && updatedTask.state2date) {
        const start = new Date(updatedTask.state1date);
        const end = new Date(updatedTask.state2date);
        timeTaken = Math.floor((end - start) / 1000);
      }
    }
    console.log("Task finished, timeTaken:", timeTaken);
    emit('navigateToEndTask', { taskId: props.taskId, timeTaken: timeTaken });
  } catch (error) {
    console.error("Error finishing task:", error);
    alert("Hiba: " + error.message);
  }
}

onMounted(() => {
  if (props.token && props.taskId) fetchTask();
});
watch(() => props.taskId, (newVal) => {
  if (newVal) fetchTask();
});
</script>

<style scoped>
.card {
  margin-top: 20px;
}
.modal {
  background: rgba(0,0,0,0.5);
}
</style>
