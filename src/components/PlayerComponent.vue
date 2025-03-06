<template>
  <div class="players-list">
    <span>Плеер:</span>
    <select v-model="selectedPlayerInternal" class="custom-select">
      <option v-for="(player, index) in playersInternal" :key="index" :value="player">
        {{ player.translate.toUpperCase() }}
      </option>
    </select>
  </div>

  <!-- Контейнер плеера -->
  <div :class="{'theater-player': theaterMode}" class="player-container">
    <div :class="theaterMode ? 'theater-iframe-container' : 'iframe-container'">
      <iframe
        v-show="!iframeLoading"
        :src="selectedPlayerInternal?.iframe"
        frameborder="0"
        @load="onIframeLoad"
        allowfullscreen
        class="responsive-iframe"
      ></iframe>
    </div>
    <SpinnerLoading v-if="iframeLoading" />
  </div>

  <!-- Кнопка закрытия в театральном режиме -->
  <button
    v-if="theaterMode"
    @click="toggleTheaterMode"
    class="close-theater-btn"
    :class="{'visible': closeButtonVisible}"
    @mousemove="showCloseButton"
  >
    ✖
  </button>

  <!-- Кнопка управления (скрыта на мобилках <600px) -->
  <div v-if="!isMobile" class="controls">
    <button @click="toggleTheaterMode" class="theater-mode-btn">
      {{ theaterMode ? 'Выйти из театрального режима' : 'Включить театральный режим' }}
    </button>
  </div>
</template>

<script setup>
import axios from 'axios';
import { ref, onMounted, onBeforeUnmount, computed } from 'vue';
import SpinnerLoading from '@/components/SpinnerLoading.vue';

const props = defineProps({
  movieId: String,
  selectedPlayer: Object
});

const emit = defineEmits(['update:selectedPlayer']);

const playersInternal = ref([]);
const selectedPlayerInternal = ref(props.selectedPlayer);
const iframeLoading = ref(true);
const theaterMode = ref(false);
const closeButtonVisible = ref(false);
const apiUrl = import.meta.env.VITE_APP_API_URL;

// Определяем, мобильное ли устройство
const isMobile = computed(() => window.innerWidth < 600);

const fetchPlayers = async () => {
  try {
    const response = await axios.post(
      `${apiUrl}/cache`,
      new URLSearchParams({
        kinopoisk: props.movieId,
        type: 'movie'
      }),
      { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
    );
    playersInternal.value = Object.values(response.data);
    if (playersInternal.value.length > 0) {
      selectedPlayerInternal.value = playersInternal.value[0];
      emit('update:selectedPlayer', selectedPlayerInternal.value);
    }
    } catch (error) {
      if (error.response && error.response.status === 403) {
        errorMessage.value = 'Упс, у нас это недоступно =(';
      } else {
        errorMessage.value = 'Произошла ошибка при загрузке данных. Пожалуйста, попробуйте позже.';
      }
      console.error('Ошибка при загрузке плееров:', error);
    } finally {
    }
};

const onIframeLoad = () => {
  iframeLoading.value = false;
};

const toggleTheaterMode = () => {
  theaterMode.value = !theaterMode.value;
  if (theaterMode.value) {
    closeButtonVisible.value = true;
    setTimeout(() => {
      closeButtonVisible.value = false;
    }, 1500);
  }
};

const showCloseButton = (event) => {
  if (event.clientY < 50) {
    closeButtonVisible.value = true;
  } else {
    closeButtonVisible.value = false;
  }
};

const onKeyDown = (event) => {
  if (event.key === 'Escape' && theaterMode.value) {
    theaterMode.value = false;
  }
};

onMounted(() => {
  fetchPlayers();
  window.addEventListener('keydown', onKeyDown);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeyDown);
});
</script>

<style scoped>
.custom-select {
  padding: 8px 16px;
  border: 1px solid #444;
  background-color: #1e1e1e;
  color: #fff;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s, border-color 0.3s;
  width: 100%;
}

.custom-select:hover {
  border-color: #666;
}

.custom-select:focus {
  border-color: #558839;
}

.players-list {
  width: 100%;
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  gap: 10px;
}

/* Стили для обычного режима плеера */
.player-container {
  position: relative;
  width: 100%;
  border-radius: 10px;
  overflow: hidden;
  margin-bottom: 20px;
  z-index: 2;
}

.iframe-container {
  position: relative;
  width: 100%;
  padding-top: 56.25%; /* Соотношение сторон 16:9 */
  height: 0;
  overflow: hidden;
}

.responsive-iframe {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border: none;
}

/* Кнопка закрытия театрального режима */
.close-theater-btn {
  position: fixed;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(255, 0, 0, 0.7);
  color: white;
  font-size: 24px;
  border: none;
  padding: 8px 14px;
  cursor: pointer;
  border-radius: 50%;
  transition: background-color 0.3s, opacity 0.3s;
  z-index: 1001;
  opacity: 0;
}

.close-theater-btn.visible {
  opacity: 1;
}

.close-theater-btn:hover {
  background: rgba(255, 0, 0, 1);
}

/* Обёртка для театрального режима */
.theater-player {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: black;
  z-index: 1000;
  border-radius: 0;
  margin: 0;
}

/* Кнопка управления (скрыта на мобилках) */
.controls {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 10px;
}

@media (max-width: 599px) {
  .theater-mode-btn {
    display: none;
  }
}

.theater-mode-btn {
  background-color: #444;
  color: #fff;
  border: none;
  padding: 8px 12px;
  cursor: pointer;
  border-radius: 4px;
  transition: background-color 0.3s;
  z-index: 10;
}

.theater-mode-btn:hover {
  background-color: #666;
}
</style>
