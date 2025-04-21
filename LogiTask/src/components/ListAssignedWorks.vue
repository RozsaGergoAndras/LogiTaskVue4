<template>
  <div class="card shadow-sm mt-4" style="height: 80vh;">
    <!-- Fejléc -->
    <div class="card-header bg-dark text-white">
      <h5 class="mb-0">Hozzárendelt feladatok listája</h5>
    </div>
    <div class="card-body d-flex flex-column px-3" style="height: calc(100% - 56px);">
      <!-- Szűrők -->
      <div class="row mb-3 flex-shrink-0">
        <div class="col-md-5">
          <label class="form-label"><strong>Kezdő dátum</strong></label>
          <input v-model="beginDate" type="date" class="form-control" />
        </div>
        <div class="col-md-5">
          <label class="form-label"><strong>Vég dátum</strong></label>
          <input v-model="endDate" type="date" class="form-control" />
        </div>
        <div class="col-md-2 d-flex align-items-end">
          <button @click="fetchAssigned" class="btn btn-outline-dark w-100">
            Szűrés
          </button>
        </div>
      </div>

      <!-- Oldalméret -->
      <div class="d-flex align-items-center mb-3 flex-shrink-0">
        <label class="form-label me-2"><strong>Oldalméret:</strong></label>
        <input
          v-model.number="pageSize"
          type="number"
          min="1"
          class="form-control w-auto"
        />
      </div>

      <!-- Betöltés / hiba -->
      <div v-if="isLoading" class="text-center mb-3 flex-shrink-0">
        <div class="spinner-border text-dark" role="status">
          <span class="visually-hidden">Betöltés...</span>
        </div>
      </div>
      <div v-else-if="errorMessage" class="alert alert-danger mb-3 flex-shrink-0">
        {{ errorMessage }}
      </div>

      <!-- Táblázat görgethetően -->
      <div class="table-responsive flex-fill mb-3">
        <table class="table table-striped mb-0">
          <thead class="table-dark text-white">
            <tr>
              <th @click="sortBy('id')" class="cursor-pointer">
                ID <span v-if="sortKey==='id'">{{ sortAsc ? '↑' : '↓' }}</span>
              </th>
              <th @click="sortBy('description')" class="cursor-pointer">
                Leírás <span v-if="sortKey==='description'">{{ sortAsc ? '↑' : '↓' }}</span>
              </th>
              <th @click="sortBy('state')" class="cursor-pointer">
                Állapot <span v-if="sortKey==='state'">{{ sortAsc ? '↑' : '↓' }}</span>
              </th>
              <th @click="sortBy('state0date')" class="cursor-pointer">
                Kiadás dátuma <span v-if="sortKey==='state0date'">{{ sortAsc ? '↑' : '↓' }}</span>
              </th>
              <th @click="sortBy('state1date')" class="cursor-pointer">
                Megkezdés dátuma <span v-if="sortKey==='state1date'">{{ sortAsc ? '↑' : '↓' }}</span>
              </th>
              <th @click="sortBy('state2date')" class="cursor-pointer">
                Befejezés dátuma <span v-if="sortKey==='state2date'">{{ sortAsc ? '↑' : '↓' }}</span>
              </th>
              <th>Akciók</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="task in paginatedTasks" :key="task.id">
              <td>{{ task.id }}</td>
              <td>{{ task.description }}</td>
              <td>{{ formatState(task.state) }}</td>
              <td>{{ task.state0date ? formatDate(task.state0date) : 'Nincs' }}</td>
              <td>{{ task.state1date ? formatDate(task.state1date) : 'Nincs' }}</td>
              <td>{{ task.state2date ? formatDate(task.state2date) : 'Nincs' }}</td>
              <td>
                <button
                  v-if="task.files && task.files.length"
                  class="btn btn-sm btn-outline-dark me-2"
                  @click="viewFiles(task)"
                >Fájlok</button>
                <button
                  v-if="!task.state2date"
                  class="btn btn-sm btn-outline-danger"
                  @click="confirmDelete(task)"
                >Törlés</button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Pagináció -->
      <nav class="mb-3 flex-shrink-0">
        <ul class="pagination justify-content-center mb-0">
          <li class="page-item" :class="{ disabled: currentPage===1 }">
            <button class="page-link"
              @click="currentPage--"
              :disabled="currentPage===1"
            >Előző</button>
          </li>
          <li
            class="page-item"
            v-for="n in totalPages"
            :key="n"
            :class="{ active: n===currentPage }"
          >
            <button class="page-link" @click="currentPage=n">{{ n }}</button>
          </li>
          <li class="page-item" :class="{ disabled: currentPage===totalPages }">
            <button class="page-link"
              @click="currentPage++"
              :disabled="currentPage===totalPages"
            >Következő</button>
          </li>
        </ul>
      </nav>
    </div>

    <!-- Részletek modal -->
    <div v-if="showModal" class="modal fade show" style="display:block;" tabindex="-1">
      <div class="modal-dialog modal-lg">
        <div class="modal-content">
          <div class="modal-header bg-dark text-white">
            <h5 class="modal-title">Feladat részletek – #{{ selectedTask.id }}</h5>
            <button type="button" class="btn-close" @click="closeModal"></button>
          </div>
          <div class="modal-body">
            <ul class="list-group list-group-flush mb-3">
              <li class="list-group-item"><strong>Leírás:</strong> {{ selectedTask.description }}</li>
              <li class="list-group-item"><strong>Kiadás:</strong> {{ selectedTask.state0date ? formatDate(selectedTask.state0date) : 'Nincs' }}</li>
              <li class="list-group-item"><strong>Megkezdés:</strong> {{ selectedTask.state1date ? formatDate(selectedTask.state1date) : 'Nincs' }}</li>
              <li class="list-group-item"><strong>Befejezés:</strong> {{ selectedTask.state2date ? formatDate(selectedTask.state2date) : 'Nincs' }}</li>
            </ul>
            <h6>Fájlok</h6>
            <div v-if="!selectedTask.files.length"><p>Nincs feltöltött fájl.</p></div>
            <ul v-else class="list-group">
              <li v-for="file in selectedTask.files" :key="file.link" class="list-group-item d-flex justify-content-between align-items-center">
                <div>
                  <strong>{{ file.link }}</strong><br>
                  <small>{{ file.created_at ? formatDate(file.created_at) : '' }}</small>
                </div>
                <button class="btn btn-sm btn-outline-dark" @click="openFile(file.link)">Megnyitás</button>
              </li>
            </ul>
          </div>
          <div class="modal-footer">
            <button class="btn btn-secondary" @click="closeModal">Bezárás</button>
          </div>
        </div>
      </div>
    </div>
    <div v-if="showModal" class="modal-backdrop fade show"></div>

    <!-- Törlés megerősítő modal -->
    <div v-if="showDeleteModal" class="modal fade show" style="display:block;" tabindex="-1">
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header bg-dark text-white">
            <h5 class="modal-title">Megerősítés</h5>
            <button type="button" class="btn-close" @click="cancelDelete"></button>
          </div>
          <div class="modal-body">
            <p>Biztosan törölni szeretnéd a(z) #{{ deleteTarget.id }} feladatot?</p>
          </div>
          <div class="modal-footer">
            <button class="btn btn-secondary" @click="cancelDelete">Mégse</button>
            <button class="btn btn-danger" @click="deleteTaskConfirmed">Törlés</button>
          </div>
        </div>
      </div>
    </div>
    <div v-if="showDeleteModal" class="modal-backdrop fade show"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { defineProps } from 'vue';

