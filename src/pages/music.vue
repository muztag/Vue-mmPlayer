<template>
  <div class="music">
    <div class="music-content">
      <div class="music-left">
        <music-btn />
        <keep-alive>
          <router-view v-if="$route.meta.keepAlive" class="music-list" />
        </keep-alive>
        <router-view
          v-if="!$route.meta.keepAlive"
          :key="$route.path"
          class="music-list"
        />
      </div>
      <lyric
        class="music-right"
        :lyric="lyric"
        :nolyric="nolyric"
        :lyric-index="lyricIndex"
      />
    </div>

    <!--播放器-->
    <div
      class="music-bar"
      :class="{ disable: !musicReady || !currentMusic.id }"
    >
      <div class="music-bar-btns">
        <mm-icon
          class="pointer"
          type="prev"
          :size="36"
          title="上一曲 Ctrl + Left"
          @click="prev"
        />
        <div
          class="control-play pointer"
          title="播放暂停 Ctrl + Space"
          @click="play"
        >
          <mm-icon :type="playing ? 'pause' : 'play'" :size="24" />
        </div>
        <mm-icon
          class="pointer"
          type="next"
          :size="36"
          title="下一曲 Ctrl + Right"
          @click="next"
        />
      </div>
      <div class="music-music">
        <div class="music-bar-info">
          <template v-if="currentMusic && currentMusic.id">
            {{ currentMusic.name }}
            <span>- {{ currentMusic.singer }}</span>
          </template>
          <template v-else>一起听歌</template>
        </div>
        <div v-if="currentMusic.id" class="music-bar-time">
          {{ currentTime | format }}/{{ currentMusic.duration % 3600 | format }}
        </div>
        <mm-progress
          class="music-progress"
          :percent="percentMusic"
          :percent-progress="currentProgress"
          @percentChange="progressMusic"
        />
      </div>

      <!-- 播放模式 -->
      <mm-icon
        class="icon-color pointer mode"
        :type="getModeIconType()"
        :title="getModeIconTitle()"
        :size="30"
        @click="modeChange"
      />

      <!-- 评论 -->
      <mm-icon
        class="icon-color pointer comment"
        type="comment"
        :size="30"
        @click="openComment"
      />

      <!-- 音量控制 -->
      <div class="music-bar-volume" title="音量加减 [Ctrl + Up / Down]">
        <volume :volume="volume" @volumeChange="volumeChange" />
      </div>
    </div>

    <!--遮罩-->
    <div class="mmPlayer-bg" :style="{ backgroundImage: picUrl }"></div>
    <div class="mmPlayer-mask"></div>
  </div>
</template>

<script>
import { getLyric, getCheckMusic, getMusicDetail } from 'api'
import mmPlayerMusic from './mmPlayer'
import {
  randomSortArray,
  parseLyric,
  format,
  silencePromise
} from '@/utils/util'
import { playMode, defaultBG } from '@/config'
import { getVolume, setVolume } from '@/utils/storage'
import { mapGetters, mapMutations, mapActions } from 'vuex'

import MmProgress from 'base/mm-progress/mm-progress'
import MusicBtn from 'components/music-btn/music-btn'
import Lyric from 'components/lyric/lyric'
import Volume from 'components/volume/volume'

// 同步用
import {
  setSyncStatus,
  setSyncDest,
  getSyncDest,
  getSyncStatus
} from "@/utils/storage.js";

