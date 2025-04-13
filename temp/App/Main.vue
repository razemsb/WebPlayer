new Vue({
  el: "#app",
  data() {
    return {
      currentSlide: 0,
      maxSlide: 0,
      visibleCards: 5,
      audio: null,
      progressBar: "0%",
      duration: "00:00",
      currentTime: "00:00",
      isPlaying: false,
      volume: 0.1,
      tracks: [],
      currentTrack: null, // Плеер изначально пустой
      currentTrackIndex: 0,
      activeTab: "home", // Активная вкладка по умолчанию
      searchQuery: "",
      searchResults: [],
      recentTracks: [],
      favoriteTracks: [],
      isLoading: true
    };
  },
  computed: {
    getTabTitle() {
      const titles = {
        home: "Главная",
        search: "Поиск",
        library: "Моя медиатека"
      };
      return titles[this.activeTab];
    },
    tracksCount() {
      return this.tracks.length;
    }
  },
  methods: {
    declension(number, titles) {
      const cases = [2, 0, 1, 1, 1, 2];
      return titles[
        number % 100 > 4 && number % 100 < 20
          ? 2
          : cases[number % 10 < 5 ? number % 10 : 5]
      ];
    },
    // Форматируем только когда нужно
    formatCount(count, words) {
      return `${count} ${this.declension(count, words)}`;
    },
    
    // Переключение между вкладками
    switchTab(tabName) {
      this.activeTab = tabName;
      if (tabName === 'library') {
        this.favoriteTracks = this.tracks.filter(track => track.favorited);
      }
    },


    // Методы для слайдера
    slide(direction) {
      const newSlide = this.currentSlide + direction;
      this.currentSlide = Math.max(0, Math.min(newSlide, this.maxSlide));
    },

    calculateMaxSlide() {
      this.maxSlide = Math.max(this.recentTracks.length - this.visibleCards, 0);
    },

    // Воспроизведение трека
    playTrack(trackId) {
      // Если кликаем по уже играющему треку - ставим на паузу
      if (this.currentTrack && this.currentTrack.id === trackId && this.isPlaying) {
        this.audio.pause();
        this.isPlaying = false;
        return;
      }

      let track = this.tracks.find(t => t.id === trackId);
      if (!track) {
        track = this.favoriteTracks.find(t => t.id === trackId);
      }

      if (!track) return;

      this.currentTrackIndex = this.tracks.findIndex(t => t.id === trackId);
      this.currentTrack = track;

      if (!this.recentTracks.some(t => t.id === trackId)) {
        this.recentTracks.unshift({ ...track });
        if (this.recentTracks.length > 6) {
          this.recentTracks.pop();
        }
        this.saveRecentTracks();
        this.calculateMaxSlide();
      }

      this.resetPlayer();
      this.play(); // Автоматически запускаем, если это новый трек
    },
    searchArtist(artistName) {
      this.activeTab = 'search'; // Переключаем на вкладку поиска
      this.searchQuery = artistName; // Вставляем имя артиста в поиск
      this.searchTracks(); // Запускаем поиск
    },

    // Управление воспроизведением
    play() {
      if (!this.currentTrack || !this.audio) return;

      if (this.audio.paused) {
        this.audio.play()
          .then(() => {
            this.isPlaying = true;
          })
          .catch(error => {
            console.error("Playback failed:", error);
          });
      } else {
        this.audio.pause();
        this.isPlaying = false;
      }
    },

    // Обновление прогресса воспроизведения
    updateProgress() {
      if (!this.audio || !this.audio.duration) return;

      const progress = (this.audio.currentTime / this.audio.duration) * 100;
      this.progressBar = progress + "%";

      const formatTime = (time) => {
        const minutes = Math.floor(time / 60);
        const seconds = Math.floor(time % 60);
        return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
      };

      this.duration = formatTime(this.audio.duration);
      this.currentTime = formatTime(this.audio.currentTime);
    },

    // Клик по прогресс-бару
    clickProgress(e) {
      if (!this.currentTrack || !this.audio || !this.audio.duration) return;

      const progressBar = e.currentTarget;
      const clickPosition = e.clientX - progressBar.getBoundingClientRect().left;
      const progressBarWidth = progressBar.offsetWidth;
      const progress = (clickPosition / progressBarWidth) * 100;

      this.progressBar = progress + "%";
      this.audio.currentTime = (progress / 100) * this.audio.duration;
    },

    // Предыдущий трек
    prevTrack() {
      if (this.tracks.length === 0) return;

      this.currentTrackIndex = this.currentTrackIndex > 0
        ? this.currentTrackIndex - 1
        : this.tracks.length - 1;
      this.currentTrack = this.tracks[this.currentTrackIndex];
      this.resetPlayer();
      if (this.isPlaying) {
        this.audio.play()
          .catch(error => {
            console.error("Playback failed:", error);
            this.isPlaying = false;
          });
      }
    },

    // Следующий трек
    nextTrack() {
      if (this.tracks.length === 0) return;

      this.currentTrackIndex = this.currentTrackIndex < this.tracks.length - 1
        ? this.currentTrackIndex + 1
        : 0;
      this.currentTrack = this.tracks[this.currentTrackIndex];
      this.resetPlayer();
      if (this.isPlaying) {
        this.audio.play()
          .catch(error => {
            console.error("Playback failed:", error);
            this.isPlaying = false;
          });
      }
    },

    // Сброс плеера
    resetPlayer() {
      if (!this.currentTrack || !this.audio) return;

      this.progressBar = "0%";
      this.currentTime = "00:00";
      this.audio.pause();
      this.audio.src = this.currentTrack.source;
      this.audio.load();
    },

    // Добавление/удаление из избранного
    toggleFavorite(trackId) {
      const track = this.tracks.find(t => t.id === trackId);
      if (!track) return;

      track.favorited = !track.favorited;
      this.favoriteTracks = this.tracks.filter(t => t.favorited);
      localStorage.setItem('favoriteTracks', JSON.stringify(this.favoriteTracks.map(t => t.id)));

      // Если мы в медиатеке, обновляем список
      if (this.activeTab === 'library') {
        this.favoriteTracks = this.tracks.filter(t => t.favorited);
      }
    },

    // Установка громкости
    setVolume() {
      if (this.audio) {
        this.audio.volume = this.volume;
      }
    },

    // Поиск треков
    searchTracks() {
      if (!this.searchQuery) {
        this.searchResults = [];
        return;
      }

      const query = this.searchQuery.toLowerCase();
      this.searchResults = this.tracks.filter(track =>
        track.name.toLowerCase().includes(query) ||
        track.artist.toLowerCase().includes(query) ||
        (track.album && track.album.toLowerCase().includes(query)))
        .slice(0, 20);
    },

    // Загрузка треков
    loadTracks() {
      fetch('temp/api/track.json')
        .then(response => {
          if (!response.ok) throw new Error('Network response was not ok');
          return response.json();
        })
        .then(data => {
          this.tracks = data.map((track, index) => {
            // Создаём аудио объект для определения длительности
            const audio = new Audio(track.source);

            return {
              ...track,
              id: track.id || index + 1,
              // Если duration не указан, ставим "загрузка" и обновим позже
              duration: track.duration || '--:--',
              _audioObj: audio // сохраняем ссылку на аудио для измерения
            };
          });

          this.initFavorites();
          this.initRecentTracks();
          this.initAudioPlayer();
          this.isLoading = false;

          // Запускаем измерение длительности для всех треков
          this.calculateDurations();
        })
        .catch(error => {
          console.error('Error loading tracks:', error);
          this.loadFallbackTracks();
          this.isLoading = false;
        });
    },

    // Новый метод для измерения длительности
    calculateDurations() {
      this.tracks.forEach(track => {
        if (track.duration === '--:--' && track._audioObj) {
          track._audioObj.addEventListener('loadedmetadata', () => {
            const duration = track._audioObj.duration;
            track.duration = this.formatTime(duration);
            this.$forceUpdate(); // принудительное обновление vue
          });

          // Начинаем загрузку метаданных
          track._audioObj.load();
        }
      });
    },

    // Вспомогательный метод для форматирования времени
    formatTime(seconds) {
      const mins = Math.floor(seconds / 60);
      const secs = Math.floor(seconds % 60);
      return `${mins}:${secs.toString().padStart(2, '0')}`;
    },

    initFavorites() {
      const savedFavorites = localStorage.getItem('favoriteTracks');
      if (savedFavorites) {
        const favoriteIds = JSON.parse(savedFavorites);
        this.tracks.forEach(track => {
          track.favorited = favoriteIds.includes(track.id);
        });
        this.favoriteTracks = this.tracks.filter(t => t.favorited);
      } else {
        this.favoriteTracks = []; // Если нет сохранённых, инициализируем пустым массивом
      }
    },

    // Инициализация недавних треков
    initRecentTracks() {
      const savedRecent = localStorage.getItem('recentTracks');
      if (savedRecent) {
        const recentIds = JSON.parse(savedRecent);
        this.recentTracks = recentIds
          .map(id => this.tracks.find(t => t.id === id))
          .filter(t => t)
          .slice(0, 6);
      }
      this.calculateMaxSlide();
    },

    // Инициализация аудиоплеера
    initAudioPlayer() {
      this.audio = new Audio();
      this.audio.volume = this.volume;

      this.audio.addEventListener('timeupdate', this.updateProgress.bind(this));
      this.audio.addEventListener('loadedmetadata', this.updateProgress.bind(this));
      this.audio.addEventListener('ended', this.nextTrack.bind(this));
    },

    // Запасные треки
    loadFallbackTracks() {
      this.tracks = [
        {
          id: 1,
          name: "В диапозоне",
          artist: "Порнофильмы",
          album: "В диапозоне между отчанием и надеждой",
          cover: "temp/audio/img/17.jpg",
          source: "temp/audio/mp3/18.mp3",
          duration: "03:20",
          favorited: false
        },
        {
          id: 2,
          name: "Мертвый Анархист",
          artist: "Король И Шут",
          album: "Король и Шут Оффициальный саундтрек",
          cover: "temp/audio/img/16.jpg",
          source: "temp/audio/mp3/16.mp3",
          duration: "04:15",
          favorited: false
        },
        {
          id: 3,
          name: "Охотник",
          artist: "Король И Шут",
          album: "Король и Шут Оффициальный саундтрек",
          cover: "temp/audio/img/16.jpg",
          source: "temp/audio/mp3/17.mp3",
          duration: "03:45",
          favorited: false
        }
      ];
      this.initAudioPlayer();
    },

    // Сохранение недавних треков
    saveRecentTracks() {
      const recentIds = this.recentTracks.map(t => t.id);
      localStorage.setItem('recentTracks', JSON.stringify(recentIds));
    }
  },
  mounted() {
    this.loadTracks();
    window.addEventListener('resize', this.calculateMaxSlide);
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.calculateMaxSlide);
  }
});