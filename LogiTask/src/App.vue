<template>
  <div>
    <!-- Navbar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
      <div class="container-fluid">
        <a class="navbar-brand" href="#">LogiTask</a>
        <div class="d-flex align-items-center">
          <!-- Felhasználó üdvözlése, csak bejelentkezett állapotban -->
          <span v-if="isLoggedIn" class="navbar-text me-3">
            <strong>Szia, {{ userName }}</strong>
          </span>
          <!-- Visszaszámláló megjelenítése, ha be van jelentkezve -->
          <span v-if="isLoggedIn" class="navbar-text me-3">
            <strong>Szünet idő:</strong> {{ timeLeft }} perc
          </span>
          <!-- Manager gomb: csak akkor jelenik meg, ha be van jelentkezve és role manager vagy 2 -->
          <button
            v-if="isLoggedIn && (role === 'manager' || role === '2' || role === 2)"
            type="button"
            class="btn btn-warning me-2"
            @click="showAdminPanel"
          >
            Manager
          </button>
          <!-- Statisztika gomb: csak manager role esetén -->
          <button
            v-if="isLoggedIn && (role === 'manager' || role === '2' || role === 2)"
            type="button"
            class="btn btn-info me-2"
            @click="showStatsPanel"
          >
            Statisztika
          </button>
          <!-- Logout gomb -->
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
      <!-- Login nézet -->
      <Login
        v-if="currentView === 'login'"
        :logoutMessage="logoutMessage"
        @login-success="handleLoginSuccess"
        @clear-logout-message="clearLogoutMessage"
      />

      <!-- TaskInfo nézet -->
      <TaskInfo
        v-else-if="currentView === 'taskInfo'"
        :token="token"
        @navigate="handleNavigate"
      />

      <!-- CurrentTask nézet -->
      <CurrentTask
        v-else-if="currentView === 'currentTask'"
        :token="token"
        :taskId="currentTaskId"
        @navigateToEndTask="handleNavigateToEndTask"
      />

      <!-- EndTask nézet -->
      <EndTask
        v-else-if="currentView === 'endTask'"
        :timeTaken="timeTaken"
        @backToTaskInfo="handleBackToTaskInfo"
      />

      <!-- AdminPanel nézet -->
      <AdminPanel
        v-else-if="currentView === 'adminPanel'"
        :token="token"
        @backToTaskInfo="handleBackToTaskInfo"
      />

      <!-- Stats (Statisztika) nézet -->
      <Stats
        v-else-if="currentView === 'stats'"
        :token="token"
        @backToTaskInfo="handleBackToTaskInfo"
      />

      <!-- Kijelentkezés esetén megjelenő progress bar -->
      <div v-if="loggingOut" class="text-center">
        <p>{{ logoutMessage }}</p>
        <div class="progress mt-3">
          <div
            class="progress-bar progress-bar-striped progress-bar-animated"
            role="progressbar"
            style="width: 100%"
          ></div>
        </div>
      </div>
    </div>

    <!-- Bootstrappal stílusos modal a szünet figyelmeztetésére -->
    <div v-if="showBreakModal" class="modal fade show" style="display: block;" tabindex="-1">
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">Szünet szükséges</h5>
            <button type="button" class="btn-close" @click="closeBreakModal"></button>
          </div>
          <div class="modal-body">
            <p>A bejelentkezés óra lejárt. Kérjük, tarts egy rövid szünetet!</p>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn btn-primary" @click="closeBreakModal">
              OK, folytatom
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
import Stats from "./components/Stats.vue";

export default {
  name: "App",
  components: {
    Login,
    TaskInfo,
    CurrentTask,
    EndTask,
    AdminPanel,
    Stats
  },
  data() {
    return {
      token: "",
      role: "",
      userName: "", // Új property a felhasználó nevének tárolására
      logoutMessage: "",
      loggingOut: false,
      currentView: "login", // "login" | "taskInfo" | "currentTask" | "endTask" | "adminPanel" | "stats"
      currentTaskId: null,
      timeTaken: 0,
      // Timer: 60 perc visszaszámláló (egység: perc)
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
            "Authorization": `Bearer ${this.token}`
          }
        });
        if (!response.ok) {
          const data = await response.json();
          throw new Error(data.error || "Hiba történt a felhasználó adatainak lekérésekor!");
        }
        const data = await response.json();
        // Role és név beállítása
        this.role = data.role.toString();
        this.userName = data.name; // A backend neve mezője
      } catch (error) {
        console.error("Error fetching user role:", error);
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
            "Authorization": `Bearer ${this.token}`
          }
        });
        if (!response.ok) {
          const data = await response.json();
          throw new Error(data.error || "Kijelentkezés sikertelen!");
        }
        message = "Sikeres kijelentkezés! Átirányítás folyamatban...";
      } catch (error) {
        message = "Kijelentkezés sikertelen: " + error.message;
      }
      this.logoutMessage = message;
      clearInterval(this.timer);
      setTimeout(() => {
        this.token = "";
        this.role = "";
        this.userName = "";
        this.currentView = "login";
        localStorage.removeItem("token");
        this.loggingOut = false;
      }, 1000);
    },
    handleNavigate(taskId) {
      console.log("App.vue: Received navigate event with taskId:", taskId);
      this.currentTaskId = taskId;
      this.currentView = "currentTask";
    },
    handleNavigateToEndTask({ taskId, timeTaken }) {
      console.log("App.vue: Received navigateToEndTask event with taskId:", taskId, "timeTaken:", timeTaken);
      this.currentTaskId = taskId;
      this.timeTaken = timeTaken;
      this.currentView = "endTask";
    },
    handleBackToTaskInfo() {
      console.log("App.vue: Received backToTaskInfo event");
      this.currentView = "taskInfo";
    },
    showAdminPanel() {
      this.currentView = "adminPanel";
    },
    showStatsPanel() {
      this.currentView = "stats";
    },
    // Timer kezelése: 60 perc visszaszámlálás, 1 percenként csökken
    startTimer() {
      clearInterval(this.timer);
      this.timeLeft = 60; // 60 perc
      this.timer = setInterval(() => {
        if (this.timeLeft > 0) {
          this.timeLeft--;
        } else {
          clearInterval(this.timer);
          this.showBreakModal = true;
        }
      }, 60000); // 60 000 ms = 1 perc
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
