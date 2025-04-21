<template>
  <div>
    <!-- Navbar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark py-2 shadow">
      <div class="container">
        <!-- Logo, kattintásra TaskInfo -->
        <a class="navbar-brand" href="#" @click.prevent="showTaskInfo">
          <img src="/logo.png" alt="LogiTask" style="height: 60px;" />
        </a>
        <button
          class="navbar-toggler"
          type="button"
          data-bs-toggle="collapse"
          data-bs-target="#mainNavbar"
          aria-controls="mainNavbar"
          aria-expanded="false"
          aria-label="Toggle navigation"
        >
          <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="mainNavbar">
          <ul class="navbar-nav mx-auto d-flex gap-4">
            <li class="nav-item" v-if="isLoggedIn && isManager">
              <button
                class="btn"
                :class="currentView==='adminPanel' ? 'btn-light text-dark' : 'btn-outline-light'"
                @click.prevent="currentView='adminPanel'"
              >Manager</button>
            </li>
            <li class="nav-item" v-if="isLoggedIn">
              <button
                class="btn"
                :class="currentView==='taskInfo' ? 'btn-light text-dark' : 'btn-outline-light'"
                @click.prevent="currentView='taskInfo'"
              >Feladataim</button>
            </li>
            <li class="nav-item" v-if="isLoggedIn && isManager">
              <button
                class="btn"
                :class="currentView==='assignedWorks' ? 'btn-light text-dark' : 'btn-outline-light'"
                @click.prevent="currentView='assignedWorks'"
              >Kiosztott feladatok</button>
            </li>
            <li class="nav-item" v-if="isLoggedIn && isManager">
              <button
                class="btn"
                :class="currentView==='stats' ? 'btn-light text-dark' : 'btn-outline-light'"
                @click.prevent="currentView='stats'"
              >Statisztika</button>
            </li>
            <li class="nav-item" v-if="isLoggedIn && isWorker">
              <button
                class="btn"
                :class="currentView==='previousWorks' ? 'btn-light text-dark' : 'btn-outline-light'"
                @click.prevent="currentView='previousWorks'"
              >Előző munkák</button>
            </li>
          </ul>
          <ul class="navbar-nav ms-auto">
            <li class="nav-item">
              <button
                v-if="isLoggedIn"
                class="btn btn-danger fw-bold"
                @click="logout"
              >Kijelentkezés</button>
            </li>
          </ul>
        </div>
      </div>
    </nav>

    <!-- Main content -->
    <div class="container my-5">
      <Login
        v-if="currentView==='login'"
        :logoutMessage="logoutMessage"
        @login-success="handleLoginSuccess"
        @clear-logout-message="logoutMessage=''"/>
      <TaskInfo
        v-else-if="currentView==='taskInfo'"
        :token="token"
        @navigate="handleNavigate"/>
      <CurrentTask
        v-else-if="currentView==='currentTask'"
        :token="token"
        :taskId="currentTaskId"
        @navigateToEndTask="handleNavigateToEndTask"/>
      <EndTask
        v-else-if="currentView==='endTask'"
        :timeTaken="timeTaken"
        @backToTaskInfo="handleBackToTaskInfo"/>
      <AdminPanel
        v-else-if="currentView==='adminPanel'"
        :token="token"
        @backToTaskInfo="handleBackToTaskInfo"/>
      <ListAssignedWorks
        v-else-if="currentView==='assignedWorks'"
        :token="token"/>
      <ListPreviousWorks
        v-else-if="currentView==='previousWorks'"
        :token="token"/>
      <Stats
        v-else-if="currentView==='stats'"
        :token="token"
        @backToTaskInfo="handleBackToTaskInfo"/>
    </div>

    <!-- Logout modal -->
    <div v-if="loggingOut" class="modal fade show" style="display: block;" tabindex="-1">
      <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content bg-dark text-white">
          <div class="modal-body text-center">
            <p>{{ logoutMessage }}</p>
            <div class="progress mt-3">
              <div
                class="progress-bar progress-bar-striped progress-bar-animated bg-dark"
                role="progressbar"
                style="width: 100%;"
              ></div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div v-if="loggingOut" class="modal-backdrop fade show"></div>

    <!-- Break modal -->
    <div
      v-if="showBreakModal"
      class="modal fade show"
      style="display: block;"
      tabindex="-1"
    >
      <div class="modal-dialog">
        <div class="modal-content bg-dark text-white">
          <div class="modal-header">
            <h5 class="modal-title">Szünet szükséges</h5>
            <button type="button" class="btn-close" @click="closeBreakModal"></button>
          </div>
          <div class="modal-body">
            <p>A szünetidő lejárt. Kérjük, tarts egy rövid szünetet!</p>
          </div>
          <div class="modal-footer">
            <button class="btn btn-light" @click="closeBreakModal">OK</button>
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
      logoutMessage: "",
      loggingOut: false,
      currentView: "login",
      currentTaskId: null,
      timeTaken: 0,
      timer: null,
      showBreakModal: false,
      timeLeft: 60
    };
  },
  computed: {
    isLoggedIn() { return this.token !== ""; },
    isManager() { return this.role === "2" || this.role === "manager"; },
    isWorker()  { return this.role === "1"; }
  },
  methods: {
    showTaskInfo() { this.currentView = "taskInfo"; },
    async handleLoginSuccess(token) {
      this.token = token;
      try {
        const res = await fetch("http://127.0.0.1:8000/api/user", {
          headers: { Authorization: `Bearer ${token}` }
        });
        const data = await res.json();
        this.role = data.role.toString();
      } catch {
        this.role = "";
      }
      this.currentView = this.isManager ? "adminPanel" : "taskInfo";
      this.startTimer();
    },
    async logout() {
      this.loggingOut = true;
      let msg;
      try {
        const res = await fetch("http://127.0.0.1:8000/api/logout", {
          method: "POST",
          headers: { Authorization: `Bearer ${this.token}` }
        });
        if (!res.ok) throw new Error("Kijelentkezés sikertelen");
        msg = "Sikeres kijelentkezés!";
      } catch (e) {
        msg = e.message;
      }
      this.logoutMessage = msg;
      setTimeout(() => this.logoutMessage = "", 5000);
      clearInterval(this.timer);
      setTimeout(() => {
        this.token = "";
        this.role = "";
        this.currentView = "login";
        this.loggingOut = false;
      }, 1000);
    },
    handleNavigate(id) {
      this.currentTaskId = id;
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
      console.log(`Hátralévő idő a szünetig: ${this.timeLeft} perc`);
      this.timer = setInterval(() => {
        if (this.timeLeft > 0) {
          this.timeLeft--;
          console.log(`Hátralévő idő a szünetig: ${this.timeLeft} perc`);
        } else {
          clearInterval(this.timer);
          console.log("Szünet ideje!");
          this.showBreakModal = true;
        }
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
.navbar {
  min-height: 60px;
}
.btn {
  font-size: 1.1rem;
  padding: 0.4rem 1.2rem;
}
</style>
