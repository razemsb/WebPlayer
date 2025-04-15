new Vue({
  el: "#app",
  data() {
    return {
      albums: [], // Автоматически сгенерируется из треков
      currentAlbum: null,
      trackResults: [],
      albumResults: [],
      currentSlide: 0,
      maxSlide: 0,
      visibleCards: 5,
      audio: null,
      progressBar: "0%",
      duration: "00:00",
      currentTime: "00:00",
      isPlaying: false,
      volume: 0.25,
      isMuted: false,
      previousVolume: 0.25,
      tracks: [],
      currentTrack: null,
      currentTrackIndex: 0,
      activeTab: "home",
      searchQuery: "",
      recentTracks: [],
      favoriteTracks: [],
      isLoading: true,
      playQueue: [],
      isRepeat: false,
      isRandom: false,
      isDraggingVolume: false,
    };
  },
  watch: {
    // Следим за изменениями состояния и сохраняем их
    activeTab(newVal) {
      localStorage.setItem('activeTab', newVal);
    },
    currentAlbum(newVal) {
      localStorage.setItem('currentAlbum', JSON.stringify(newVal));
    },
    currentTrack(newVal) {
      localStorage.setItem('currentTrack', JSON.stringify(newVal));
    },
    volume(newVal) {
      localStorage.setItem('volume', newVal);
    },
    isPlaying(newVal) {
      localStorage.setItem('isPlaying', newVal);
    }
  },
  computed: {
    getTabTitle() {
      const titles = {
        home: "Главная",
        search: "Поиск",
        library: "Моя медиатека",
        'album-tracks': "Альбом"
      };
      return titles[this.activeTab];
    },
    formattedTracksCount() {
      return this.formatCount(this.tracks.length, ['трек', 'трека', 'треков']);
    },
    albumTracks() {
      if (!this.currentAlbum) return [];
      return this.tracks.filter(track => track.album === this.currentAlbum.name);
    }
  },
  methods: {
    // Сохранение состояния
    saveState() {
      if (this.audio) {
        localStorage.setItem('currentTime', this.audio.currentTime);
      }
    },

    // Восстановление состояния
    restoreState() {
      // Восстанавливаем активную вкладку
      const savedTab = localStorage.getItem('activeTab');
      if (savedTab) {
        this.activeTab = savedTab;
      }

      // Восстанавливаем текущий альбом
      const savedAlbum = localStorage.getItem('currentAlbum');
      if (savedAlbum) {
        this.currentAlbum = JSON.parse(savedAlbum);
      }

      // Восстанавливаем громкость
      const savedVolume = localStorage.getItem('volume');
      if (savedVolume !== null) {
        this.volume = parseFloat(savedVolume);
        if (this.audio) {
          this.audio.volume = this.volume;
        }
      }

      // Восстанавливаем текущий трек и его состояние
      const savedTrack = localStorage.getItem('currentTrack');
      const savedTime = localStorage.getItem('currentTime');
      const wasPlaying = localStorage.getItem('isPlaying') === 'true';
      
      if (savedTrack) {
        const trackData = JSON.parse(savedTrack);
        const track = this.tracks.find(t => t.id === trackData.id);
        if (track) {
          this.currentTrack = track;
          if (this.audio) {
            this.audio.src = track.source;
            if (savedTime) {
              this.audio.currentTime = parseFloat(savedTime);
            }
            if (wasPlaying) {
              this.play();
            }
          }
        }
      }
    },

    // Инициализация
    async loadTracks() {
      try {
        const response = await fetch('temp/api/track.json');
        const tracks = await response.json();
        
        this.tracks = tracks.map(track => ({
          ...track,
          dateAdded: this.formatDate(new Date(track.dateAdded || Date.now())),
          formattedDuration: '--:--',
          favorited: false
        }));
        
        this.tracks.forEach(track => {
          this.loadTrackDuration(track);
        });
        
        this.generateAlbums();
        this.initFavorites();
        this.initRecentTracks();
        this.initAudioPlayer();
        
        // Сразу инициализируем рекомендации после загрузки треков
        this.trackResults = this.getRandomItems(this.tracks, 5);
        this.albumResults = this.getRandomItems(this.albums, 2);
        
        // Восстанавливаем состояние после загрузки треков
        this.restoreState();
        
        this.isLoading = false;
      } catch (error) {
        console.error('Ошибка загрузки треков:', error);
        this.isLoading = false;
      }
    },

    formatDate(date) {
      const now = new Date();
      const diff = now - date;
      const days = Math.floor(diff / (1000 * 60 * 60 * 24));
      
      if (days === 0) return 'Сегодня';
      if (days === 1) return 'Вчера';
      if (days < 7) return `${days} дн. назад`;
      if (days < 30) return `${Math.floor(days / 7)} нед. назад`;
      if (days < 365) return `${Math.floor(days / 30)} мес. назад`;
      return `${Math.floor(days / 365)} г. назад`;
    },

    loadTrackDuration(track) {
      const audio = new Audio();
      audio.preload = 'metadata';
      
      audio.addEventListener('loadedmetadata', () => {
        track.duration = audio.duration;
        track.formattedDuration = this.formatDuration(audio.duration);
        this.$forceUpdate();
      });

      audio.addEventListener('error', () => {
        track.duration = 0;
        track.formattedDuration = '--:--';
        console.error(`Ошибка загрузки длительности для трека ${track.name}`);
      });

      audio.src = track.source;
    },

    formatDuration(seconds) {
      if (!seconds) return '--:--';
      const minutes = Math.floor(seconds / 60);
      const remainingSeconds = Math.floor(seconds % 60);
      return `${String(minutes).padStart(2, '0')}:${String(remainingSeconds).padStart(2, '0')}`;
    },

    generateAlbums() {
      const albumsMap = {};
      
      this.tracks.forEach(track => {
        if (!albumsMap[track.album]) {
          albumsMap[track.album] = {
            name: track.album,
            artist: track.artist,
            cover: track.cover,
            tracks: []
          };
        }
        albumsMap[track.album].tracks.push(track.id);
      });
      
      this.albums = Object.values(albumsMap).map((album, index) => ({
        album_id: index + 1,
        name: album.name,
        artist: album.artist,
        cover: album.cover,
        tracks_ids: album.tracks,
        tracks_count: album.tracks.length
      }));
    },

    // Работа с альбомами
    showAlbumTracks(album) {
      this.currentAlbum = album;
      this.activeTab = 'album-tracks';
    },

    showAlbumByName(albumName) {
      console.log('Searching for album:', albumName);
      console.log('Available albums:', this.albums.map(a => a.name));
      const album = this.albums.find(a => a.name === albumName);
      console.log('Found album:', album);
      if (album) {
        this.showAlbumTracks(album);
      }
    },

    playAlbum(album) {
      const tracks = this.tracks.filter(t => album.tracks_ids.includes(t.id));
      if (tracks.length === 0) return;
      
      this.currentTrack = tracks[0];
      this.playQueue = tracks.slice(1);
      this.resetPlayer();
      this.play();
    },

    // Поиск
    searchTracks() {
      if (!this.searchQuery) {
        this.updateRandomSuggestions();
        return;
      }
      
      const query = this.searchQuery.toLowerCase();
      this.trackResults = this.tracks.filter(track => 
        track.name.toLowerCase().includes(query) || 
        track.artist.toLowerCase().includes(query) ||
        track.album.toLowerCase().includes(query)
      );
      
      this.albumResults = this.albums.filter(album => 
        album.name.toLowerCase().includes(query) ||
        album.artist.toLowerCase().includes(query)
      );
    },

    searchArtist(artist) {
      this.activeTab = 'search';
      this.searchQuery = artist;
      this.searchTracks();
      // Предотвращаем воспроизведение текущего трека
      if (this.currentTrack) {
        this.pause();
      }
    },

    searchAlbum(album) {
      this.activeTab = 'search';
      this.searchQuery = album;
      this.searchTracks();
      // Предотвращаем воспроизведение текущего трека
      if (this.currentTrack) {
        this.pause();
      }
    },

    // Воспроизведение
    playTrack(trackId) {
      const track = this.tracks.find(t => t.id === trackId);
      if (!track) return;

      // Если это тот же трек
      if (this.currentTrack?.id === trackId) {
        if (this.isPlaying) {
          this.pause();
        } else {
          // Просто возобновляем воспроизведение без изменения времени
          this.audio.play().then(() => {
            this.isPlaying = true;
          }).catch(error => {
            console.error('Ошибка воспроизведения:', error);
            this.isPlaying = false;
          });
        }
        return;
      }

      // Если это новый трек
      if (this.audio) {
        // Останавливаем текущий трек если он есть
        this.audio.pause();
        this.isPlaying = false;
      }

      this.currentTrack = track;
      this.currentTrackIndex = this.tracks.indexOf(track);
      this.addToRecent(track);
      
      // Загружаем и воспроизводим новый трек
      this.audio.src = track.source;
      this.audio.currentTime = 0; // Сбрасываем время только для нового трека
      this.audio.play().then(() => {
        this.isPlaying = true;
      }).catch(error => {
        console.error('Ошибка воспроизведения:', error);
        this.isPlaying = false;
      });
    },

    addToRecent(track) {
      const index = this.recentTracks.findIndex(t => t.id === track.id);
      if (index !== -1) {
        this.recentTracks.splice(index, 1);
      }
      this.recentTracks.unshift(track);
      if (this.recentTracks.length > 10) {
        this.recentTracks.pop();
      }
      this.saveRecentTracks();
    },

    initAudioPlayer() {
      this.audio = new Audio();
      this.audio.volume = this.volume;
      
      this.audio.addEventListener('timeupdate', () => {
        if (!this.audio.duration) return;
        this.currentTime = this.formatDuration(this.audio.currentTime);
        this.progressBar = (this.audio.currentTime / this.audio.duration * 100) + '%';
        // Сохраняем текущее время воспроизведения
        this.saveState();
      });
      
      this.audio.addEventListener('loadedmetadata', () => {
        this.duration = this.formatDuration(this.audio.duration);
      });
      
      this.audio.addEventListener('ended', () => {
        if (this.isRepeat) {
          this.audio.currentTime = 0;
          this.play();
        } else {
          this.nextTrack();
        }
      });

      this.audio.addEventListener('error', (e) => {
        console.error('Ошибка аудио:', e);
        this.isPlaying = false;
      });

      // Добавляем обработчик для сохранения состояния при закрытии страницы
      window.addEventListener('beforeunload', () => {
        this.saveState();
      });
    },

    pause() {
      if (this.audio && this.isPlaying) {
        this.audio.pause();
        this.isPlaying = false;
      }
    },

    play() {
      if (!this.currentTrack || !this.audio) return;

      this.audio.play().then(() => {
        this.isPlaying = true;
      }).catch(error => {
        console.error('Ошибка воспроизведения:', error);
        this.isPlaying = false;
      });
    },

    nextTrack() {
      if (!this.currentTrack) return;
      
      let nextIndex = this.currentTrackIndex + 1;
      if (nextIndex >= this.tracks.length) {
        nextIndex = 0;
      }
      
      this.playTrack(this.tracks[nextIndex].id);
    },

    prevTrack() {
      if (!this.currentTrack) return;
      
      let prevIndex = this.currentTrackIndex - 1;
      if (prevIndex < 0) {
        prevIndex = this.tracks.length - 1;
      }
      
      this.playTrack(this.tracks[prevIndex].id);
    },

    toggleRepeat() {
      this.isRepeat = !this.isRepeat;
      if (this.audio) {
        this.audio.loop = this.isRepeat;
      }
    },

    toggleRandom() {
      this.isRandom = !this.isRandom;
    },

    toggleMute() {
      if (this.volume > 0) {
        this.previousVolume = this.volume;
        this.volume = 0;
      } else {
        this.volume = this.previousVolume;
      }
      if (this.audio) {
        this.audio.volume = this.volume;
      }
    },

    handleVolumeClick(event) {
      const rect = event.currentTarget.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const width = rect.width;
      // Округляем до 2 знаков после запятой для получения 100 шагов (0.01 * 100 = 1)
      this.volume = Math.round(Math.max(0, Math.min(1, x / width)) * 100) / 100;
      if (this.audio) {
        this.audio.volume = this.volume;
      }
    },

    clickProgress(event) {
      if (!this.audio || !this.audio.duration) return;
      
      const rect = event.currentTarget.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const width = rect.width;
      const time = (x / width) * this.audio.duration;
      
      this.audio.currentTime = time;
    },

    // Вспомогательные методы
    formatCount(count, titles) {
      return `${count} ${this.declension(count, titles)}`;
    },

    declension(number, titles) {
      const cases = [2, 0, 1, 1, 1, 2];
      return titles[
        number % 100 > 4 && number % 100 < 20
          ? 2
          : cases[number % 10 < 5 ? number % 10 : 5]
      ];
    },

    resetPlayer() {
      if (!this.currentTrack || !this.audio) return;

      // Сбрасываем прогресс только если это новый трек
      if (this.audio.src !== this.currentTrack.source) {
        this.progressBar = "0%";
        this.currentTime = "00:00";
        this.duration = "00:00";
        this.audio.src = this.currentTrack.source;
        this.audio.load();
        
        // Применяем текущие настройки
        this.audio.volume = this.volume;
        this.audio.loop = this.isRepeat;
      }
    },

    updateProgress() {
      if (!this.audio) return;

      if (this.audio.duration) {
        const progress = (this.audio.currentTime / this.audio.duration) * 100;
        this.progressBar = progress + "%";
        this.currentTime = this.formatTime(this.audio.currentTime);
        this.duration = this.formatTime(this.audio.duration);
      }
    },

    formatTime(seconds) {
      if (!seconds || isNaN(seconds)) return '00:00';
      const minutes = Math.floor(seconds / 60);
      const remainingSeconds = Math.floor(seconds % 60);
      return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
    },

    // Работа с избранным
    isTrackFavorite(track) {
      return this.favoriteTracks.some(t => t.id === track.id);
    },

    toggleFavorite(track) {
      const index = this.favoriteTracks.findIndex(t => t.id === track.id);
      if (index === -1) {
        this.favoriteTracks.push(track);
      } else {
        this.favoriteTracks.splice(index, 1);
      }
      this.saveFavorites();
    },

    initFavorites() {
      try {
        const saved = localStorage.getItem('favoriteTracks');
        if (saved) {
          const favoriteIds = JSON.parse(saved);
          this.favoriteTracks = this.tracks.filter(track => 
            favoriteIds.includes(track.id)
          );
        }
      } catch (error) {
        console.error('Ошибка загрузки избранных треков:', error);
      }
    },

    saveFavorites() {
      try {
        const favoriteIds = this.favoriteTracks.map(track => track.id);
        localStorage.setItem('favoriteTracks', JSON.stringify(favoriteIds));
      } catch (error) {
        console.error('Ошибка сохранения избранных треков:', error);
      }
    },

    initRecentTracks() {
      try {
        const saved = localStorage.getItem('recentTracks');
        if (saved) {
          const recentIds = JSON.parse(saved);
          this.recentTracks = recentIds
            .map(id => this.tracks.find(track => track.id === id))
            .filter(Boolean);
        }
      } catch (error) {
        console.error('Ошибка загрузки недавних треков:', error);
      }
    },

    saveRecentTracks() {
      try {
        const recentIds = this.recentTracks.map(track => track.id);
        localStorage.setItem('recentTracks', JSON.stringify(recentIds));
      } catch (error) {
        console.error('Ошибка сохранения недавних треков:', error);
      }
    },

    // Навигация
    switchTab(tabName) {
      this.activeTab = tabName;
      if (tabName === 'search') {
        this.updateRandomSuggestions();
      } else if (tabName === 'library') {
        this.favoriteTracks = this.tracks.filter(t => t.favorited);
      }
    },

    slide(direction) {
      const maxSlide = Math.max(0, Math.ceil(this.recentTracks.length / this.visibleCards) - 1);
      this.currentSlide = Math.max(0, Math.min(maxSlide, this.currentSlide + direction));
    },

    handleVolumeChange(event) {
      // Используем значение из range input (0-100) и преобразуем в диапазон 0-1
      this.volume = parseFloat(event.target.value) / 100;
      if (this.audio) {
        this.audio.volume = this.volume;
      }
    },

    // Добавляем новые методы
    getRandomItems(array, count) {
      const shuffled = [...array].sort(() => 0.5 - Math.random());
      return shuffled.slice(0, count);
    },

    updateRandomSuggestions() {
      if (this.tracks.length && this.albums.length) {
        this.trackResults = this.getRandomItems(this.tracks, 5);
        this.albumResults = this.getRandomItems(this.albums, 2);
      }
    },
  },
  async mounted() {
    await this.loadTracks();
    window.addEventListener('resize', this.calculateMaxSlide);
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.calculateMaxSlide);
  }
});