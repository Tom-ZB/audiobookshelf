
<template>
  <div class="w-full -mt-6">
    <div class="w-full relative mb-1">
      <div class="absolute -top-10 lg:top-0 right-0 lg:right-2 flex items-center h-full">
        <controls-playback-speed-control v-model="playbackRate" @input="setPlaybackRate" @change="playbackRateChanged" :playbackRateIncrementDecrement="playbackRateIncrementDecrement" class="mx-2 block" />
        
            <!-- allow user to select the audio and vtt file -->
    <div class="flex items-center justify-center mt-4">
      <ui-tooltip direction="top" text="Choose Local Audio">
        <button
          class="text-gray-300 hover:text-white mx-2"
          @click="openLocalFiles"
        >
          <span class="material-symbols text-2xl">audio_file</span>
        </button>
      </ui-tooltip>

      <ui-tooltip direction="top" text="Choose Local VTT">
        <button
          class="text-gray-300 hover:text-white mx-2"
          @click="openLocalVtt"
        >
          <span class="material-symbols text-2xl">subtitles</span>
        </button>
      </ui-tooltip>
    </div>
    <!-- allow user to select the audio and vtt file -->
     
        <ui-tooltip direction="bottom" :text="$strings.LabelVolume">
          <controls-volume-control ref="volumeControl" v-model="volume" @input="setVolume" class="mx-2 hidden sm:block" />
        </ui-tooltip>

        <ui-tooltip v-if="!hideSleepTimer" direction="top" :text="$strings.LabelSleepTimer">
          <button :aria-label="$strings.LabelSleepTimer" class="text-gray-300 hover:text-white mx-1 lg:mx-2" @mousedown.prevent @mouseup.prevent @click.stop="$emit('showSleepTimer')">
            <span v-if="!sleepTimerSet" class="material-symbols text-2xl">snooze</span>
            <div v-else class="flex items-center">
              <span class="material-symbols text-lg text-warning">snooze</span>
              <p class="text-sm sm:text-lg text-warning font-semibold text-center px-0.5 sm:pb-0.5 sm:min-w-8">{{ sleepTimerRemainingString }}</p>
            </div>
          </button>
        </ui-tooltip>

        <ui-tooltip v-if="!isPodcast && !hideBookmarks" direction="top" :text="$strings.LabelViewBookmarks">
          <button :aria-label="$strings.LabelViewBookmarks" class="text-gray-300 hover:text-white mx-1 lg:mx-2" @mousedown.prevent @mouseup.prevent @click.stop="$emit('showBookmarks')">
            <span class="material-symbols text-2xl">{{ bookmarks.length ? 'bookmarks' : 'bookmark_border' }}</span>
          </button>
        </ui-tooltip>

        <ui-tooltip v-if="chapters.length" direction="top" :text="$strings.LabelViewChapters">
          <button :aria-label="$strings.LabelViewChapters" class="text-gray-300 hover:text-white mx-1 lg:mx-2" @mousedown.prevent @mouseup.prevent @click.stop="showChapters">
            <span class="material-symbols text-2xl">format_list_bulleted</span>
          </button>
        </ui-tooltip>

        <ui-tooltip v-if="playerQueueItems.length" direction="top" :text="$strings.LabelViewQueue">
          <button :aria-label="$strings.LabelViewQueue" class="outline-hidden text-gray-300 mx-1 lg:mx-2 hover:text-white" @mousedown.prevent @mouseup.prevent @click.stop="$emit('showPlayerQueueItems')">
            <span class="material-symbols text-2.5xl sm:text-3xl">playlist_play</span>
          </button>
        </ui-tooltip>

        <ui-tooltip direction="top" :text="$strings.LabelViewPlayerSettings">
          <button :aria-label="$strings.LabelViewPlayerSettings" class="outline-hidden text-gray-300 mx-1 lg:mx-2 hover:text-white" @mousedown.prevent @mouseup.prevent @click.stop="showPlayerSettings">
            <span class="material-symbols text-2xl sm:text-2.5xl">settings_slow_motion</span>
          </button>
        </ui-tooltip>

        <!-- toggle for control if the subtitle display -->
        <ui-tooltip direction="top" text="Toggle Subtitles">
          <button
            aria-label="Toggle Subtitles"
            class="text-gray-300 hover:text-white mx-1 lg:mx-2"
            @click="showSubtitles = !showSubtitles"
          >
            <span class="material-symbols text-2xl">subtitles</span>
          </button>
        </ui-tooltip>

        <!-- button for control the subtitle color -->
        <ui-tooltip direction="top" text="Subtitle Color">
          <button
            aria-label="Subtitle Color"
            class="text-gray-300 hover:text-white mx-1 lg:mx-2"
            @click="subtitleColor = subtitleColor === '#ffffff' ? '#ffcc00' : '#ffffff'"
          >
            <span class="material-symbols text-2xl">palette</span>
          </button>
        </ui-tooltip>

        <!-- button for switch the subtitle style -->
        <ui-tooltip direction="top" text="Subtitle Font Style">
          <button
            aria-label="Subtitle Font Style"
            class="text-gray-300 hover:text-white mx-1 lg:mx-2"
            @click="toggleSubtitleFont"
          >
            <span class="material-symbols text-2xl">text_fields</span>
          </button>
        </ui-tooltip>
      </div>

      <player-playback-controls :loading="loading" :seek-loading="seekLoading" :playback-rate.sync="playbackRate" :paused="paused" :hasNextChapter="hasNextChapter" :hasNextItemInQueue="hasNextItemInQueue" @prevChapter="prevChapter" @next="goToNext" @jumpForward="jumpForward" @jumpBackward="jumpBackward" @setPlaybackRate="setPlaybackRate" @playPause="playPause" />
    </div>

    <player-track-bar ref="trackbar" :loading="loading" :chapters="chapters" :duration="duration" :current-chapter="currentChapter" :playback-rate="playbackRate" @seek="seek" />

    <div class="relative flex items-center justify-between">
      <div class="grow flex items-center">
        <p ref="currentTimestamp" class="font-mono text-xxs sm:text-sm text-gray-100 pointer-events-auto">00:00:00</p>
        <p class="font-mono text-sm hidden sm:block text-gray-100 pointer-events-auto">&nbsp;/&nbsp;{{ progressPercent }}%</p>
      </div>
      <div class="absolute left-1/2 transform -translate-x-1/2">
        <p class="text-xs sm:text-sm text-gray-300 pt-0.5 px-2 truncate">
          {{ currentChapterName }} <span v-if="useChapterTrack" class="text-xs text-gray-400">&nbsp;({{ $getString('LabelPlayerChapterNumberMarker', [currentChapterIndex + 1, chapters.length]) }})</span>
        </p>
      </div>
      <div class="grow flex items-center justify-end">
        <p class="font-mono text-xxs sm:text-sm text-gray-100 pointer-events-auto">{{ timeRemainingPretty }}</p>

        <!-- new area for vtt START -->
        <transition name="fade">
          <div
            v-if="showSubtitles && currentSubtitle"
            class="absolute bottom-[-10px] left-1/2 -translate-x-1/2  mb-2 px-3 py-2 bg-black bg-opacity-80 rounded-lg text-white text-sm max-w-xs text-center"
            style="min-width: 120px;"
            :style="{ fontFamily: subtitleFont,color: subtitleColor }"
          >
            {{ currentSubtitle }}
          </div>
        </transition>
        <!-- END -->

      </div>

    </div>
    <modals-chapters-modal v-model="showChaptersModal" :current-chapter="currentChapter" :playback-rate="playbackRate" :chapters="chapters" @select="selectChapter" />

    <modals-player-settings-modal v-model="showPlayerSettingsModal" />
  </div>
