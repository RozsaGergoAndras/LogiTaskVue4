<template>
  <div class="card shadow-sm mt-4">
    <!-- Header -->
    <div class="card-header bg-dark text-white">
      <h5 class="mb-0">Admin Panel</h5>
    </div>
    <div class="card-body">
      <div class="row">
        <!-- Sidebar menu -->
        <div class="col-md-3 border-end pe-3">
          <ul class="list-group">
            <li
              v-for="item in menuItems"
              :key="item.key"
              class="list-group-item list-group-item-action"
              :class="{ active: selectedMenu === item.key }"
              @click="selectedMenu = item.key"
            >
              {{ item.label }}
            </li>
          </ul>
        </div>

        <!-- Main content -->
        <div class="col-md-9 ps-4">
          <!-- 1) Add User -->
          <div v-if="selectedMenu === 'user'">
            <form @submit.prevent="createUser">
              <div class="mb-3">
                <label class="form-label">Név</label>
                <input v-model="userName" class="form-control" required maxlength="255" />
              </div>
              <div class="mb-3">
                <label class="form-label">Email</label>
                <input v-model="userEmail" type="email" class="form-control" required maxlength="255" />
              </div>
              <div class="mb-3">
                <label class="form-label">Jelszó</label>
                <input v-model="userPassword" type="password" class="form-control" required maxlength="255" />
              </div>
              <div class="mb-3">
                <label class="form-label">Jelszó megerősítése</label>
                <input v-model="userPasswordConfirmation" type="password" class="form-control" required maxlength="255" />
              </div>
              <div class="mb-3">
                <label class="form-label">Szerep</label>
                <select v-model="userRole" class="form-select" required>
                  <option disabled value="">Válassz szerepet</option>
                  <option v-for="r in roles" :key="r.id" :value="r.id">{{ r.role_name }}</option>
                </select>
              </div>
              <button class="btn btn-outline-dark">Hozzáadás</button>
            </form>
          </div>

          <!-- 2) Add Task Type -->
          <div v-else-if="selectedMenu === 'taskType'">
            <form @submit.prevent="createTaskType">
              <div class="mb-3">
                <label class="form-label">Típus név</label>
                <input v-model="typeName" class="form-control" required maxlength="255" />
              </div>
              <div class="mb-3">
                <label class="form-label">Hozzárendelhető szerep</label>
                <select v-model="assignableRole" class="form-select" required>
                  <option disabled value="">Válassz szerepet</option>
                  <option v-for="r in roles" :key="r.id" :value="r.id">{{ r.role_name }}</option>
                </select>
              </div>
              <button class="btn btn-outline-dark">Hozzáadás</button>
            </form>
          </div>

          <!-- 3) Create Task -->
          <div v-else-if="selectedMenu === 'task'">
            <form @submit.prevent="createTask">
              <div class="mb-3">
                <label class="form-label">Feladattípus</label>
                <select v-model="taskTypeId" class="form-select" required>
                  <option disabled value="">Válassz típust</option>
                  <option v-for="t in taskTypes" :key="t.id" :value="t.id">{{ t.type_name }}</option>
                </select>
              </div>
              <div class="mb-3">
                <label class="form-label">Végrehajtó</label>
                <select v-model="worker" class="form-select">
                  <option value="">Automatikus beosztás</option>
                  <option v-for="u in userList" :key="u.id" :value="u.id">{{ u.name }}</option>
                </select>
              </div>
              <div class="mb-3">
                <label class="form-label">Leírás</label>
                <textarea v-model="description" class="form-control" required maxlength="2000"></textarea>
              </div>
              <button class="btn btn-outline-dark">Hozzáadás</button>
            </form>
          </div>

          <!-- 4) Robot Task -->
          <div v-else-if="selectedMenu === 'robotTask'">
            <form @submit.prevent="createRobotTask">
              <div class="mb-3">
                <label class="form-label">Feladattípus</label>
                <select v-model="robotType" class="form-select" required>
                  <option disabled value="">Válassz típust</option>
                  <option v-for="t in robotTypes" :key="t.id" :value="t.id">{{ t.type_name }}</option>
                </select>
              </div>
              <div class="mb-3">
                <label class="form-label">Művelet</label>
                <select v-model="robotMethod" class="form-select" required>
                  <option disabled value="">GET vagy STORE</option>
                  <option value="GET">GET</option>
                  <option value="STORE">STORE</option>
                </select>
              </div>
              <div class="row">
                <div class="col-md-6 mb-3">
                  <label class="form-label">X koordináta (0–2)</label>
                  <input type="number" v-model.number="robotParam1" class="form-control" min="0" max="2" required />
                </div>
                <div class="col-md-6 mb-3">
                  <label class="form-label">Y koordináta (0–2)</label>
                  <input type="number" v-model.number="robotParam2" class="form-control" min="0" max="2" required />
                </div>
              </div>
              <div class="mb-3">
                <label class="form-label">Robot kiválasztása</label>
                <select v-model="robotWorker" class="form-select" required>
                  <option disabled value="">Válassz robotot</option>
                  <option v-for="u in robotList" :key="u.id" :value="u.id">{{ u.name }}</option>
                </select>
              </div>
              <button class="btn btn-outline-dark">Kiadás</button>
            </form>
          </div>

          <!-- Back -->
          <div class="mt-4 text-center">
            <button class="btn btn-secondary" @click="backToMain">Vissza</button>
          </div>
        </div>
      </div>
    </div>

    <!-- Result Modal -->
    <div v-if="showResultModal" class="modal fade show" style="display:block;" tabindex="-1">
      <div class="modal-dialog">
        <div class="modal-content">
          <div
            class="modal-header"
            :class="resultModalType === 'alert-success' ? 'bg-success text-white' : 'bg-danger text-white'"
          >
            <h5 class="modal-title">Értesítés</h5>
            <button type="button" class="btn-close btn-close-white" @click="closeResultModal"></button>
          </div>
          <div class="modal-body">
            <p>{{ resultModalMessage }}</p>
          </div>
          <div class="modal-footer">
            <button class="btn btn-outline-dark" @click="closeResultModal">Bezárás</button>
          </div>
        </div>
      </div>
      
    </div>
    <div class="modal-backdrop fade show" v-if="showResultModal" style="display:block;"></div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue';
