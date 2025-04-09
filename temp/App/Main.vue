new Vue({
  el: "#app",
  data() {
    return {
      audio: null,
      progressBar: "0%",
      duration: "00:00",
      currentTime: "00:00",
      isPlaying: false,
      volume: 0.1,
      tracks: [],
      currentTrack: null,
      currentTrackIndex: 0,
      transitionName: null,
      isLoading: true 
    };
  },
  methods: {
    play() {
      if (!this.currentTrack) return;
      
      if (this.audio.paused) {
        this.audio.play();
        this.isPlaying = true;
      } else {
        this.audio.pause();
        this.isPlaying = false;
      }
    },
    updateProgress() {
      if (this.audio.duration) {
        const progress = (this.audio.currentTime / this.audio.duration) * 100;
        this.progressBar = progress + "%";

        const formatTime = (time) => {
          const minutes = Math.floor(time / 60);
          const seconds = Math.floor(time % 60);
          return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
        };

        this.duration = formatTime(this.audio.duration);
        this.currentTime = formatTime(this.audio.currentTime);
      }
    },
    clickProgress(e) {
      if (!this.currentTrack) return;
      
      const progressBar = this.$refs.progress;
      const clickPosition = e.clientX - progressBar.getBoundingClientRect().left;
      const progressBarWidth = progressBar.offsetWidth;
      const progress = (clickPosition / progressBarWidth) * 100;

      this.progressBar = progress + "%";
      this.audio.currentTime = (progress / 100) * this.audio.duration;
    },
    prevTrack() {
      if (this.tracks.length === 0) return;
      
      this.transitionName = "scale-in";
      this.currentTrackIndex = this.currentTrackIndex > 0
        ? this.currentTrackIndex - 1
        : this.tracks.length - 1;
      this.resetPlayer();
    },
    nextTrack() {
      if (this.tracks.length === 0) return;
      
      this.transitionName = "scale-out";
      this.currentTrackIndex = this.currentTrackIndex < this.tracks.length - 1
        ? this.currentTrackIndex + 1
        : 0;
      this.resetPlayer();
    },
    resetPlayer() {
      if (this.tracks.length === 0) return;
      
      this.progressBar = "0%";
      this.currentTime = "00:00";
      this.currentTrack = this.tracks[this.currentTrackIndex];

      if (this.audio) {
        this.audio.pause();
        this.audio.src = this.currentTrack.source;
        this.audio.load();

        if (this.isPlaying) {
          this.audio.play();
        }
      }
    },
    favorite() {
      if (this.currentTrack) {
        this.currentTrack.favorited = !this.currentTrack.favorited;
      }
    },
    setVolume() {
      if (this.audio) {
        this.audio.volume = this.volume;
      }
    },
    loadTracks() {
      fetch('temp/api/track.json')
        .then(response => {
          if (!response.ok) {
            throw new Error('Network response was not ok');
          }
          return response.json();
        })
        .then(data => {
          this.tracks = data;
          if (this.tracks.length > 0) {
            this.currentTrack = this.tracks[0];
            this.audio = new Audio(this.currentTrack.source);
            this.audio.volume = this.volume;
            
            this.audio.addEventListener('timeupdate', this.updateProgress);
            this.audio.addEventListener('loadedmetadata', this.updateProgress);
            this.audio.addEventListener('ended', () => {
              this.nextTrack();
              this.isPlaying = true;
            });
          }
          this.isLoading = false;
        })
        .catch(error => {
          console.error('Error loading tracks:', error);
          this.isLoading = false;
          // Можно добавить fallback-треки или сообщение об ошибке
        });
    }
  },
  created() {
    this.loadTracks();
  }
});