</template>

<script>

import { WebVTT } from 'vtt.js';
import axios from 'axios'
export default {
  props: {
    loading: Boolean,
    paused: Boolean,
    chapters: {
      type: Array,
      default: () => []
    },
    currentChapter: Object,
    bookmarks: {
      type: Array,
      default: () => []
    },
    sleepTimerSet: Boolean,
    sleepTimerRemaining: Number,
    sleepTimerType: String,
    isPodcast: Boolean,
    hideBookmarks: Boolean,
    hideSleepTimer: Boolean,
    hasNextItemInQueue: Boolean
  },
  data() {
    return {
      volume: 1,
      playbackRate: 1,
      audioEl: null,
      seekLoading: false,
      showChaptersModal: false,
      showPlayerSettingsModal: false,
      currentTime: 0,
      duration: 0,
      // new attributions START
      currentSubtitle: 'test',
      subtitles: [],
      subtitleFile: null, //load path from localstorage
      showSubtitles: true, // default as true
      subtitleColor: "#ffffff",
      subtitleFont: "Arial, sans-serif", //subtitle style

      // allow user to select local file
      localAudioUrl: null,
      localVttUrl: null,

      // END
    }
  },
  watch: {
    playbackRate() {
      this.updateTimestamp()
    },
    useChapterTrack() {
      if (this.$refs.trackbar) this.$refs.trackbar.setUseChapterTrack(this.useChapterTrack)
      this.updateTimestamp()
    },

    // monitor watch START
    currentTime(newTime) {
      this.updateCurrentSubtitle();
    }
    // watch END
  },
  computed: {
    sleepTimerRemainingString() {
      if (this.sleepTimerType === this.$constants.SleepTimerTypes.CHAPTER) {
        return 'EoC'
      } else {
        var rounded = Math.round(this.sleepTimerRemaining)
        if (rounded < 90) {
          return `${rounded}s`
        }
        var minutesRounded = Math.round(rounded / 60)
        if (minutesRounded <= 90) {
          return `${minutesRounded}m`
        }
        var hoursRounded = Math.round(minutesRounded / 60)
        return `${hoursRounded}h`
      }
    },
    timeRemaining() {
      if (this.useChapterTrack && this.currentChapter) {
        var currChapTime = this.currentTime - this.currentChapter.start
        return (this.currentChapterDuration - currChapTime) / this.playbackRate
      }
      return (this.duration - this.currentTime) / this.playbackRate
    },
    timeRemainingPretty() {
      if (this.timeRemaining < 0) {
        return this.$secondsToTimestamp(this.timeRemaining * -1)
      }
      return '-' + this.$secondsToTimestamp(this.timeRemaining)
    },
    progressPercent() {
      const duration = this.useChapterTrack ? this.currentChapterDuration : this.duration
      const time = this.useChapterTrack ? Math.max(this.currentTime - this.currentChapterStart) : this.currentTime

      if (!duration) return 0
      return Math.round((100 * time) / duration)
    },
    currentChapterName() {
      return this.currentChapter?.title || ''
    },
    currentChapterDuration() {
      if (!this.currentChapter) return 0
      return this.currentChapter.end - this.currentChapter.start
    },
    currentChapterStart() {
      if (!this.currentChapter) return 0
      return this.currentChapter.start
    },
    isFullscreen() {
      return this.$store.state.playerIsFullscreen
    },
    currentChapterIndex() {
      if (!this.currentChapter) return 0
      return this.chapters.findIndex((ch) => ch.id === this.currentChapter.id)
    },
    hasNextChapter() {
      if (!this.chapters.length) return false
      return this.currentChapterIndex < this.chapters.length - 1
    },
    playerQueueItems() {
      return this.$store.state.playerQueueItems || []
    },
    useChapterTrack() {
      const _useChapterTrack = this.$store.getters['user/getUserSetting']('useChapterTrack') || false
      return this.chapters.length ? _useChapterTrack : false
    },
    playbackRateIncrementDecrement() {
      return this.$store.getters['user/getUserSetting']('playbackRateIncrementDecrement')
    }
  },
  methods: {
      // method for open local file
      openLocalFiles() {
      const audioInput = document.createElement('input');
      audioInput.type = 'file';
      audioInput.accept = 'audio/*';  // 接受所有音频文件
      
      audioInput.onchange = (e) => {
        const file = e.target.files[0];
        if (file) {
          // 创建本地 URL
          this.localAudioUrl = URL.createObjectURL(file);
          this.$emit('setAudioSource', this.localAudioUrl);
          console.log('Selected audio file:', file.name);
        }
      };

      audioInput.click();
    },
    // method for open local vtt file
    openLocalVtt() {
      const vttInput = document.createElement('input');
      vttInput.type = 'file';
      vttInput.accept = '.vtt';  // 只接受 VTT 文件

      vttInput.onchange = (e) => {
        const file = e.target.files[0];
        if (file) {
          // 读取 VTT 文件内容
          const reader = new FileReader();
          reader.onload = (e) => {
            const vttContent = e.target.result;
            this.parseVTT(vttContent);
            console.log('Loaded VTT file:', file.name);
          };
          reader.readAsText(file);
        }
      };

      vttInput.click();
    },

    toggleFullscreen(isFullscreen) {
      this.$store.commit('setPlayerIsFullscreen', isFullscreen)
    },
    setDuration(duration) {
      this.duration = duration
    },
    setCurrentTime(time) {
      this.currentTime = time
      this.updateTimestamp()
      if (this.$refs.trackbar) this.$refs.trackbar.setCurrentTime(time)
    },
    playPause() {
      this.$emit('playPause')
    },
    jumpBackward() {
      this.$emit('jumpBackward')
    },
    jumpForward() {
      this.$emit('jumpForward')
    },
    increaseVolume() {
      if (this.volume >= 1) return
      this.volume = Math.min(1, this.volume + 0.1)
      this.setVolume(this.volume)
    },
    decreaseVolume() {
      if (this.volume <= 0) return
      this.volume = Math.max(0, this.volume - 0.1)
      this.setVolume(this.volume)
    },
    setVolume(volume) {
      this.$emit('setVolume', volume)
    },
    toggleMute() {
      if (this.$refs.volumeControl && this.$refs.volumeControl.toggleMute) {
        this.$refs.volumeControl.toggleMute()
      }
    },
    increasePlaybackRate() {
      if (this.playbackRate >= 10) return
      this.playbackRate = Number((this.playbackRate + this.playbackRateIncrementDecrement || 0.1).toFixed(2))
      this.setPlaybackRate(this.playbackRate)
    },
    decreasePlaybackRate() {
      if (this.playbackRate <= 0.5) return
      this.playbackRate = Number((this.playbackRate - this.playbackRateIncrementDecrement || 0.1).toFixed(2))
      this.setPlaybackRate(this.playbackRate)
    },
    playbackRateChanged(playbackRate) {
      this.setPlaybackRate(playbackRate)
      this.$store.dispatch('user/updateUserSettings', { playbackRate }).catch((err) => {
        console.error('Failed to update settings', err)
      })
    },
    setPlaybackRate(playbackRate) {
      this.$emit('setPlaybackRate', playbackRate)
    },
    selectChapter(chapter) {
      this.seek(chapter.start)
      this.showChaptersModal = false
    },
    setUseChapterTrack() {
      this.useChapterTrack = !this.useChapterTrack
      if (this.$refs.trackbar) this.$refs.trackbar.setUseChapterTrack(this.useChapterTrack)

      this.$store.dispatch('user/updateUserSettings', { useChapterTrack: this.useChapterTrack })
      this.updateTimestamp()
    },
    checkUpdateChapterTrack() {
      // Changing media in player may not have chapters
      if (!this.chapters.length && this.useChapterTrack) {
        this.useChapterTrack = false
        if (this.$refs.trackbar) this.$refs.trackbar.setUseChapterTrack(this.useChapterTrack)
      }
    },
    seek(time) {
      this.$emit('seek', time)
    },
    restart() {
      this.seek(0)
    },
    prevChapter() {
      if (!this.currentChapter || this.currentChapterIndex === 0) {
        return this.restart()
      }
      var timeInCurrentChapter = this.currentTime - this.currentChapter.start
      if (timeInCurrentChapter <= 3 && this.chapters[this.currentChapterIndex - 1]) {
        var prevChapter = this.chapters[this.currentChapterIndex - 1]
        this.seek(prevChapter.start)
      } else {
        this.seek(this.currentChapter.start)
      }
    },
    goToNext() {
      if (this.hasNextChapter) {
        const nextChapter = this.chapters[this.currentChapterIndex + 1]
        this.seek(nextChapter.start)
      } else if (this.hasNextItemInQueue) {
        this.$emit('nextItemInQueue')
      }
    },
    setStreamReady() {
      if (this.$refs.trackbar) this.$refs.trackbar.setPercentageReady(1)
    },
    setChunksReady(chunks, numSegments) {
      var largestSeg = 0
      for (let i = 0; i < chunks.length; i++) {
        var chunk = chunks[i]
        if (typeof chunk === 'string') {
          var chunkRange = chunk.split('-').map((c) => Number(c))
          if (chunkRange.length < 2) continue
          if (chunkRange[1] > largestSeg) largestSeg = chunkRange[1]
        } else if (chunk > largestSeg) {
          largestSeg = chunk
        }
      }
      var percentageReady = largestSeg / numSegments
      if (this.$refs.trackbar) this.$refs.trackbar.setPercentageReady(percentageReady)
    },
    updateTimestamp() {
      const ts = this.$refs.currentTimestamp
      if (!ts) {
        console.error('No timestamp el')
        return
      }
      const time = this.useChapterTrack ? Math.max(0, this.currentTime - this.currentChapterStart) : this.currentTime
      ts.innerText = this.$secondsToTimestamp(time / this.playbackRate)
    },
    setBufferTime(bufferTime) {
      if (this.$refs.trackbar) this.$refs.trackbar.setBufferTime(bufferTime)
    },
    showChapters() {
      if (!this.chapters.length) return
      this.showChaptersModal = !this.showChaptersModal
    },
    showPlayerSettings() {
      this.showPlayerSettingsModal = !this.showPlayerSettingsModal
    },
    init() {
      this.playbackRate = this.$store.getters['user/getUserSetting']('playbackRate') || 1

      if (this.$refs.trackbar) this.$refs.trackbar.setUseChapterTrack(this.useChapterTrack)
      this.setPlaybackRate(this.playbackRate)
    },
    settingsUpdated(settings) {
      if (settings.playbackRate && this.playbackRate !== settings.playbackRate) {
        this.setPlaybackRate(settings.playbackRate)
      }
    },
    closePlayer() {
      if (this.isFullscreen) {
        this.toggleFullscreen(false)
        return
      }

      if (this.loading) return
      this.$emit('close')
    },
    hotkey(action) {
      if (action === this.$hotkeys.AudioPlayer.PLAY_PAUSE) this.playPause()
      else if (action === this.$hotkeys.AudioPlayer.JUMP_FORWARD) this.jumpForward()
      else if (action === this.$hotkeys.AudioPlayer.JUMP_BACKWARD) this.jumpBackward()
      else if (action === this.$hotkeys.AudioPlayer.VOLUME_UP) this.increaseVolume()
      else if (action === this.$hotkeys.AudioPlayer.VOLUME_DOWN) this.decreaseVolume()
      else if (action === this.$hotkeys.AudioPlayer.MUTE_UNMUTE) this.toggleMute()
      else if (action === this.$hotkeys.AudioPlayer.SHOW_CHAPTERS) this.showChapters()
      else if (action === this.$hotkeys.AudioPlayer.INCREASE_PLAYBACK_RATE) this.increasePlaybackRate()
      else if (action === this.$hotkeys.AudioPlayer.DECREASE_PLAYBACK_RATE) this.decreasePlaybackRate()
      else if (action === this.$hotkeys.AudioPlayer.CLOSE) this.closePlayer()
    },

    // load vtt method START
      async loadVTTFile() {
        console.log('loadVTTFile called, subtitleFile:', this.subtitleFile);
        if (!this.subtitleFile) {
          console.error('not found');
          return;
        }
      //   try {
      //     console.log('method start');
      //     const response = await axios.get(this.subtitleFile);
      //     // const response = await axios.get('http://10.24.0.165:3333/static/1757393246709-Recording.vtt',{ responseType: 'text' });
      //     // console.log(response.json())
      //     if (!response.status === 200) return  new Error(`HTTP error! status: ${response.status}`);
      //     console.log('vtt load successful');
      //     const text = await response.data;
      //     console.log("TEXT: ")
      //     console.log(text)
      //     this.parseVTT(text);
      //   } catch (error) {
      //     console.error('vtt load failed:', error);
      //   }
      // },
    },
    parseVTT(vttData) {
      this.subtitles = []; // clear the old content

      // construct new parse instance
      const parser = new WebVTT.Parser(window, WebVTT.StringDecoder());

      // when parse a piece of subtitles, push it into subtitle array
      parser.oncue = (cue) => {
        this.subtitles.push({
          start: cue.startTime,
          end: cue.endTime,
          text: cue.text
        });
      };

      // catch error
      parser.onparsingerror = (error) => {
        console.error('Parsing error:', error);
      };

      // Sort the results after parsing (ensuring the order is chronological)
      parser.onflush = () => {
        this.subtitles.sort((a, b) => a.start - b.start);
        console.log('VTT parsing completed:', this.subtitles);
      };

      // perform the parse
      parser.parse(vttData);
      parser.flush();
    },

    updateCurrentSubtitle() {
      if (!this.subtitles.length || !this.currentTime) {
        this.currentSubtitle = '';
        return;
      }

      let low = 0, high = this.subtitles.length - 1;
      while (low <= high) {
        const mid = Math.floor((low + high) / 2);
        const sub = this.subtitles[mid];

        if (this.currentTime >= sub.start && this.currentTime <= sub.end) {
          this.currentSubtitle = sub.text;
          return;
        } else if (this.currentTime < sub.start) {
          high = mid - 1;
        } else {
          low = mid + 1;
        }
      }
      this.currentSubtitle = '';
    },
    toggleSubtitleFont() {
      const fonts = [
        "Arial, sans-serif",
        "'Times New Roman', serif",
        "'Courier New', monospace",
        "'Comic Sans MS', cursive"
      ]
      const currentIndex = fonts.indexOf(this.subtitleFont)
      this.subtitleFont = fonts[(currentIndex + 1) % fonts.length] // % fonts.length   (3 + 1) % 4 = 0 防止越界
    }
    // END
  },
  mounted() {
    console.log('mounted performed'); // ???no log
    this.$eventBus.$on('player-hotkey', this.hotkey)
    this.$eventBus.$on('user-settings', this.settingsUpdated)

    this.init()

    // execute method START
    //this.loadVTTFile();
    //  END
  },
  beforeDestroy() {
    this.$eventBus.$off('player-hotkey', this.hotkey)
    this.$eventBus.$off('user-settings', this.settingsUpdated)
    // revoke the local url to avoid memory leak
      if (this.localAudioUrl) {
      URL.revokeObjectURL(this.localAudioUrl);
    }
  }
}
</script>

<style>
.loadingTrack {
  animation-name: loadingTrack;
  animation-duration: 1s;
  animation-iteration-count: infinite;
}
@keyframes loadingTrack {
  0% {
    left: -25%;
  }
  100% {
    left: 100%;
  }
}

/* new style START */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter, .fade-leave-to {
  opacity: 0;
}
/*  END */
</style>
