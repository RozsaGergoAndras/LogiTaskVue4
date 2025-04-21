<script setup>
import { ref, reactive, watch, defineEmits, defineProps } from "vue";

const props = defineProps({
  logoutMessage: { type: String, default: "" }
});
const emit = defineEmits(["login-success", "clear-logout-message"]);

const email = ref("");
const password = ref("");
const logoutMessage = ref(props.logoutMessage);

const state = reactive({
  errorMessage: "",
  isLoading: false,
  redirecting: false,
});

// Ha a szülő küld logoutMessage-et, 5 másodperc múlva töröljük
watch(() => props.logoutMessage, (val) => {
  logoutMessage.value = val;
  if (val) {
    setTimeout(() => {
      logoutMessage.value = "";
      emit("clear-logout-message");
    }, 5000);
  }
});

const login = async () => {
  if (!email.value || !password.value) {
    state.errorMessage = "Az email és a jelszó megadása kötelező!";
    setTimeout(() => state.errorMessage = "", 5000);
    return;
  }
  state.isLoading = true;
  state.errorMessage = "";
  try {
    const res = await fetch("http://127.0.0.1:8000/api/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email: email.value, password: password.value })
    });
    const data = await res.json();
    if (!res.ok) throw new Error(data.message || "Hibás bejelentkezési adatok!");
    localStorage.setItem("token", data.token);
    state.redirecting = true;
    setTimeout(() => emit("login-success", data.token), 1000);
  } catch (e) {
    state.errorMessage = e.message;
    setTimeout(() => state.errorMessage = "", 5000);
  } finally {
    state.isLoading = false;
  }
};
</script>

<template>
  <div class="row justify-content-center">
    <div class="col-md-6 col-lg-4">
      <div class="card shadow-sm mt-4">
        <div class="card-header bg-dark text-white">
          <h4 class="mb-0">Bejelentkezés</h4>
        </div>
        <div class="card-body">
          <div v-if="!state.redirecting">
            <div class="mb-3">
              <label for="email" class="form-label text-dark">Email</label>
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
              <label for="password" class="form-label text-dark">Jelszó</label>
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
              class="btn btn-outline-dark w-100"
              :disabled="state.isLoading"
              @click="login"
            >
              {{ state.isLoading ? "Bejelentkezés folyamatban..." : "Bejelentkezés" }}
            </button>
            <div v-if="logoutMessage" class="alert alert-light text-dark mt-3" role="alert">
              {{ logoutMessage }}
            </div>
            <div v-if="state.errorMessage" class="alert alert-danger mt-3" role="alert">
              {{ state.errorMessage }}
            </div>
          </div>
          <div v-else class="text-center">
            <p class="text-dark">Bejelentkezés sikeres, kérlek várj...</p>
            <div class="progress mt-3">
              <div
                class="progress-bar bg-dark progress-bar-striped progress-bar-animated"
                style="width: 100%"
              ></div>
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
/* Input fókusz: sötét szegély, nincs box-shadow */
.form-control:focus {
  border-color: #343a40 !important;
  box-shadow: none !important;
}
/* Alert light háttér sötét szöveggel */
.alert-light {
  background-color: #f8f9fa;
  color: #212529;
}
</style>
