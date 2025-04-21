<template>
  <div class="card shadow-sm mt-4" style="he">
    <!-- Fejléc a navbar sötét színével -->
    <div class="card-header bg-dark text-white">
      <h5 class="mb-0">Admin Panel</h5>
    </div>
    <div class="card-body">
      <div class="row">
        <!-- Menü a kártyán belül, separatorral -->
        <div class="col-md-3 border-end pe-3">
          <ul class="list-group">
            <li
              v-for="(item, idx) in menuItems"
              :key="item.key"
              class="list-group-item list-group-item-action"
              :class="{ active: selectedMenu === item.key }"
              @click="selectedMenu = item.key"
            >
              {{ item.label }}
            </li>
          </ul>
        </div>

        <!-- Tartalom -->
        <div class="col-md-9 ps-4">
          <!-- Felhasználó hozzáadása -->
          <div v-if="selectedMenu === 'user'">
            <form @submit.prevent="createUser">
              <div class="mb-3">
                <label class="form-label">Név</label>
                <input v-model="userName" class="form-control" required maxlength="255"/>
              </div>
              <div class="mb-3">
                <label class="form-label">Email</label>
                <input v-model="userEmail" type="email" class="form-control" required maxlength="255"/>
              </div>
              <div class="mb-3">
                <label class="form-label">Jelszó</label>
                <input v-model="userPassword" type="password" class="form-control" required maxlength="255"/>
              </div>
              <div class="mb-3">
                <label class="form-label">Jelszó megerősítése</label>
                <input v-model="userPasswordConfirmation" type="password" class="form-control" required maxlength="255"/>
              </div>
              <div class="mb-3">
                <label class="form-label">Szerep</label>
                <select v-model="userRole" class="form-select" required>
                  <option disabled value="">Válassz szerepet</option>
                  <option v-for="r in roles" :key="r.id" :value="r.id">{{ r.role_name }}</option>
                </select>
              </div>
              <!-- Mentés gomb fekete kerettel -->
              <button class="btn btn-outline-dark">Hozzáadás</button>
            </form>
            <p class="mt-3 text-success" v-if="userResultMessage">{{ userResultMessage }}</p>
          </div>

          <!-- Feladat típus hozzáadása -->
          <div v-else-if="selectedMenu === 'taskType'">
            <form @submit.prevent="createTaskType">
              <div class="mb-3">
                <label class="form-label">Típus név</label>
                <input v-model="typeName" class="form-control" required maxlength="255"/>
              </div>
              <div class="mb-3">
                <label class="form-label">Hozzárendelhető szerep</label>
                <select v-model="assignableRole" class="form-select" required>
                  <option disabled value="">Válassz szerepet</option>
                  <option v-for="r in roles" :key="r.id" :value="r.id">{{ r.role_name }}</option>
                </select>
              </div>
              <!-- Mentés gomb fekete kerettel -->
              <button class="btn btn-outline-dark">Hozzáadás</button>
            </form>
            <p class="mt-3 text-success" v-if="taskTypeResultMessage">{{ taskTypeResultMessage }}</p>
          </div>

          <!-- Feladat létrehozása -->
          <div v-else>
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
                <textarea name="" id="" v-model="description" class="form-control" required maxlength="2000"></textarea>
              </div>
              <!-- Mentés gomb fekete kerettel -->
              <button class="btn btn-outline-dark">Hozzáadás</button>
            </form>
            <p class="mt-3 text-success" v-if="taskResultMessage">{{ taskResultMessage }}</p>
          </div>

          <!-- Vissza gomb -->
          <div class="mt-4 text-center">
            <button class="btn btn-secondary" @click="backToMain">Vissza</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';
import { defineProps, defineEmits } from 'vue';

const props = defineProps({ token: String });
const emit = defineEmits(['backToTaskInfo']);

const selectedMenu = ref('user');
const menuItems = [
  { key: 'user',     label: 'Felhasználó hozzáadása' },
  { key: 'taskType', label: 'Feladat típus hozzáadása' },
  { key: 'task',     label: 'Feladat létrehozása' }
];