const props = defineProps({ token: String });

const tasks = ref([]);
const isLoading = ref(false);
const errorMessage = ref('');
const beginDate = ref('');
const endDate = ref('');
const pageSize = ref(10);
const currentPage = ref(1);
const sortKey = ref('id');
const sortAsc = ref(true);

const showModal = ref(false);
const selectedTask = ref({ files: [] });
const showDeleteModal = ref(false);
const deleteTarget = ref(null);

const totalPages = computed(() =>
  Math.max(1, Math.ceil(tasks.value.length / pageSize.value))
);
const paginatedTasks = computed(() =>
  sortedTasks.value.slice(
    (currentPage.value - 1) * pageSize.value,
    currentPage.value   * pageSize.value
  )
);

watch(pageSize, val => {
  pageSize.value = val < 1 ? 1 : val;
  currentPage.value = 1;
});

const formatDate = s => new Date(s).toLocaleString();
const formatState = s =>
  s === 2 ? 'Befejezett' : s === 1 ? 'Megkezdett' : 'Kiadott';

async function fetchTaskContent(task) {
  try {
    const res = await fetch(`http://127.0.0.1:8000/api/task/${task.id}`, {
      headers: { Authorization: `Bearer ${props.token}` }
    });
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
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${props.token}`
      },
      body: JSON.stringify({
        begin_date: beginDate.value,
        end_date:   endDate.value
      })
    });
    const data = await res.json();
    if (!res.ok || !data.success)
      throw new Error(data.error || 'Hiba a lekéréskor');
    tasks.value = data.tasks.map(t => ({ ...t, files: [] }));
    await Promise.all(tasks.value.map(fetchTaskContent));
    currentPage.value = 1;
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
  try {
    const res = await fetch(
      `http://127.0.0.1:8000/api/file/signed-url/${link}`,
      { headers: { Authorization: `Bearer ${props.token}` } }
    );
    const { url } = await res.json();
    window.open(url, '_blank');
  } catch (e) {
    alert('Hiba a fájl megnyitásakor: ' + e.message);
  }
}