export default {
  name: 'Music',
  components: {
    MmProgress,
    MusicBtn,
    Lyric,
    Volume
  },
  filters: {
    // 时间格式化
    format
  },
  data() {
    const volume = getVolume()
    return {
      musicReady: false, // 是否可以使用播放器
      currentTime: 0, // 当前播放时间
      currentProgress: 0, // 当前缓冲进度
      lyric: [], // 歌词
      nolyric: false, // 是否有歌词
      lyricIndex: 0, // 当前播放歌词下标
      isMute: false, // 是否静音
      volume , // 音量大小
      websock: null, //同步增加，websocket
      timer: null, //同步增加，用于心跳包的发送.
      self:this //同步增加
    }
  },
  computed: {
    picUrl() {
      return this.currentMusic.id && this.currentMusic.image
        ? `url(${this.currentMusic.image}?param=300y300)`
        : `url(${defaultBG})`
    },
    percentMusic() {
      const duration = this.currentMusic.duration
      return this.currentTime && duration ? this.currentTime / duration : 0
    },
    ...mapGetters([
      'audioEle',
      'mode',
      'playing',
      'playlist',
      'orderList',
      'currentIndex',
      'currentMusic',
      'historyList',
      "uid" //同步
    ])
  },
  watch: {
    currentMusic(newMusic, oldMusic) {
      if (!newMusic.id) {
        this.lyric = []
        return
      }
      if (newMusic.id === oldMusic.id) {
        return
      }
      this.audioEle.src = newMusic.url
      // 重置相关参数
      this.lyricIndex = this.currentTime = this.currentProgress = 0
      silencePromise(this.audioEle.play())
      this.$nextTick(() => {
        this._getLyric(newMusic.id)
      })
      var self =this; //这行和下一行，同步
      this.sync();
    },
    playing(newPlaying) {
      const audio = this.audioEle
      this.$nextTick(() => {
        newPlaying ? silencePromise(audio.play()) : audio.pause()
        this.musicReady = true
      })
      this.sync(); //同步
    },
    currentTime(newTime) {
      if (this.nolyric) {
        return
      }
      let lyricIndex = 0
      for (let i = 0; i < this.lyric.length; i++) {
        if (newTime > this.lyric[i].time) {
          lyricIndex = i
        }
      }
      this.lyricIndex = lyricIndex
    }
  },
  mounted() {
    this.$nextTick(() => {
      mmPlayerMusic.initAudio(this)
      this.initKeyDown()
      this.volumeChange(this.volume)
    })
  },
  methods: {
    // 按键事件
    initKeyDown() {
      document.onkeydown = e => {
        switch (e.ctrlKey && e.keyCode) {
          case 32: // 播放暂停Ctrl + Space
            this.play()
            break
          case 37: // 上一曲Ctrl + Left
            this.prev()
            break
          case 38: // 音量加Ctrl + Up
            let plus = Number((this.volume += 0.1).toFixed(1))
            if (plus > 1) {
              plus = 1
            }
            this.volumeChange(plus)
            break
          case 39: // 下一曲Ctrl + Right
            this.next()
            break
          case 40: // 音量减Ctrl + Down
            let reduce = Number((this.volume -= 0.1).toFixed(1))
            if (reduce < 0) {
              reduce = 0
            }
            this.volumeChange(reduce)
            break
          case 79: // 切换播放模式Ctrl + O
            this.modeChange()
            break
        }
      }
    },
    // 上一曲
    prev() {
      if (!this.musicReady) {
        return
      }
      if (this.playlist.length === 1) {
        this.loop()
      } else {
        let index = this.currentIndex - 1
        if (index < 0) {
          index = this.playlist.length - 1
        }
        this.setCurrentIndex(index)
        if (!this.playing && this.musicReady) {
          this.setPlaying(true)
        }
        this.musicReady = false
      }
    },
    // 播放暂停
    play() {
      if (!this.musicReady) {
        return
      }
      this.setPlaying(!this.playing)
    },
    // 下一曲
    // 当 flag 为 true 时，表示上一曲播放失败
    next(flag = false) {
      if (!this.musicReady) {
        return
      }
      const {
        playlist: { length }
      } = this
      if (
        (length - 1 === this.currentIndex && this.mode === playMode.order) ||
        (length === 1 && flag)
      ) {
        this.setCurrentIndex(-1)
        this.setPlaying(false)
        return
      }
      if (length === 1) {
        this.loop()
      } else {
        let index = this.currentIndex + 1
        if (index === length) {
          index = 0
        }
        if (!this.playing && this.musicReady) {
          this.setPlaying(true)
        }
        this.setCurrentIndex(index)
        this.musicReady = false
      }
    },
    // 循环
    loop() {
      this.audioEle.currentTime = 0
      silencePromise(this.audioEle.play())
      this.setPlaying(true)
      if (this.lyric.length > 0) {
        this.lyricIndex = 0
      }
    },
    // 修改音乐进度
    progressMusic(percent) {
      this.audioEle.currentTime = this.currentMusic.duration * percent
      this.sync()
    },
    //同步
    sync() {
      if (getSyncStatus() !== 'true') {
        return
      }
      var self = this
      self.websock.send(
        JSON.stringify({
          code: 600,
          id: self.uid,
          songid: self.currentMusic.id,
          currentTime: self.audioEle.currentTime,
          album: self.currentMusic.album,
          duration: self.currentMusic.duration,
          image: self.currentMusic.image,
          name: self.currentMusic.name,
          singer: self.currentMusic.singer,
          url: self.currentMusic.url,
          play: self.playing,
          timestamp: new Date().valueOf()
        })
      )
      console.log('立即更新数据..')
    },
    changeCurrentTime(time) {
      this.audioEle.currentTime = time
    },

    // 切换播放顺序
    modeChange() {
      const mode = (this.mode + 1) % 4
      this.setPlayMode(mode)
      if (mode === playMode.loop) {
        return
      }
      let list = []
      switch (mode) {
        case playMode.listLoop:
        case playMode.order:
          list = this.orderList
          break
        case playMode.random:
          list = randomSortArray(this.orderList)
          break
      }
      this.resetCurrentIndex(list)
      this.setPlaylist(list)
    },
    // 修改当前歌曲索引
    resetCurrentIndex(list) {
      const index = list.findIndex(item => {
        return item.id === this.currentMusic.id
      })
      this.setCurrentIndex(index)
    },
    // 打开音乐评论
    openComment() {
      if (!this.currentMusic.id) {
        this.$mmToast('还没有播放歌曲哦！')
        return false
      }
      this.$router.push(`/music/comment/${this.currentMusic.id}`)
    },
    // 修改音量大小
    volumeChange(percent) {
      percent === 0 ? (this.isMute = true) : (this.isMute = false)
      this.volume = percent
      this.audioEle.volume = percent
      setVolume(percent)
    },
    // 获取播放模式 icon
    getModeIconType() {
      return {
        [playMode.listLoop]: 'loop',
        [playMode.order]: 'sequence',
        [playMode.random]: 'random',
        [playMode.loop]: 'loop-one'
      }[this.mode]
    },
    // 获取播放模式 title
    getModeIconTitle() {
      const key = 'Ctrl + O'
      return {
        [playMode.listLoop]: `列表循环 ${key}`,
        [playMode.order]: `顺序播放 ${key}`,
        [playMode.random]: `随机播放 ${key}`,
        [playMode.loop]: `单曲循环 ${key}`
      }[this.mode]
    },
    // 获取歌词
    _getLyric(id) {
      getLyric(id).then(res => {
        if (res.status === 200) {
          if (res.data.nolyric) {
            this.nolyric = true
          } else {
            this.nolyric = false
            this.lyric = parseLyric(res.data.lrc.lyric)
          }
          silencePromise(this.audioEle.play())
        }
      })
    },
    ...mapMutations({
      setPlaying: 'SET_PLAYING',
      setPlaylist: 'SET_PLAYLIST',
      setCurrentIndex: 'SET_CURRENTINDEX'
    }),
    ...mapActions(['setHistory', 'setPlayMode', 'selectAddPlay'])
  },
  created() {
    // const wsuri = "ws://127.0.0.1:12345/" + this.uid; //这个地址由后端童鞋提供
    const wsuri = 'ws://129.204.108.71:12345/' + this.uid //这个地址由后端童鞋提供

    var self = this
    this.websock = new WebSocket(wsuri)
    this.websock.onmessage = function(e) {
      var responed = JSON.parse(e.data)
      if (getSyncStatus() === 'true') {
        //表示现在是房主,后续需要添加新的标识符改掉这里
        if (responed.code == 200) {
          //说明请求一切正常.开房成功
          //this.$mmToast("开启同步完成!");
          //每隔十秒钟进行一次心跳请求.
          this.timer = setInterval(function() {
            self.websock.send(
              JSON.stringify({
                code: 600,
                id: self.uid,
                songid: self.currentMusic.id,
                currentTime: self.audioEle.currentTime,
                album: self.currentMusic.album,
                duration: self.currentMusic.duration,
                image: self.currentMusic.image,
                name: self.currentMusic.name,
                singer: self.currentMusic.singer,
                url: self.currentMusic.url,
                play: self.playing,
                timestamp: new Date().valueOf()
              })
            )
          }, 10000)
        } else if (responed.code === 400) {
          //说明服务器已经开启了房间了,不能在开启了.
          console.log('房间已经有人开启了.')
        }
      } else {
        //表示不是房主,需要同步其他用户的数据
        if (responed.code === 500) {
          //说明没有房间.
          console.log('没有找到房间!')
        } else if (responed.code === 200) {
          //表示一切正常,有房间
          console.log('正常加入房间,准备同步房主数据!')
        } else if (responed.code === 600) {
          console.log('接收到数据:' + responed)
          console.log(self.currentMusic.id)
          console.log(responed.songid)
          if (self.currentMusic.id !== responed.songid) {
            //首先判断房主播放的歌曲是否还是一样的
            //如果是一样的话,表示是歌曲发生了变化.
            var music = {
              album: responed.album,
              duration: responed.duration,
              id: responed.songid,
              image: responed.image,
              name: responed.name,
              singer: responed.singer,
              url: responed.url
            }
            console.log(music)
            // self.currentMusic.album = responed.album;
            // self.currentMusic.id = responed.songid;
            // self.currentMusic.image = responed.image;
            // self.currentMusic.name = responed.name;
            // self.currentMusic.singer = responed.singer;
            // self.currentMusic.url = responed.url;
            self.selectAddPlay(music)
            self.setPlaying(false)
            self.setPlaying(true)
          }
          //计算进度是否发生了变化.
          var duringTime = (new Date().valueOf() - responed.timestamp) / 1000
          //计算经过了多少的传输时间加上进度
          var accuteDuration = responed.currentTime + duringTime
          if (accuteDuration > self.currentMusic.duration) {
            //如果时间已经超过这首歌的总长度说明房主应该已经切换到了下一首了.这里直接进行暂停.等待同步数据
            self.setPlaying(false)
          } else if (Math.abs(self.audioEle.currentTime - accuteDuration) > 3) {
            //如果目前两者的播放进度小于3秒的话那么就不直接跳转了.太精确容易导致卡顿
            self.changeCurrentTime(accuteDuration)
          }
          if (responed.play === true) {
            //说明还在播放.
            if (
              accuteDuration < self.currentMusic.duration &&
              self.playing === false
            ) {
              self.setPlaying(true)
            }
          } else {
            if (self.playing === true) {
              self.setPlaying(false)
            }
          }
        }
      }
    }
    this.websock.onopen = function(e) {
      //发送当前的用户id到服务器
      console.log('连接到服务器成功~')
      if (getSyncStatus() === 'true') {
        //表示当前是房主
        self.websock.send(
          JSON.stringify({
            code: 100,
            id: self.uid
          })
        )
      } else if (
        getSyncDest() !== undefined &&
        getSyncDest() !== null &&
        getSyncDest() !== ''
      ) {
        //如果dest目标不是为空的话.就发送到服务器请求连接.
        console.log('听众尝试连接到服务器')
        self.websock.send(
          JSON.stringify({
            code: 200,
            id: getSyncDest()
          })
        )
      }
    }
    this.websock.onerror = function(e) {
      //关闭计时器
      stopInterval(this.timer)
    }
    this.websock.onclose = function(e) {
      //关闭计时器
      stopInterval(this.timer)
    }
  },
  destroyed() { //同步
    // this.websocketclose();
}
}
</script>