import { defineProps, defineEmits } from 'vue';

const props = defineProps({ token: String });
const emit  = defineEmits(['backToTaskInfo']);

const menuItems = [
  { key: 'user',      label: 'Felhasználó hozzáadása' },
  { key: 'taskType',  label: 'Feladat típus hozzáadása' },
  { key: 'task',      label: 'Feladat létrehozása' },
  { key: 'robotTask', label: 'Robot feladat kiadása' }
];
const selectedMenu = ref('user');

// Shared data
const roles         = ref([]);
const taskTypes     = ref([]);
const currentUserId = ref('');

// 1) Add User
const userName                 = ref('');
const userEmail                = ref('');
const userPassword             = ref('');
const userPasswordConfirmation = ref('');
const userRole                 = ref('');

// 2) Add Task Type
const typeName       = ref('');
const assignableRole = ref('');

// 3) Create Task
const userList    = ref([]);
const worker      = ref('');
const taskTypeId  = ref('');
const description = ref('');

// 4) Robot Task
const robotTypes  = computed(() =>
  taskTypes.value.filter(t => t.assignable_role === 6)
);
const robotType    = ref('');
const robotMethod  = ref('');
const robotParam1  = ref(0);
const robotParam2  = ref(0);
const robotList    = ref([]);
const robotWorker  = ref('');

// Modal result
const showResultModal    = ref(false);
const resultModalMessage = ref('');
const resultModalType    = ref('alert-success');
function openResult(type, msg) {
  resultModalType.value    = type === 'error' ? 'alert-danger' : 'alert-success';
  resultModalMessage.value = msg;
  showResultModal.value    = true;
}
function closeResultModal() {
  showResultModal.value = false;
}

// Fetch initial lists
onMounted(async () => {
  const [r1, r2, r3] = await Promise.all([
    fetch('http://127.0.0.1:8000/api/roles',     { headers:{ Authorization:`Bearer ${props.token}` }}),
    fetch('http://127.0.0.1:8000/api/tasktype', { headers:{ Authorization:`Bearer ${props.token}` }}),
    fetch('http://127.0.0.1:8000/api/user',     { headers:{ Authorization:`Bearer ${props.token}` }})
  ]);
  roles.value         = (await r1.json()).roles    || [];
  taskTypes.value     = (await r2.json()).data     || [];
  currentUserId.value = (await r3.json()).id       || '';
});

