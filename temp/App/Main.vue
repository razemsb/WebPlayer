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
      volume: 0.7,
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
    }
  },
  methods: {
    // Переключение между вкладками
    switchTab(tabName) {
      this.activeTab = tabName;
      // При открытии медиатеки обновляем список избранных
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
      let track = this.tracks.find(t => t.id === trackId);
      if (!track) {
        // Если трек не найден в основном списке, ищем в избранных
        track = this.favoriteTracks.find(t => t.id === trackId);
      }
      
      if (!track) return;
      
      this.currentTrackIndex = this.tracks.findIndex(t => t.id === trackId);
      this.currentTrack = track;
      
      // Добавляем в недавние
      if (!this.recentTracks.some(t => t.id === trackId)) {
        this.recentTracks.unshift({...track});
        if (this.recentTracks.length > 6) {
          this.recentTracks.pop();
        }
        this.saveRecentTracks();
        this.calculateMaxSlide();
      }
      
      this.resetPlayer();
      this.play();
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
          this.tracks = data.map((track, index) => ({
            ...track,
            id: track.id || index + 1,
            duration: track.duration || '03:30'
          }));
          
          this.initFavorites();
          this.initRecentTracks();
          this.initAudioPlayer();
          
          this.isLoading = false;
        })
        .catch(error => {
          console.error('Error loading tracks:', error);
          this.loadFallbackTracks();
          this.isLoading = false;
        });
    },
    
    // Инициализация избранных треков
    initFavorites() {
      const savedFavorites = localStorage.getItem('favoriteTracks');
      if (savedFavorites) {
        const favoriteIds = JSON.parse(savedFavorites);
        this.tracks.forEach(track => {
          track.favorited = favoriteIds.includes(track.id);
        });
        this.favoriteTracks = this.tracks.filter(t => t.favorited);
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