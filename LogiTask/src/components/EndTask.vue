<template>
    <div class="card">
      <div class="card-header">
        End Task - <small>EndTask Component</small>
      </div>
      <div class="card-body">
        <div v-if="isLoading" class="text-center">
          <div class="spinner-border" role="status">
            <span class="visually-hidden">Betöltés...</span>
          </div>
          <p>Betöltés...</p>
        </div>
        <div v-else>
          <p>
            A feladatot sikeresen lezárta, ez <strong>{{ timeTakenText }}</strong> vett igénybe.
          </p>
          <div class="mt-3 text-center">
            <button v-if="!backLoading" class="btn btn-primary" @click="backToTaskInfo">
              Vissza a főoldalra
            </button>
            <div v-else class="progress mt-3">
              <div class="progress-bar progress-bar-striped progress-bar-animated" style="width: 100%"></div>
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
  
  const timeTakenText = computed(() => {
    return props.timeTaken + " másodperc";
  });
  
  const backToTaskInfo = () => {
    backLoading.value = true;
    setTimeout(() => {
      backLoading.value = false;
      emit('backToTaskInfo');
    }, 1000);
  };
  </script>
  
  <style scoped>
  .card {
    margin-top: 20px;
  }
  </style>
  