function confirmDelete(task) {
  deleteTarget.value = task;
  showDeleteModal.value = true;
}
function cancelDelete() {
  deleteTarget.value = null;
  showDeleteModal.value = false;
}
async function deleteTaskConfirmed() {
  try {
    const res = await fetch(
      `http://127.0.0.1:8000/api/task/${deleteTarget.value.id}`,
      {
        method: 'DELETE',
        headers: { Authorization: `Bearer ${props.token}` }
      }
    );
    if (!res.ok) {
      const err = await res.json();
      throw new Error(err.error || 'Törlés sikertelen');
    }
    showDeleteModal.value = false;
    await fetchAssigned();
  } catch (e) {
    alert('Hiba a törléskor: ' + e.message);
  }
}

const sortBy = key => {
  if (sortKey.value === key) sortAsc.value = !sortAsc.value;
  else {
    sortKey.value = key;
    sortAsc.value = true;
  }
};

const sortedTasks = computed(() =>
  [...tasks.value].sort((a, b) => {
    let va = a[sortKey.value] || '', vb = b[sortKey.value] || '';
    if (sortKey.value.includes('date')) {
      va = va ? new Date(va).getTime() : 0;
      vb = vb ? new Date(vb).getTime() : 0;
    }
    return (va < vb ? -1 : va > vb ? 1 : 0) * (sortAsc.value ? 1 : -1);
  })
);

onMounted(() => {
  const d = new Date(), lm = new Date(d); lm.setMonth(d.getMonth()-1);
  const tm = new Date(d); tm.setDate(d.getDate()+1);
  beginDate.value = lm.toISOString().slice(0,10);
  endDate.value   = tm.toISOString().slice(0,10);
  fetchAssigned();
});
</script>

<style scoped>
.card-header {
  background-color: #343a40;
  color: #ffffff;
}
.cursor-pointer { cursor: pointer; }

/* Input fókusz: sötét szegély, nincs box-shadow */
.form-control:focus {
  border-color: #343a40 !important;
  box-shadow: none !important;
}

/* Lapozó linkek semleges stílus */
.pagination .page-link {
  color: #343a40;
  background-color: transparent;
  border: 1px solid #dee2e6;
}
.pagination .page-link:hover,
.pagination .page-link:focus {
  background-color: #f8f9fa;
  box-shadow: none;
}
.pagination .page-item.active .page-link {
  background-color: #f8f9fa;
  color: #212529;
  border-color: #dee2e6;
}

/* Görgethető táblázat */
.table-responsive {
  max-height: calc(80vh - 200px);
  overflow-y: auto;
}
.modal {
  background: rgba(0, 0, 0, 0.5);
}
</style>
