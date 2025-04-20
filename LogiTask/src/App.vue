<template>
  <div>
    <!-- Navbar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
      <div class="container-fluid">
        <a class="navbar-brand" href="#">LogiTask</a>
        <div class="d-flex align-items-center">
          <!-- Felhasználó üdvözlése -->
          <span v-if="isLoggedIn" class="navbar-text me-3">
            <strong>Szia, {{ userName }}</strong>
          </span>
          <!-- Visszaszámláló -->
          <span v-if="isLoggedIn" class="navbar-text me-3">
            <strong>Szünet idő:</strong> {{ timeLeft }} perc
          </span>
          <!-- Előző munkák: csak worker (role=1) -->
          <button
            v-if="isLoggedIn && (role === '1' || role === 1)"
            type="button"
            class="btn btn-secondary me-2"
            @click="currentView = 'previousWorks'"
          >
            Előző munkák
          </button>
          <!-- Manager felület: csak manager (role=2) -->
          <button
            v-if="isLoggedIn && (role === '2' || role === 'manager')"
            type="button"
            class="btn btn-warning me-2"
            @click="currentView = 'adminPanel'"
          >
            Manager
          </button>
          <!-- Assigned: csak manager -->
          <button
            v-if="isLoggedIn && (role === '2' || role === 'manager')"
            type="button"
            class="btn btn-secondary me-2"
            @click="currentView = 'assignedWorks'"
          >
            Assigned
          </button>
          <!-- Statisztika: csak manager -->
          <button
            v-if="isLoggedIn && (role === '2' || role === 'manager')"
            type="button"
            class="btn btn-info me-2"
            @click="currentView = 'stats'"
          >
            Statisztika
          </button>
          <!-- Logout -->
          <button
            v-if="isLoggedIn"
            type="button"
            class="btn btn-danger"
            @click="logout"
          >
            Logout
          </button>
        </div>
      </div>
    </nav>

    <!-- Fő tartalom -->
    <div class="container my-5">
      <Login
        v-if="currentView === 'login'"
        :logoutMessage="logoutMessage"
        @login-success="handleLoginSuccess"
        @clear-logout-message="clearLogoutMessage"
      />

      <TaskInfo
        v-else-if="currentView === 'taskInfo'"
        :token="token"
        @navigate="handleNavigate"
      />

      <CurrentTask
        v-else-if="currentView === 'currentTask'"
        :token="token"
        :taskId="currentTaskId"
        @navigateToEndTask="handleNavigateToEndTask"
      />

      <EndTask
        v-else-if="currentView === 'endTask'"
        :timeTaken="timeTaken"
        @backToTaskInfo="handleBackToTaskInfo"
      />

      <AdminPanel
        v-else-if="currentView === 'adminPanel'"
        :token="token"
        @backToTaskInfo="handleBackToTaskInfo"
      />

      <ListAssignedWorks
        v-else-if="currentView === 'assignedWorks'"
        :token="token"
      />

      <ListPreviousWorks
        v-else-if="currentView === 'previousWorks'"
        :token="token"
      />

      <Stats
        v-else-if="currentView === 'stats'"
        :token="token"
        @backToTaskInfo="handleBackToTaskInfo"
      />

      <!-- Logout progress -->
      <div v-if="loggingOut" class="text-center">
        <p>{{ logoutMessage }}</p>
        <div class="progress mt-3">
          <div class="progress-bar progress-bar-striped progress-bar-animated" role="progressbar" style="width: 100%"></div>
        </div>
      </div>
    </div>

    <!-- Szünet modal -->
    <div v-if="showBreakModal" class="modal fade show" style="display: block;" tabindex="-1">
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">Szünet szükséges</h5>
            <button type="button" class="btn-close" @click="closeBreakModal"></button>
          </div>
          <div class="modal-body">
            <p>A szünetidő lejárt. Kérjük, tarts egy rövid szünetet!</p>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn btn-primary" @click="closeBreakModal">OK</button>
          </div>
        </div>
      </div>
    </div>
    <div v-if="showBreakModal" class="modal-backdrop fade show"></div>
  </div>
</template>

<script>
import Login from "./components/Login.vue";
import TaskInfo from "./components/TaskInfo.vue";
import CurrentTask from "./components/CurrentTask.vue";
import EndTask from "./components/EndTask.vue";
import AdminPanel from "./components/AdminPanel.vue";
import ListAssignedWorks from "./components/ListAssignedWorks.vue";
import ListPreviousWorks from "./components/ListPreviousWorks.vue";
import Stats from "./components/Stats.vue";

export default {
  name: "App",
  components: {
    Login,
    TaskInfo,
    CurrentTask,
    EndTask,
    AdminPanel,
    ListAssignedWorks,
    ListPreviousWorks,
    Stats
  },
  data() {
    return {
      token: "",
      role: "",
      userName: "",
      logoutMessage: "",
      loggingOut: false,
      currentView: "login",
      currentTaskId: null,
      timeTaken: 0,
      timeLeft: 60,
      timer: null,
      showBreakModal: false
    };
  },
  computed: {
    isLoggedIn() {
      return this.token !== "";
    }
  },
  methods: {
    async handleLoginSuccess(token) {
      this.token = token;
      try {
        const response = await fetch("http://127.0.0.1:8000/api/user", {
          method: "GET",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${this.token}`
          }
        });
        if (!response.ok) {
          const data = await response.json();
          throw new Error(data.error || "Hiba a felhasználó adatainak lekérésekor");
        }
        const data = await response.json();
        this.role = data.role.toString();
        this.userName = data.name;
      } catch (error) {
        console.error(error);
        this.role = "";
      }
      this.currentView = "taskInfo";
      this.logoutMessage = "";
      this.startTimer();
    },
    clearLogoutMessage() {
      this.logoutMessage = "";
    },
    async logout() {
      if (!this.token) return;
      this.loggingOut = true;
      let message = "";
      try {
        const response = await fetch("http://127.0.0.1:8000/api/logout", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${this.token}`
          }
        });
        if (!response.ok) {
          const data = await response.json();
          throw new Error(data.error || "Kijelentkezés sikertelen");
        }
        message = "Sikeres kijelentkezés!";
      } catch (e) {
        message = "Kijelentkezés sikertelen: " + e.message;
      }
      this.logoutMessage = message;
      clearInterval(this.timer);
      setTimeout(() => {
        this.token = "";
        this.role = "";
        this.userName = "";
        this.currentView = "login";
        this.loggingOut = false;
      }, 1000);
    },
    handleNavigate(taskId) {
      this.currentTaskId = taskId;
      this.currentView = "currentTask";
    },
    handleNavigateToEndTask({ taskId, timeTaken }) {
      this.currentTaskId = taskId;
      this.timeTaken = timeTaken;
      this.currentView = "endTask";
    },
    handleBackToTaskInfo() {
      this.currentView = "taskInfo";
    },
    startTimer() {
      clearInterval(this.timer);
      this.timeLeft = 60;
      this.timer = setInterval(() => {
        if (this.timeLeft > 0) this.timeLeft--; else { clearInterval(this.timer); this.showBreakModal = true; }
      }, 60000);
    },
    closeBreakModal() {
      this.showBreakModal = false;
      this.startTimer();
    }
  },
  beforeUnmount() {
    clearInterval(this.timer);
  }
};
</script>

<style scoped>
.navbar-text {
  font-size: 1.2rem;
  color: #fff;
}
</style>