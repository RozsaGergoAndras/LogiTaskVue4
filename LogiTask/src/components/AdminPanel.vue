<template>
  <div class="card">
    <div class="card-header">
      Task Admin Panel
    </div>
    <div class="card-body">
      <div class="row">
        <!-- Bal oldali menüsáv -->
        <div class="col-md-3">
          <div class="list-group">
            <button
              type="button"
              class="list-group-item list-group-item-action"
              :class="{ active: selectedMenu === 'user' }"
              @click="selectedMenu = 'user'"
            >
              Felhasználó hozzáadása
            </button>
            <button
              type="button"
              class="list-group-item list-group-item-action"
              :class="{ active: selectedMenu === 'taskType' }"
              @click="selectedMenu = 'taskType'"
            >
              Feladat típus hozzáadása
            </button>
            <button
              type="button"
              class="list-group-item list-group-item-action"
              :class="{ active: selectedMenu === 'task' }"
              @click="selectedMenu = 'task'"
            >
              Feladat létrehozása
            </button>
          </div>
        </div>
        <!-- Jobb oldali tartalom -->
        <div class="col-md-9">
          <!-- Felhasználó hozzáadása -->
          <div v-if="selectedMenu === 'user'">
            <h5>Felhasználó hozzáadása</h5>
            <form @submit.prevent="createUser">
              <div class="mb-3">
                <label for="user_name" class="form-label">Név</label>
                <input v-model="userName" type="text" id="user_name" class="form-control" placeholder="Béla" required />
              </div>
              <div class="mb-3">
                <label for="user_email" class="form-label">Email</label>
                <input v-model="userEmail" type="email" id="user_email" class="form-control" placeholder="test2@example.com" required />
              </div>
              <div class="mb-3">
                <label for="user_password" class="form-label">Jelszó</label>
                <input v-model="userPassword" type="password" id="user_password" class="form-control" placeholder="password" required />
              </div>
              <div class="mb-3">
                <label for="user_password_confirmation" class="form-label">Jelszó megerősítése</label>
                <input v-model="userPasswordConfirmation" type="password" id="user_password_confirmation" class="form-control" placeholder="password" required />
              </div>
              <div class="mb-3">
                <label for="user_role" class="form-label">Szerep</label>
                <select v-model="userRole" id="user_role" class="form-select" required>
                  <option disabled value="">Válassz szerepet</option>
                  <option v-for="roleItem in roles" :key="roleItem.id" :value="roleItem.id">
                    {{ roleItem.role_name }}
                  </option>
                </select>
              </div>
              <button type="submit" class="btn btn-primary">Felhasználó hozzáadása</button>
            </form>
            <div class="mt-3">
              <p v-if="userResultMessage">{{ userResultMessage }}</p>
            </div>
          </div>

          <!-- Feladat típus hozzáadása -->
          <div v-else-if="selectedMenu === 'taskType'">
            <h5>Feladat típus hozzáadása</h5>
            <form @submit.prevent="createTaskType">
              <div class="mb-3">
                <label for="type_name" class="form-label">Típus név</label>
                <input v-model="typeName" type="text" id="type_name" class="form-control" placeholder="Pl. Javítás" required />
              </div>
              <div class="mb-3">
                <label for="assignable_role" class="form-label">Hozzárendelhető szerep</label>
                <select v-model="assignableRole" id="assignable_role" class="form-select" required>
                  <option disabled value="">Válassz szerepet</option>
                  <option v-for="roleItem in roles" :key="roleItem.id" :value="roleItem.id">
                    {{ roleItem.role_name }}
                  </option>
                </select>
              </div>
              <button type="submit" class="btn btn-primary">Feladat típus hozzáadása</button>
            </form>
            <div class="mt-3">
              <p v-if="taskTypeResultMessage">{{ taskTypeResultMessage }}</p>
            </div>
          </div>

          <!-- Feladat létrehozása -->
          <div v-else-if="selectedMenu === 'task'">
            <h5>Feladat létrehozása</h5>
            <form @submit.prevent="createTask">
              <!-- Feladatszervező: a bejelentkezett user ID-je -->
              <p class="mb-3"><strong>Feladatszervező:</strong> {{ currentUserId }}</p>

              <!-- Végrehajtó legördülő -->
              <div class="mb-3">
                <label for="worker" class="form-label">Végrehajtó</label>
                <select v-model="worker" id="worker" class="form-select">
                  <option disabled value="">Válassz végrehajtót</option>
                  <option v-for="user in userList" :key="user.id" :value="user.id">
                    {{ user.name }}
                  </option>
                </select>
              </div>

              <!-- Feladat típus legördülő -->
              <div class="mb-3">
                <label for="task_type" class="form-label">Feladat típus</label>
                <select v-model="taskTypeId" id="task_type" class="form-select" required>
                  <option disabled value="">Válassz típus</option>
                  <option v-for="type in taskTypes" :key="type.id" :value="type.id">
                    {{ type.type_name }}
                  </option>
                </select>
              </div>

              <div class="mb-3">
                <label for="description" class="form-label">Leírás (description)</label>
                <input v-model="description" type="text" id="description" class="form-control" placeholder="Pl. Javítás szükséges" required />
              </div>
              <button type="submit" class="btn btn-primary">Feladat létrehozása</button>
            </form>
            <div class="mt-3">
              <p v-if="taskResultMessage">{{ taskResultMessage }}</p>
            </div>
          </div>
        </div>
      </div>

      <!-- Vissza a főoldalra gomb -->
      <div class="mt-3 text-center">
        <button class="btn btn-secondary" @click="backToMain">Vissza a főoldalra</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';
