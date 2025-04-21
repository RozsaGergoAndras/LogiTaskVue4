<template>
  <div class="card shadow-sm mt-4">
    <!-- Card header -->
    <div class="card-header bg-dark text-white">
      <h5 class="mb-0">Feladat lezárva</h5>
    </div>

    <!-- Card body -->
    <div class="card-body">
      <!-- Loading -->
      <div v-if="isLoading" class="text-center my-5">
        <div class="spinner-border text-dark" role="status">
          <span class="visually-hidden">Betöltés...</span>
        </div>
        <p class="mt-2">Betöltés...</p>
      </div>

      <!-- Content -->
      <div v-else>
        <p class="time-text">
          A feladat lezárása <strong>{{ timeTakenText }}</strong> vett igénybe.
        </p>
        <div class="mt-4 text-center">
          <!-- ha még nem megy vissza -->
          <button
            v-if="!backLoading"
            class="btn btn-outline-dark px-4"
            @click="backToTaskInfo"
          >
            Vissza a főoldalra
          </button>

          <!-- ha már vissza tölt -->
          <div v-else class="alert alert-dark d-flex justify-content-center align-items-center">
            <div class="spinner-border text-light me-3" role="status">
              <span class="visually-hidden">Visszatérés...</span>
            </div>
            <span>Visszatérés...</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import { defineProps, defineEmits } from 'vue';

const props = defineProps({
  timeTaken: { type: Number, required: true }
});
const emit = defineEmits(['backToTaskInfo']);

const isLoading = ref(false);
const backLoading = ref(false);

const timeTakenText = computed(() => `${props.timeTaken} másodperc`);

function backToTaskInfo() {
  backLoading.value = true;
  // 1 másodperces “visszatérési” delay
  setTimeout(() => {
    backLoading.value = false;
    emit('backToTaskInfo');
  }, 1000);
}
</script>

<style scoped>
.card-header {
  background-color: #343a40;
  color: #fff;
}
.time-text {
  font-size: 1.2rem;
  font-weight: 500;
  text-align: center;
}
.btn-outline-dark {
  border-width: 2px;
}
.card {
  margin-top: 20px;
}
.alert-dark {
  background-color: #444 !important;
  color: #fff !important;
  border-color: #444 !important;
}
</style>