<style lang="less">
.music {
  padding: 75px 25px 25px 25px;
  width: 100%;
  max-width: 1750px;
  margin: 0 auto;
  height: 100%;
  box-sizing: border-box;
  overflow: hidden;
  .music-content {
    display: flex;
    width: 100%;
    height: calc(~'100% - 80px');
    .music-left {
      flex: 1;
      height: 100%;
      overflow: hidden;
      .music-list {
        height: calc(~'100% - 60px');
      }
    }
    .music-right {
      position: relative;
      width: 310px;
      margin-left: 10px;
    }
  }

  /*底部mmPlayer-bar*/
  .music-bar {
    display: flex;
    align-items: center;
    width: 100%;
    height: 80px;
    box-sizing: border-box;
    padding-bottom: 15px;
    color: #fff;
    &.disable {
      pointer-events: none;
      opacity: 0.6;
    }
    .icon-color {
      color: #fff;
    }
    .music-bar-btns {
      display: flex;
      justify-content: space-between;
      align-items: center;
      width: 180px;
      .control-play {
        .flex-center;
        border-radius: 50%;
        width: 40px;
        height: 40px;
        color: #fff;
        background-color: rgba(255, 255, 255, 0.3);
        .icon-bofang101 {
          transform: translateX(2px);
        }
      }
    }

    .flex-center;
    .btn-prev {
      width: 19px;
      min-width: 19px;
      height: 20px;
      background-position: 0 -30px;
    }
    .btn-play {
      width: 21px;
      min-width: 21px;
      height: 29px;
      margin: 0 50px;
      background-position: 0 0;
      &.btn-play-pause {
        background-position: -30px 0;
      }
    }
    .btn-next {
      width: 19px;
      min-width: 19px;
      height: 20px;
      background-position: 0 -52px;
    }
    .music-music {
      position: relative;
      width: 100%;
      flex: 1;
      box-sizing: border-box;
      padding-left: 40px;
      font-size: @font_size_small;
      color: @text_color_active;
      .music-bar-info {
        height: 15px;
        padding-right: 80px;
        line-height: 15px;
        text-overflow: ellipsis;
        overflow: hidden;
        display: -webkit-box;
        -webkit-line-clamp: 1;
        -webkit-box-orient: vertical;
      }
      .music-bar-time {
        position: absolute;
        top: 0;
        right: 5px;
      }
    }
    .mode,
    .comment,
    .music-bar-volume {
      margin-left: 20px;
    }

    // 音量控制
    .volume-wrapper {
      margin-left: 20px;
      width: 150px;
    }
  }

  /*遮罩*/
  .mmPlayer-mask,
  .mmPlayer-bg {
    position: absolute;
    top: 0;
    right: 0;
    left: 0;
    bottom: 0;
  }

  .mmPlayer-mask {
    z-index: -1;
    background-color: @mask_color;
  }

  .mmPlayer-bg {
    z-index: -2;
    background-repeat: no-repeat;
    background-size: cover;
    background-position: 50%;
    filter: blur(12px);
    opacity: 0.7;
    transition: all 0.8s;
  }

  //当屏幕小于960时
  @media (max-width: 960px) {
    .music-right {
      display: none;
    }
  }
  //当屏幕小于768时
  @media (max-width: 768px) {
    & {
      padding: 75px 15px 5px 15px;
    }

    .music-content .music-left {
      .music-list {
        font-size: @font_size_medium;
      }
    }

    .music-bar {
      .music-bar-info span,
      .music-bar-volume .mmProgress {
        display: none;
      }
    }
  }
  //当屏幕小于520时
  @media (max-width: 520px) {
    .music-bar {
      position: relative;
      flex-direction: column;
      .music-bar-btns {
        width: 60%;
        margin-top: 15px;
        order: 2;
      }
      .music-music {
        padding-left: 0;
        order: 1;
      }
      & > i.mode {
        position: absolute;
        top: 40px;
        left: 5px;
        margin: 0;
      }
      .comment {
        position: absolute;
        top: 40px;
        right: 5px;
      }
      .music-bar-volume {
        display: none;
      }
    }
  }
}
</style>