// 1) createUser
async function createUser() {
  const res = await fetch('http://127.0.0.1:8000/api/user/create', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization:   `Bearer ${props.token}`
    },
    body: JSON.stringify({
      name:                  userName.value,
      email:                 userEmail.value,
      password:              userPassword.value,
      password_confirmation: userPasswordConfirmation.value,
      role:                  userRole.value
    })
  });
  const j = await res.json();
  console.log('createUser resp:', j);
  if (!res.ok) openResult('error', j.error || 'Hiba a felhasználó hozzáadásakor');
  else {
    openResult('success', 'Felhasználó sikeresen hozzáadva');
    userName.value = userEmail.value = userPassword.value = userPasswordConfirmation.value = userRole.value = '';
  }
}

// 2) createTaskType
async function createTaskType() {
  const res = await fetch('http://127.0.0.1:8000/api/tasktype/create/new', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization:   `Bearer ${props.token}`
    },
    body: JSON.stringify({
      type_name:       typeName.value,
      assignable_role: assignableRole.value
    })
  });
  const j = await res.json();
  console.log('createTaskType resp:', j);
  if (!res.ok) openResult('error', j.error || 'Hiba a típus hozzáadásakor');
  else {
    openResult('success', 'Feladattípus sikeresen hozzáadva');
    typeName.value = assignableRole.value = '';
    // Refresh taskTypes
    const r = await fetch('http://127.0.0.1:8000/api/tasktype', { headers:{ Authorization:`Bearer ${props.token}` }});
    taskTypes.value = (await r.json()).data || [];
  }
}

// 3) Create Task: watch & fetch users
watch(taskTypeId, val => {
  if (val) fetchUsersByTaskType(val);
  else     userList.value = [];
});
async function fetchUsersByTaskType(id) {
  const res = await fetch('http://127.0.0.1:8000/api/users', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization:   `Bearer ${props.token}`
    },
    body: JSON.stringify({ task_type_id: id })
  });
  const j = await res.json();
  userList.value = Array.isArray(j) ? j : j.users || [];
}
async function createTask() {
  const res = await fetch('http://127.0.0.1:8000/api/task/create/new', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization:   `Bearer ${props.token}`
    },
    body: JSON.stringify({
      assigner:    currentUserId.value,
      worker:      worker.value || null,
      task_type:   taskTypeId.value,
      description: description.value
    })
  });
  const j = await res.json();
  console.log('createTask resp:', j);
  if (!res.ok) openResult('error', j.error || 'Hiba a feladat létrehozásakor');
  else {
    openResult('success', 'Feladat sikeresen létrehozva');
    worker.value = taskTypeId.value = description.value = '';
    userList.value = [];
  }
}

// 4) Robot users list
watch(robotType, async id => {
  if (!id) return robotList.value = [];
  const res = await fetch('http://127.0.0.1:8000/api/users', {
    method:'POST',
    headers:{ 'Content-Type':'application/json', Authorization:`Bearer ${props.token}` },
    body: JSON.stringify({ task_type_id: id })
  });
  const j = await res.json();
  console.log('robot users resp:', j);
  robotList.value = Array.isArray(j) ? j : j.users || [];
});

// 4) createRobotTask
async function createRobotTask() {
  const desc = `${robotMethod.value} ${robotParam1.value}:${robotParam2.value}`;
  const payload = {
    assigner:    currentUserId.value,
    worker:      robotWorker.value,
    task_type:   robotType.value,
    description: desc
  };
  console.log('createRobotTask payload:', payload);
  const res = await fetch('http://127.0.0.1:8000/api/task/create/new', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization:   `Bearer ${props.token}`
    },
    body: JSON.stringify(payload)
  });
  const j = await res.json();
  console.log('createRobotTask resp:', j);
  if (!res.ok) openResult('error', j.error || 'Hiba a robot feladat kiadásakor');
  else {
    openResult('success', 'Robot feladat sikeresen kiadva');
    robotType.value = robotMethod.value = robotWorker.value = '';
    robotParam1.value = robotParam2.value = 0;
  }
}

// Back action
function backToMain() {
  emit('backToTaskInfo');
}
</script>

<style scoped>
.card-header           { background-color: #343a40; color: #fff; }
.list-group-item       { cursor: pointer; border: none; }
.list-group-item.active{ background-color: #f8f9fa; color: #212529; font-weight: bold; }
.btn-outline-dark      { border-width: 2px; }
.form-control:focus    { border-color: #343a40 !important; box-shadow: none !important; }
.mb-3 label            { font-weight: 600; }
</style>
