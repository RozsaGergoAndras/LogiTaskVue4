<template>
  <div>
    <!-- Navbar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark py-3 shadow">
      <div class="container">
        <!-- Brand with logo -->
        <a class="navbar-brand" href="#">
          <img src="/logo.png" alt="LogiTask"style="height: 65px;" />
        </a>

        <!-- Mobile toggler -->
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

        <!-- Links -->
        <div class="collapse navbar-collapse" id="mainNavbar">
          <!-- Centered buttons -->
          <ul class="navbar-nav mx-auto d-flex gap-4">
            <!-- Manager -->
            <li
              class="nav-item"
              v-if="isLoggedIn && (role === '2' || role === 2 || role === 'manager')"
            >
              <button
                class="btn btn-outline-light"
                @click.prevent="currentView = 'adminPanel'"
              >
                Manager
              </button>
            </li>
            <!-- Kiosztott feladatok -->
            <li
              class="nav-item"
              v-if="isLoggedIn && (role === '2' || role === 2 || role === 'manager')"
            >
              <button
                class="btn btn-outline-light"
                @click.prevent="currentView = 'assignedWorks'"
              >
                Kiosztott feladatok
              </button>
            </li>
            <!-- Statisztika -->
            <li
              class="nav-item"
              v-if="isLoggedIn && (role === '2' || role === 2 || role === 'manager')"
            >
              <button
                class="btn btn-outline-light"
                @click.prevent="currentView = 'stats'"
              >
                Statisztika
              </button>
            </li>
            <!-- Előző munkák -->
            <li
              class="nav-item"
              v-if="isLoggedIn && (role === '1' || role === 1)"
            >
              <button
                class="btn btn-outline-light"
                @click.prevent="currentView = 'previousWorks'"
              >
                Előző munkák
              </button>
            </li>
          </ul>
          <!-- Logout button aligned right -->
          <ul class="navbar-nav ms-auto">
            <li class="nav-item">
              <button
                v-if="isLoggedIn"
                class="btn btn-danger fw-bold"
                @click="logout"
              >
                Kijelentkezés
              </button>
            </li>
          </ul>
        </div>
      </div>
    </nav>

    <!-- Main content -->
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

      <div v-if="loggingOut" class="text-center mt-4">
        <p>{{ logoutMessage }}</p>
        <div class="progress mt-2">
          <div
            class="progress-bar progress-bar-striped progress-bar-animated"
            role="progressbar"
            style="width: 100%;"
          ></div>
        </div>
      </div>
    </div>

    <!-- Break modal -->
    <div
      v-if="showBreakModal"
      class="modal fade show"
      style="display: block;"
      tabindex="-1"
    >
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
            <button
              type="button"
              class="btn btn-primary"
              @click="closeBreakModal"
            >
              OK
            </button>
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
      this.currentView =
        (this.role === "2" || this.role === "manager")
          ? "adminPanel"
          : "taskInfo";
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
        if (this.timeLeft > 0) {
          this.timeLeft--;
        } else {
          clearInterval(this.timer);
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
/* Increase navbar height */
.navbar {
  min-height: 70px;
}
/* Make buttons larger */
.btn {
  font-size: 1.1rem;
  padding: 0.5rem 1.5rem;
}
/* Outline-light buttons on dark background */
.btn-outline-light {
  border-width: 2px;
}
/* Dark navbar text color */
.navbar-dark .navbar-brand img {
  display: inline-block;
  vertical-align: middle;
}
</style>