import { defineProps, defineEmits } from 'vue';

const props = defineProps({ token: { type: String, required: true } });
const emit = defineEmits(['backToTaskInfo']);

// Állapotok
const selectedMenu = ref('user');

// Felhasználó hozzáadása
const userName = ref('');
const userEmail = ref('');
const userPassword = ref('');
const userPasswordConfirmation = ref('');
const userRole = ref('');
const userResultMessage = ref('');

// Feladat típus létrehozása
const typeName = ref('');
const assignableRole = ref('');
const taskTypeResultMessage = ref('');

// Feladat létrehozása
const worker = ref('');
const taskTypeId = ref('');
const description = ref('');
const taskResultMessage = ref('');

// Lekért adatok
const roles = ref([]);
const userList = ref([]);
const taskTypes = ref([]);
const currentUserId = ref('');

// Fetch függvények
async function fetchRoles() {
  const res = await fetch('http://127.0.0.1:8000/api/roles', { headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${props.token}` } });
  const data = await res.json(); roles.value = Array.isArray(data.roles) ? data.roles : data;
}
async function fetchTaskTypes() {
  const res = await fetch('http://127.0.0.1:8000/api/tasktype', { headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${props.token}` } });
  const d = await res.json(); taskTypes.value = d.data || [];
}
async function fetchCurrentUser() {
  const res = await fetch('http://127.0.0.1:8000/api/user', { headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${props.token}` } });
  const d = await res.json(); if (res.ok && d.id) currentUserId.value = d.id;
}
async function fetchUsersByTaskType(val) {
  console.log('fetchUsers POST body:', { task_type_id: val });
  const res = await fetch('http://127.0.0.1:8000/api/users', { method: 'POST', headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${props.token}` }, body: JSON.stringify({ task_type_id: val }) });
  const d = await res.json(); console.log('fetchUsers response:', d); userList.value = Array.isArray(d) ? d : d.users || [];
}

// Watch
watch(taskTypeId, newVal => { if (newVal) fetchUsersByTaskType(newVal); else userList.value = []; });

// Init
onMounted(() => { fetchRoles(); fetchTaskTypes(); fetchCurrentUser(); });

// Műveletek
async function createUser() {
  const body = { name: userName.value, email: userEmail.value, password: userPassword.value, password_confirmation: userPasswordConfirmation.value, role: userRole.value };
  console.log('createUser body', body);
  const res = await fetch('http://127.0.0.1:8000/api/user/create', { method: 'POST', headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${props.token}` }, body: JSON.stringify(body) });
  const d = await res.json(); console.log('createUser res', d);
  if (!res.ok) { userResultMessage.value = 'Hiba: ' + (d.error || ''); return; }
  userResultMessage.value = 'Felhasználó sikeresen hozzáadva!';
  userName.value = userEmail.value = userPassword.value = userPasswordConfirmation.value = userRole.value = '';
}
async function createTaskType() {
  const body = { type_name: typeName.value, assignable_role: assignableRole.value };
  console.log('createTaskType body', body);
  const res = await fetch('http://127.0.0.1:8000/api/tasktype/create/new', { method: 'POST', headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${props.token}` }, body: JSON.stringify(body) });
  const d = await res.json(); console.log('createTaskType res', d);
  if (!res.ok) { taskTypeResultMessage.value = 'Hiba: ' + (d.error || ''); return; }
  taskTypeResultMessage.value = 'Feladat típus sikeresen hozzáadva!';
  typeName.value = assignableRole.value = '';
  fetchTaskTypes();
}
async function createTask() {
  const body = { assigner: currentUserId.value, worker: worker.value || null, task_type: taskTypeId.value, description: description.value };
  console.log('createTask body', body);
  const res = await fetch('http://127.0.0.1:8000/api/task/create/new', { method: 'POST', headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${props.token}` }, body: JSON.stringify(body) });
  const d = await res.json(); console.log('createTask res', d);
  if (!res.ok) { taskResultMessage.value = 'Hiba: ' + (d.error || ''); return; }
  taskResultMessage.value = 'Feladat sikeresen létrehozva!';
  worker.value = taskTypeId.value = description.value = '';
  userList.value = [];
}

function backToMain() { emit('backToTaskInfo'); }
</script>

<style scoped>
.card { margin-top: 20px; }
.list-group-item { cursor: pointer; }
.list-group-item.active { background-color: #0d6efd; border-color: #0d6efd; color: #fff; }
</style>
