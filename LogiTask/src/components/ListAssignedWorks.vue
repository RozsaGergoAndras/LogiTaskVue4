<template>
    <div class="card mt-4">
      <div class="card-header">
        Hozzárendelt feladatok listája
      </div>
      <div class="card-body">
        <!-- Dátum szűrő -->
        <div class="row mb-4">
          <div class="col-md-5">
            <label for="beginDate" class="form-label"><strong>Kezdő dátum</strong></label>
            <input v-model="beginDate" id="beginDate" type="date" class="form-control" />
          </div>
          <div class="col-md-5">
            <label for="endDate" class="form-label"><strong>Vég dátum</strong></label>
            <input v-model="endDate" id="endDate" type="date" class="form-control" />
          </div>
          <div class="col-md-2 d-flex align-items-end">
            <button @click="fetchAssigned" class="btn btn-primary w-100">Szűrés</button>
          </div>
        </div>
  
        <!-- Töltési állapot és hiba -->
        <div v-if="isLoading" class="text-center">
          <div class="spinner-border" role="status">
            <span class="visually-hidden">Betöltés...</span>
          </div>
          <p>Betöltés...</p>
        </div>
        <div v-else-if="errorMessage" class="alert alert-danger">
          {{ errorMessage }}
        </div>
  
        <!-- Táblázat a feladatokkal -->
        <div v-else>
          <table class="table table-striped">
            <thead>
              <tr>
                <th @click="sortBy('id')" style="cursor:pointer">
                  ID <span v-if="sortKey==='id'">{{ sortAsc ? '↑' : '↓' }}</span>
                </th>
                <th @click="sortBy('description')" style="cursor:pointer">
                  Leírás <span v-if="sortKey==='description'">{{ sortAsc ? '↑' : '↓' }}</span>
                </th>
                <th @click="sortBy('state')" style="cursor:pointer">
                  Állapot <span v-if="sortKey==='state'">{{ sortAsc ? '↑' : '↓' }}</span>
                </th>
                <th @click="sortBy('state0date')" style="cursor:pointer">
                  Kiadás dátuma <span v-if="sortKey==='state0date'">{{ sortAsc ? '↑' : '↓' }}</span>
                </th>
                <th @click="sortBy('state1date')" style="cursor:pointer">
                  Megkezdés dátuma <span v-if="sortKey==='state1date'">{{ sortAsc ? '↑' : '↓' }}</span>
                </th>
                <th @click="sortBy('state2date')" style="cursor:pointer">
                  Befejezés dátuma <span v-if="sortKey==='state2date'">{{ sortAsc ? '↑' : '↓' }}</span>
                </th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="task in sortedTasks" :key="task.id">
                <td>{{ task.id }}</td>
                <td>{{ task.description }}</td>
                <td>{{ formatState(task.state) }}</td>
                <td>{{ task.state0date ? formatDate(task.state0date) : 'Nincs' }}</td>
                <td>{{ task.state1date ? formatDate(task.state1date) : 'Nincs' }}</td>
                <td>{{ task.state2date ? formatDate(task.state2date) : 'Nincs' }}</td>
                <td>
                  <button
                    v-if="task.files && task.files.length"
                    class="btn btn-sm btn-outline-secondary"
                    @click="viewFiles(task)"
                  >
                    Fájlok megtekintése
                  </button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
  
        <!-- Modal a feladat részleteivel és fájlokkal -->
        <div v-if="showModal" class="modal fade show" style="display: block;" tabindex="-1">
          <div class="modal-dialog modal-lg">
            <div class="modal-content">
              <div class="modal-header">
                <h5 class="modal-title">Feladat részletek - #{{ selectedTask.id }}</h5>
                <button type="button" class="btn-close" @click="closeModal"></button>
              </div>
              <div class="modal-body">
                <ul class="list-group list-group-flush mb-3">
                  <li class="list-group-item"><strong>Leírás:</strong> {{ selectedTask.description }}</li>
                  <li class="list-group-item"><strong>Kiadás dátuma:</strong> {{ selectedTask.state0date ? formatDate(selectedTask.state0date) : 'Nincs' }}</li>
                  <li class="list-group-item"><strong>Megkezdés dátuma:</strong> {{ selectedTask.state1date ? formatDate(selectedTask.state1date) : 'Nincs' }}</li>
                  <li class="list-group-item"><strong>Befejezés dátuma:</strong> {{ selectedTask.state2date ? formatDate(selectedTask.state2date) : 'Nincs' }}</li>
                </ul>
                <h6>Fájlok</h6>
                <div v-if="selectedTask.files.length === 0">
                  <p>Nincs feltöltött fájl.</p>
                </div>
                <ul v-else class="list-group">
                  <li v-for="file in selectedTask.files" :key="file.link" class="list-group-item d-flex justify-content-between align-items-center">
                    <div>
                      <strong>{{ file.link }}</strong>
                      <div><small>{{ file.created_at ? formatDate(file.created_at) : '' }}</small></div>
                    </div>
                    <button class="btn btn-sm btn-outline-primary" @click="openFile(file.link)">
                      Megnyitás
                    </button>
                  </li>
                </ul>
              </div>
              <div class="modal-footer">
                <button type="button" class="btn btn-secondary" @click="closeModal">Bezárás</button>
              </div>
            </div>
          </div>
        </div>
        <div v-if="showModal" class="modal-backdrop fade show"></div>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted, computed } from 'vue';
  import { defineProps } from 'vue';
  
  const props = defineProps({ token: { type: String, required: true } });
  
  const tasks = ref([]);
  const isLoading = ref(false);
  const errorMessage = ref('');
  const beginDate = ref('');
  const endDate = ref('');
  const showModal = ref(false);
  const selectedTask = ref({ files: [] });
  
  const sortKey = ref('id');
  const sortAsc = ref(true);
  
  function formatDate(dateStr) {
    return new Date(dateStr).toLocaleString();
  }
  
  function formatState(state) {
    if (state === 2) return 'Befejezett';
    if (state === 1) return 'Megkezdett';
    if (state === 0) return 'Kiadott';
    return '-';
  }
  
  async function fetchTaskContent(task) {
    try {
      const res = await fetch(`http://127.0.0.1:8000/api/task/${task.id}`, { headers: { 'Authorization': `Bearer ${props.token}` } });
      const data = await res.json();
      const raw = data.taskContents || data.taskContent || [];
      task.files = Array.isArray(raw) ? raw : Object.values(raw);
    } catch {
      task.files = [];
    }
  }
  
  async function fetchAssigned() {
    isLoading.value = true;
    errorMessage.value = '';
    try {
      const res = await fetch('http://127.0.0.1:8000/api/tasks/author', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${props.token}` },
        body: JSON.stringify({ begin_date: beginDate.value, end_date: endDate.value })
      });
      const data = await res.json();
      if (!res.ok || !data.success) throw new Error(data.error || 'Hiba a feladatok lekérésekor');
      tasks.value = data.tasks.map(t => ({ ...t, files: [] }));
      await Promise.all(tasks.value.map(fetchTaskContent));
    } catch (e) {
      errorMessage.value = e.message;
    } finally {
      isLoading.value = false;
    }
  }
  
  function viewFiles(task) {
    selectedTask.value = { ...task };
    showModal.value = true;
  }
  
  function closeModal() {
    showModal.value = false;
  }
  
  async function openFile(link) {
    const res = await fetch(`http://127.0.0.1:8000/api/file/signed-url/${link}`, { headers: { 'Authorization': `Bearer ${props.token}` } });
    const { url } = await res.json();
    window.open(url, '_blank');
  }
  
  function sortBy(key) {
    if (sortKey.value === key) {
      sortAsc.value = !sortAsc.value;
    } else {
      sortKey.value = key;
      sortAsc.value = true;
    }
  }
  
  const sortedTasks = computed(() => {
    return [...tasks.value].sort((a, b) => {
      let va = a[sortKey.value] || '';
      let vb = b[sortKey.value] || '';
      if (sortKey.value.includes('date')) {
        va = va ? new Date(va).getTime() : 0;
        vb = vb ? new Date(vb).getTime() : 0;
      }
      if (va < vb) return sortAsc.value ? -1 : 1;
      if (va > vb) return sortAsc.value ? 1 : -1;
      return 0;
    });
  });
  
  onMounted(() => {
    const today = new Date();
    const lastMonth = new Date(today);
    lastMonth.setMonth(today.getMonth() - 1);
    const tomorrow = new Date(today);
    tomorrow.setDate(today.getDate() + 1);
    beginDate.value = lastMonth.toISOString().slice(0, 10);
    endDate.value = tomorrow.toISOString().slice(0, 10);
    fetchAssigned();
  });
  </script>
  
  <style scoped>
  .card { margin-top: 20px; }
  ul { list-style: none; padding-left: 0; margin: 0; }
  .modal { background: rgba(0,0,0,0.5); }
  </style>