const roles = ref([]);
const userList = ref([]);
const taskTypes = ref([]);
const currentUserId = ref('');

const userName = ref('');
const userEmail = ref('');
const userPassword = ref('');
const userPasswordConfirmation = ref('');
const userRole = ref('');
const userResultMessage = ref('');

const typeName = ref('');
const assignableRole = ref('');
const taskTypeResultMessage = ref('');

const worker = ref('');
const taskTypeId = ref('');
const description = ref('');
const taskResultMessage = ref('');

async function fetchRoles() {
  const res = await fetch('http://127.0.0.1:8000/api/roles', {
    headers: { Authorization: `Bearer ${props.token}` }
  });
  const j = await res.json();
  roles.value = j.roles || [];
}
async function fetchTaskTypes() {
  const res = await fetch('http://127.0.0.1:8000/api/tasktype', {
    headers: { Authorization: `Bearer ${props.token}` }
  });
  const j = await res.json();
  taskTypes.value = j.data || [];
}
async function fetchCurrentUser() {
  const res = await fetch('http://127.0.0.1:8000/api/user', {
    headers: { Authorization: `Bearer ${props.token}` }
  });
  const j = await res.json();
  if (res.ok) currentUserId.value = j.id;
}
async function fetchUsersByTaskType(id) {
  const res = await fetch('http://127.0.0.1:8000/api/users', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${props.token}`
    },
    body: JSON.stringify({ task_type_id: id })
  });
  const j = await res.json();
  userList.value = Array.isArray(j) ? j : j.users || [];
}

watch(taskTypeId, val => {
  if (val) fetchUsersByTaskType(val);
  else userList.value = [];
});

onMounted(() => {
  fetchRoles();
  fetchTaskTypes();
  fetchCurrentUser();
});

async function createUser() {
  const res = await fetch('http://127.0.0.1:8000/api/user/create', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${props.token}`
    },
    body: JSON.stringify({
      name: userName.value,
      email: userEmail.value,
      password: userPassword.value,
      password_confirmation: userPasswordConfirmation.value,
      role: userRole.value
    })
  });
  const j = await res.json();
  if (!res.ok) return userResultMessage.value = j.error || 'Hiba';
  userResultMessage.value = 'Sikeres hozzáadás';
  userName.value = userEmail.value = userPassword.value = userPasswordConfirmation.value = userRole.value = '';
}

async function createTaskType() {
  const res = await fetch('http://127.0.0.1:8000/api/tasktype/create/new', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${props.token}`
    },
    body: JSON.stringify({
      type_name: typeName.value,
      assignable_role: assignableRole.value
    })
  });
  const j = await res.json();
  if (!res.ok) return taskTypeResultMessage.value = j.error || 'Hiba';
  taskTypeResultMessage.value = 'Sikeres hozzáadás';
  typeName.value = assignableRole.value = '';
  fetchTaskTypes();
}

async function createTask() {
  const res = await fetch('http://127.0.0.1:8000/api/task/create/new', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${props.token}`
    },
    body: JSON.stringify({
      assigner: currentUserId.value,
      worker: worker.value || null,
      task_type: taskTypeId.value,
      description: description.value
    })
  });
  const j = await res.json();
  if (!res.ok) return taskResultMessage.value = j.error || 'Hiba';
  taskResultMessage.value = 'Sikeres hozzáadás';
  worker.value = taskTypeId.value = description.value = '';
  userList.value = [];
}

function backToMain() {
  emit('backToTaskInfo');
}
</script>

<style scoped>
.card-header {
  background-color: #343a40; /* megegyezik a navbar bg-dark-szel */
  color: #fff;
  /* a kék alulvonal eltávolítva */
}
.list-group-item {
  cursor: pointer;
  border: none;
}
.list-group-item.active {
  background-color: #f8f9fa;   /* világos, mint a btn-light */
  color: #212529;               /* text-dark */
  font-weight: bold;
}
.card-body {
  padding: 1.5rem;
}
</style>
