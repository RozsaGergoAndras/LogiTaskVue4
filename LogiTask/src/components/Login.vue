<script setup>
import { ref, reactive, defineEmits, defineProps } from "vue";

const props = defineProps({
  logoutMessage: {
    type: String,
    default: ''
  }
});
const emit = defineEmits(['login-success']);

const email = ref("");
const password = ref("");
const state = reactive({
  errorMessage: "",
  isLoading: false,
  redirecting: false, // Csak a bejelentkezéshez szükséges
});

// Bejelentkezési függvény
const login = async () => {
  if (!email.value || !password.value) {
    state.errorMessage = "Az email és a jelszó megadása kötelező!";
    return;
  }
  state.isLoading = true;
  state.errorMessage = "";
  try {
    const response = await fetch("http://127.0.0.1:8000/api/login", {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        email: email.value,
        password: password.value
      })
    });
    const data = await response.json();
    if (!response.ok) {
      throw new Error(data.message || "Hibás bejelentkezési adatok!");
    }
    const token = data.token;
    localStorage.setItem("token", token);
    state.redirecting = true;
    // 3 másodperces várakozás után jelzést küldünk a szülő felé
    setTimeout(() => {
      emit('login-success', token);
    }, 1000);
  } catch (error) {
    state.errorMessage = error.message;
  } finally {
    state.isLoading = false;
  }
};
</script>

<template>
  <div class="row justify-content-center">
    <div class="col-md-6 col-lg-4">
      <div class="card shadow">
        <div class="card-header bg-primary text-white">
          <h4 class="mb-0">Bejelentkezés</h4>
        </div>
        <div class="card-body">
          <!-- Ha nincs átirányítás folyamatban -->
          <div v-if="!state.redirecting">
            <div class="mb-3">
              <label for="email" class="form-label">Email</label>
              <input
                id="email"
                type="text"
                class="form-control"
                v-model="email"
                placeholder="Email"
                :disabled="state.isLoading"
              />
            </div>
            <div class="mb-3">
              <label for="password" class="form-label">Jelszó</label>
              <input
                id="password"
                type="password"
                class="form-control"
                v-model="password"
                placeholder="Jelszó"
                :disabled="state.isLoading"
              />
            </div>
            <button
              type="button"
              class="btn btn-primary w-100"
              :disabled="state.isLoading"
              @click="login"
            >
              {{ state.isLoading ? "Bejelentkezés folyamatban..." : "Bejelentkezés" }}
            </button>
            <!-- Logout üzenet megjelenítése a login gomb alatt -->
            <div v-if="logoutMessage" class="alert alert-info mt-3" role="alert">
              {{ logoutMessage }}
            </div>
            <!-- Hibák megjelenítése -->
            <div v-if="state.errorMessage" class="alert alert-danger mt-3" role="alert">
              {{ state.errorMessage }}
            </div>
          </div>
          <!-- Átirányítás esetén megjelenő loading bar (csak a bejelentkezésnél) -->
          <div v-else class="text-center">
            <p>Bejelentkezés sikeres, kérlek várj...</p>
            <div class="progress mt-3">
              <div class="progress-bar progress-bar-striped progress-bar-animated" style="width: 100%"></div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.card {
  margin-top: 20px;
}
</style>
