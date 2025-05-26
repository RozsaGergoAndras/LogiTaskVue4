<template>
  <div>
    <!-- Navbar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark py-2 shadow">
      <div class="container">
        <!-- Logo, kattintásra TaskInfo -->
        <a class="navbar-brand" href="#" @click.prevent="showTaskInfo">
          <img src="/logo.png" alt="LogiTask" style="max-height: 60px;" />
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
            <li class="nav-item" v-if="isLoggedIn">
              <button
                class="btn btn-outline-light"
                @click.prevent="openPasswordModal"
              >Jelszó átállítása</button>
            </li>
          </ul>
          <ul class="nav">
            <li class="nav-item">
              <button
                v-if="isLoggedIn"
                class="btn btn-danger fw-bold btn-logout"
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
      />
      <TaskInfo
        v-else-if="currentView==='taskInfo'"
        :token="token"
        @navigate="handleNavigate"
      />
      <CurrentTask
        v-else-if="currentView==='currentTask'"
        :token="token"
        :taskId="currentTaskId"
        @navigateToEndTask="handleNavigateToEndTask"
      />
      <EndTask
        v-else-if="currentView==='endTask'"
        :timeTaken="timeTaken"
        @backToTaskInfo="handleBackToTaskInfo"
      />
      <AdminPanel
        v-else-if="currentView==='adminPanel'"
        :token="token"
        @backToTaskInfo="handleBackToTaskInfo"
      />
      <ListAssignedWorks
        v-else-if="currentView==='assignedWorks'"
        :token="token"
      />
      <ListPreviousWorks
        v-else-if="currentView==='previousWorks'"
        :token="token"
      />
      <Stats
        v-else-if="currentView==='stats'"
        :token="token"
        @backToTaskInfo="handleBackToTaskInfo"
      />
    </div>

    <!-- Password Change Modal -->
    <div v-if="showPasswordModal" class="modal fade show" style="display: block;" tabindex="-1">
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header bg-dark text-white">
            <h5 class="modal-title">Jelszó megváltoztatása</h5>
            <button class="btn-close btn-close-white" @click="closePasswordModal"></button>
          </div>
          <div class="modal-body">
            <!-- Input fields -->
            <div v-if="!confirming && !passwordMessage">
              <div class="mb-3">
                <label class="form-label">Jelenlegi jelszó</label>
                <input type="password" v-model="pw.current" class="form-control" />
              </div>
              <div class="mb-3">
                <label class="form-label">Új jelszó</label>
                <input type="password" v-model="pw.new" class="form-control" />
              </div>
              <div class="mb-3">
                <label class="form-label">Új jelszó megerősítése</label>
                <input type="password" v-model="pw.confirm" class="form-control" />
              </div>
              <div v-if="passwordError" class="alert alert-danger">{{ passwordError }}</div>
            </div>
            <!-- Confirmation -->
            <div v-else-if="confirming && !passwordMessage">
              <p>Biztosan meg szeretnéd változtatni a jelszavad?</p>
            </div>
            <!-- Result message -->
            <div v-if="passwordMessage" class="alert" :class="passwordMessageType">
              {{ passwordMessage }}
            </div>
          </div>
          <div class="modal-footer">
            <!-- Buttons -->
            <button
              v-if="!processing && !confirming && !passwordMessage"
              class="btn btn-outline-dark"
              @click="cancelPassword"
            >Mégse</button>
            <button
              v-if="!processing && !confirming && !passwordMessage"
              class="btn btn-dark"
              @click="confirming = true"
            >Megváltoztatás</button>
            <button
              v-if="!processing && confirming && !passwordMessage"
              class="btn btn-outline-dark"
              @click="confirming = false"
            >Nem</button>
            <button
              v-if="!processing && confirming && !passwordMessage"
              class="btn btn-dark"
              @click="changePassword"
            >Igen</button>
            <!-- Processing spinner in dark alert -->
            <div v-if="processing" class="alert alert-dark d-flex align-items-center">
              <div class="spinner-border text-white me-2" role="status"></div>
              <span class="text-white">Feldolgozás...</span>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div v-if="showPasswordModal" class="modal-backdrop fade show"></div>
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
      timeLeft: 60,

      // Password change modal state
      showPasswordModal: false,
      pw: { current: "", new: "", confirm: "" },
      passwordError: "",
      passwordMessage: "",
      passwordMessageType: "alert-success",
      confirming: false,
      processing: false
    };
  },
  computed: {
    isLoggedIn() { return this.token !== ""; },
    isManager()  { return this.role === "2" || this.role === "manager"; },
    isWorker()   { return this.role === "1"; }
  },
  methods: {
    showTaskInfo() {
      this.currentView = "taskInfo";
    },
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
      try {
        const res = await fetch("http://127.0.0.1:8000/api/logout", {
          method: "POST",
          headers: { Authorization: `Bearer ${this.token}` }
        });
        this.logoutMessage = res.ok ? "Sikeres kijelentkezés!" : "Kijelentkezés sikertelen";
      } catch (e) {
        this.logoutMessage = e.message;
      }
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
    },

    // Password modal methods
    openPasswordModal() {
      this.showPasswordModal = true;
      this.pw = { current: "", new: "", confirm: "" };
      this.passwordError = "";
      this.passwordMessage = "";
      this.confirming = false;
    },
    closePasswordModal() {
      this.showPasswordModal = false;
    },
    cancelPassword() {
      this.confirming = false;
      this.passwordError = "";
      this.closePasswordModal();
    },
    async changePassword() {
      if (!this.pw.current || !this.pw.new) {
        this.passwordError = "Mindkét mező kitöltése kötelező!";
        return;
      }
      if (this.pw.new !== this.pw.confirm) {
        this.passwordError = "Az új jelszavak nem egyeznek!";
        return;
      }
      this.passwordError = "";
      this.processing = true;
      console.log("Password change payload:", {
        current_password: this.pw.current,
        password: this.pw.new,
        password_confirmation: this.pw.new
      });
      try {
        const res = await fetch("http://127.0.0.1:8000/api/user/password", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            "Authorization": `Bearer ${this.token}`
          },
          body: JSON.stringify({
            current_password: this.pw.current,
            password: this.pw.new,
            password_confirmation: this.pw.new
          })
        });
        const json = await res.json();
        console.log("Password change response:", json);
        if (!res.ok) throw new Error(json.error || "Hiba történt!");
        this.passwordMessage = "A jelszó sikeresen megváltozott.";
        this.passwordMessageType = "alert-success";
        this.confirming = false;
        setTimeout(this.closePasswordModal, 3000);
      } catch (e) {
        this.passwordMessage = e.message;
        this.passwordMessageType = "alert-danger";
      } finally {
        this.processing = false;
      }
    }
  },
  beforeUnmount() {
    clearInterval(this.timer);
  }
};
</script>

<style scoped>
.navbar { min-height: 60px; }
.btn { font-size: 1.1rem; padding: 0.4rem 1.2rem; }
.form-control:focus { border-color: #343a40 !important; box-shadow: none !important; }
.alert-dark { background-color: #343a40; color: #fff; border: none; }
.btn-outline-dark { border-width: 2px; }
.mb-3 label { font-weight: 600; }

@media (max-width: 1399px) {
  /* 1300px alatt legyen bal padding */
  .btn-logout {
    margin-left: 1rem;
  }
}

@media (max-width: 991px) {
  /* 900px alatt már csak felső padding, a bal paddingot visszaállítjuk */
  .btn-logout {
    margin-left: 0 !important;
    margin-top: 1.4rem !important;
  }
}
</style>
