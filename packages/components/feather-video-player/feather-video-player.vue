<script setup lang="ts">
import { ref, nextTick, watch, onMounted, onUnmounted } from 'vue'
import { useThrottleFn } from 'v3hooks'
import vueDanmaku from 'vue3-danmaku'
import Hls from 'hls.js'
import Flv from 'flv.js'
import './css/popover.css'

interface Episode {
  urls: string[],
  names?: string[],
  header?: string[],
  danmu?: string,
}

interface VideoProps {
  url: string | Episode,
  type?: 'm3u8' | 'flv' | 'normal',
}

const props = withDefaults(defineProps<VideoProps>(), {
  url: '',
  type: undefined,
})

const containerRef = ref()

// video 实例
const videoRef = ref()
const videoPreviewRef = ref()
// hls 实例
const hlsContainer = ref<null|Hls>()
const hlsPreviewContainer = ref<null|Hls>()
// flv 实例
const flvContainer = ref<null|Flv.Player>()
const flvPreviewContainer = ref<null|Flv.Player>()

const controlRef = ref()
const progressRef = ref()
const progressDoneRef = ref()
const progressBufferedRef = ref()
const progressBlockRef = ref()
const progressPreviewRef = ref()
const progressPreviewTimeRef = ref()

const controlDisappearTimeout = ref()

const videoType = ref<'video'|'live'>('video')
const controlShow = ref(false)
const progressBlockShow = ref(false)
const progressBlockBeenClick = ref(false)
const videoPlayState = ref(false)
const videoCurrentTime = ref(0)
const videoBufferedTime = ref(0)
const videoDragCurrentTime  = ref(0)
const videoPreviewCurrentWidth  = ref(0)
const videoDuration = ref(0)
const currentImage = ref('')
const currentImageShow = ref(false)


// 倍速
const speedList = [2, 1.5, 1.25, 1, 0.75, 0.5]
const currentSpeed = ref(1)

// 是否全屏
const videoFullscreen = ref(false)
const videoPictureInPicture = ref(false)

// 音量
const videoVolumeState = ref(true)
const videoVolumeValue = ref(100)

// 弹幕
const danmaku = ref()
const danmus = ref([])
const videoDanmuShow = ref(true)

// 循环
const videoLoop = ref(false)

// 镜像
const videoReverse = ref(false)

const url = ref('')
const episodeList = ref<{name: string, url: string}[]>([])

if (typeof props.url === 'string') {
  url.value = props.url
} else {
  if (props.url.urls.length > 0) {
    url.value = props.url.urls[0]
    const nameLength = props.url.names?.length
    episodeList.value = props.url.urls.map((episodeUrl, index) => {
      return {
        url: episodeUrl,
        // @ts-ignore
        name: 1 <= nameLength - index ? props.url.names[index] : '线路'+(index+1)
      }
    })
  }
}

watch(progressBlockBeenClick, (val) => {
  controlRef.value.style.pointerEvents = val ? 'none' : 'auto'
})
watch(videoCurrentTime, (val) => {
  // TODO 时间对应弹幕
})
watch(videoLoop, (val) => {
  if (videoRef.value) videoRef.value.loop = val
})
watch(videoReverse, (val) => {
  if (videoRef.value) videoRef.value.style.transform = val ? 'rotateY(180deg)' : ''
})

onMounted(() => {
  console.log('onMounted')
  nextTick(() => {
    if (containerRef.value) containerRef.value.style.height = containerRef.value.clientWidth * 0.5625 + 'px'
    playerInit()
    videoInit(videoRef.value)
  })
})

onUnmounted(() => {
  videoUninstall()
})


const playerInit = () => {
  window.addEventListener('resize', windowResizeHandle)
  window.addEventListener('fullscreenchange', windowFullscreenchangeHandle)
  window.addEventListener('enterpictureinpicture', windowEnterPictureInPictureHandle)
  window.addEventListener('leavepictureinpicture', windowEnterPictureInPictureHandle)

  containerRef.value.addEventListener('mouseenter', containerMouseenterHandle)
  containerRef.value.addEventListener('mousemove', containerMousemoveHandle)
  containerRef.value.addEventListener('mouseleave', containerMouseleaveHandle)

  videoRef.value.addEventListener('click', videoClickHandle)
  videoRef.value.addEventListener('loadedmetadata', videoLoadedmetadataHandle)
  videoRef.value.addEventListener('timeupdate', videoTimeupdateHandle)
  videoRef.value.addEventListener('progress', videoProgressHandle)
  videoRef.value.addEventListener('ended', videoEndedHandle)
  videoRef.value.addEventListener('canplay', videoCanplayHandle)
  videoRef.value.addEventListener('pause', videoPauseHandle)

  progressRef.value.addEventListener('mouseenter', progressMouseenterHandle)
  progressRef.value.addEventListener('mousedown', progressMouseDownHandle)
  progressRef.value.addEventListener('mousemove', progressMousemoveHandle)
  progressRef.value.addEventListener('mouseleave', progressMouseleaveHandle)
}

const videoInit = (element: HTMLVideoElement, type: 'normal' | 'preview' = 'normal') => {
  switch (props.type) {
    case 'm3u8':
    {
      hlsVideoInit(element, type)
      break
    }
    case 'flv':
    {
      flvVideoInit(element, type)
      break
    }
    default:
    {
      if (url.value.includes('.m3u8')) {
        hlsVideoInit(element, type)
      } else if (url.value.includes('.flv')) {
        flvVideoInit(element, type)
      } else {
        normalVideoInit(element, type)
      }
    }
  }
}

const hlsVideoInit = (element: HTMLVideoElement, type: 'normal' | 'preview' = 'normal') => {
  if (type === 'normal') {
    if (Hls.isSupported()) {
      hlsContainer.value = new Hls({
        liveDurationInfinity: true,
      })
      hlsContainer.value?.attachMedia(element)
      hlsContainer.value?.on(Hls.Events.MEDIA_ATTACHED, () => {
        console.log('初始化成功')
        hlsContainer.value?.loadSource(url.value)
        hlsContainer.value?.on(Hls.Events.MANIFEST_PARSED, (event, data) => {
          console.log('manifest loaded, found ' + data.levels.length + ' quality level')
        })

        hlsContainer.value?.on(Hls.Events.ERROR, function (event, data) {
          let errorType = data.type;
          let errorDetails = data.details;
          let errorFatal = data.fatal;

          console.log("视频错误信息回调")
          console.log(errorType)
          console.log(errorDetails)
          console.log(errorFatal)

          switch (data.details) {
            case Hls.ErrorDetails.FRAG_LOAD_ERROR:
              break;
            default:
              break;
          }
        })
      })
    } else {
      console.error('不支持M3U8')
    }
  } else {
    if (Hls.isSupported()) {
      hlsPreviewContainer.value = new Hls({
        liveDurationInfinity: true,
      })
      hlsPreviewContainer.value?.attachMedia(element)
      hlsPreviewContainer.value?.on(Hls.Events.MEDIA_ATTACHED, () => {
        console.log('Preview初始化成功')
        hlsPreviewContainer.value?.loadSource(url.value)
        hlsPreviewContainer.value?.on(Hls.Events.MANIFEST_PARSED, (event, data) => {
          console.log('Preview manifest loaded, found ' + data.levels.length + ' quality level')
        })

        hlsPreviewContainer.value?.on(Hls.Events.ERROR, function (event, data) {
          let errorType = data.type;
          let errorDetails = data.details;
          let errorFatal = data.fatal;

          console.log("视频错误信息回调")
          console.log(errorType)
          console.log(errorDetails)
          console.log(errorFatal)

          switch (data.details) {
            case Hls.ErrorDetails.FRAG_LOAD_ERROR:
              break;
            default:
              break;
          }
        })
      })
    } else {
      console.error('Preview 不支持M3U8')
    }
  }
}

const flvVideoInit = (element: HTMLVideoElement, type: 'normal' | 'preview' = 'normal') => {
  if (type === 'normal') {
    if (Flv.isSupported()) {
      flvContainer.value = Flv.createPlayer({
        type: 'flv',
        url: url.value,
      })
      flvContainer.value?.attachMediaElement(element)
      flvContainer.value?.load()

      flvContainer.value?.on(Flv.Events.ERROR, (errorType, errorDetail, errorInfo) => {
        console.log("视频错误信息回调")
        if (errorInfo.code === -1) {
          console.log('视频不是flv格式')
        }
      })
    } else {
      console.error('不支持FLV')
    }
  } else {
    if (Flv.isSupported()) {
      flvPreviewContainer.value = Flv.createPlayer({
        type: 'flv',
        url: url.value,
      })
      flvPreviewContainer.value?.attachMediaElement(element)
      flvPreviewContainer.value?.load()

      flvPreviewContainer.value?.on(Flv.Events.ERROR, (errorType, errorDetail, errorInfo) => {
        console.log("Preview 视频错误信息回调")
        if (errorInfo.code === -1) {
          console.log('Preview 视频不是flv格式')
        }
      })
    } else {
      console.error('Preview 不支持FLV')
    }
  }
}

const normalVideoInit = (element: HTMLVideoElement, type: 'normal' | 'preview' = 'normal') => {
  if (type === 'normal') {
    videoRef.value.src = url.value
    // videoRef.value.reload()
  } else {
    videoPreviewRef.value.src = url.value
    // videoPreviewRef.value.reload()
  }
}

const videoUninstall = () => {
  hlsContainer.value?.destroy()
  hlsPreviewContainer.value?.destroy()
  flvContainer.value?.destroy()
  flvPreviewContainer.value?.destroy()
  hlsContainer.value = null
  hlsPreviewContainer.value = null
  flvContainer.value = null
  flvPreviewContainer.value = null
}

const togglePlay = () => {
  if (videoRef.value.paused) {
    videoRef.value.play()
    videoPlayState.value = true
  } else {
    videoRef.value.pause()
    videoPlayState.value = false
  }
}

const toggleFullscreen = () => {
  if (videoFullscreen.value && document.fullscreenElement) {
    videoFullscreen.value = false
    document.exitFullscreen()
  } else {
    videoFullscreen.value = true
    containerRef.value.requestFullscreen()
  }
}

const togglePictureInPicture = () => {
  if (videoPictureInPicture.value && document.pictureInPictureElement) {
    videoPictureInPicture.value = false
    document.exitPictureInPicture()
  } else {
    videoPictureInPicture.value = true
    videoRef.value.requestPictureInPicture()
  }
}

const toggleDanmu = () => {
  if (videoDanmuShow.value) {
    videoDanmuShow.value = false
    danmaku.value.hide()
    danmaku.value.pause()
  } else {
    videoDanmuShow.value = true
    danmaku.value.show()
    danmaku.value.play()
  }
}

const toggleVolume = () => {
  if (videoVolumeState.value) {
    videoVolumeState.value = false
    videoRef.value.muted = true
  } else {
    videoVolumeState.value = true
    videoRef.value.muted = false
  }
}

const volumeChange = (a: number) => {
  videoVolumeValue.value = a
  videoVolumeState.value = a !== 0
  if (videoRef.value) {
    videoRef.value.volume = a / 100
    if (a === 0) {
      videoRef.value.muted = true
    }
  }
}

const speedSelect = (speed: number) => {
  currentSpeed.value = speed
  if (videoRef.value) videoRef.value.playbackRate = speed
}

const secondFormat = (value: number) => {
  let secondTime = Math.round(value)
  let minuteTime: number = 0
  let hourTime: number = 0
  if (secondTime > 60) {
    minuteTime = Math.floor(secondTime / 60)
    secondTime = Math.floor(secondTime % 60)

    if(minuteTime > 60) {
      hourTime = Math.floor(minuteTime / 60)
      minuteTime = Math.floor(minuteTime % 60)
    }
  }
  let time = "00:" + AddZero(secondTime)
  if (minuteTime > 0) time = AddZero(minuteTime) + ":" + AddZero(secondTime)
  if (hourTime > 0 || videoDuration.value >= 3600) time = AddZero(hourTime) + ":" + AddZero(minuteTime) + ":" + AddZero(secondTime)
  return time
}
const AddZero = (num: number) => {
  let text: string = num.toString()
  if (num < 10) {
    text = '0' + num
  }
  return text
}

const windowResizeHandle = () => {
  if (containerRef.value) {
    containerRef.value.style.height = containerRef.value.clientWidth * 0.5625 + 'px'
  }

  if (danmaku.value) {
    danmaku.value.resize()
  }

  if (progressRef.value) {
    const progressClientRect = progressRef.value.getBoundingClientRect()
    const progressWidth = progressClientRect.width
    progressBufferedRef.value.style.width = progressWidth * (videoBufferedTime.value / videoDuration.value) + 'px'
    progressDoneRef.value.style.width = progressWidth * (videoCurrentTime.value / videoDuration.value) + 'px'
    progressBlockRef.value.style.left = (progressWidth * (videoCurrentTime.value / videoDuration.value) - 12) + 'px'
  }

}

const windowFullscreenchangeHandle = () => {
  videoFullscreen.value = document.fullscreenElement !== null
}
const windowEnterPictureInPictureHandle = () => {
  videoPictureInPicture.value = document.pictureInPictureElement !== null
}
const videoClickHandle = (event: MouseEvent) => {
  event.preventDefault()
  togglePlay()
}
const videoLoadedmetadataHandle = () => {
  const duration = videoRef.value.duration
  if (duration === Infinity) {
    videoType.value = 'live'
  } else {
    videoDuration.value = duration
    videoInit(videoPreviewRef.value, 'preview')
  }
  danmaku.value.play()
}
const videoTimeupdateHandle = () => {
  if (videoType.value === 'video') {
    videoCurrentTime.value = videoRef.value.currentTime
    const progressClientRect = progressRef.value.getBoundingClientRect()
    const progressWidth = progressClientRect.width

    if (! progressBlockBeenClick.value) {
      progressDoneRef.value.style.width = progressWidth * (videoCurrentTime.value / videoDuration.value) + 'px'
      progressBlockRef.value.style.left = (progressWidth * (videoCurrentTime.value / videoDuration.value) - 12) + 'px'
    }
  }
}
const videoProgressHandle = () => {
  if (videoType.value === 'video') {
    const buffered = videoRef.value.buffered
    const progressClientRect = progressRef.value.getBoundingClientRect()
    const progressWidth = progressClientRect.width
    if (buffered.length > 0) {
      videoBufferedTime.value = buffered.end(buffered.length - 1)
      progressBufferedRef.value.style.width = progressWidth * (videoBufferedTime.value / videoDuration.value) + 'px'
    }
  }
}
const videoEndedHandle = () => {
  videoPlayState.value = false
  if (danmaku.value) danmaku.value.stop()
}
const videoCanplayHandle = () => {
  console.log('Canplay')
  if (videoRef.value.paused) videoRef.value.play()
  videoPlayState.value = true
}
const videoPauseHandle = () => {
  videoPlayState.value = false
}

const containerMouseenterHandle = () => {
  controlShow.value = true
  clearTimeout(controlDisappearTimeout.value)
  controlDisappear()
}
const containerMousemoveHandle = () => {
  controlShow.value = true
  clearTimeout(controlDisappearTimeout.value)
  controlDisappear()
}
const containerMouseleaveHandle = () => {
  if (! progressBlockBeenClick.value) controlShow.value = false
}
const controlDisappear = () => {
  controlDisappearTimeout.value = setTimeout(() => {
    controlShow.value = false
    clearTimeout(controlDisappearTimeout.value)
  }, 3200)
}
const progressMouseDownHandle = () => {
  progressBlockBeenClick.value = true
  document.addEventListener('mousemove', documentMousemoveHandle)
  document.addEventListener('mouseup', documentMouseupHandle)
}
const progressMousemoveHandle = (event: MouseEvent) => {
  const progressClientRect = progressRef.value.getBoundingClientRect()
  const progressLeft = progressClientRect.left
  const progressRight = progressClientRect.right
  let true_document_offset_x: number = 0
  if (event.pageX >= progressLeft && event.pageX <= progressRight)
    true_document_offset_x = event.pageX - progressLeft
  else if (event.pageX > progressRight)
    true_document_offset_x = progressRight - progressLeft

  videoPreviewCurrentWidth.value = true_document_offset_x

  if (videoType.value === 'video') {
    getPreviewBase64Run()
    progressPreviewRender()
  }
}

const progressPreviewRender = () => {
  currentImageShow.value = true
  const progressClientRect = progressRef.value.getBoundingClientRect()
  const progressPreviewClientRect = progressPreviewRef.value.getBoundingClientRect()
  const progressPreviewTimeClientRect = progressPreviewTimeRef.value.getBoundingClientRect()

  const progressWidth = progressClientRect.width
  const progressPreviewWidth = progressPreviewClientRect.width
  const progressPreviewTimeWidth = progressPreviewTimeClientRect.width

  let progress_preview_left: number = videoPreviewCurrentWidth.value
  if (progress_preview_left < progressPreviewWidth / 2) {
    progress_preview_left = 0
  } else if (progress_preview_left > progressWidth - progressPreviewWidth / 2) {
    progress_preview_left = progressWidth - progressPreviewWidth
  } else {
    progress_preview_left -= progressPreviewWidth / 2
  }
  progressPreviewRef.value.style.left = progress_preview_left + 'px'
  progressPreviewTimeRef.value.style.left = (progress_preview_left + (progressPreviewWidth - progressPreviewTimeWidth) / 2) + 'px'
  progressPreviewTimeRef.value.innerText = secondFormat((videoPreviewCurrentWidth.value / progressWidth) * videoDuration.value)
}

const progressMouseenterHandle = () => {
  const children = progressRef.value.children[0]
  if (children) children.style.height = '4px'
  progressBlockShow.value = true
}
const progressMouseleaveHandle = () => {
  const children = progressRef.value.children[0]
  if (! progressBlockBeenClick.value) {
    if (children) children.style.height = '2px'
    progressBlockShow.value = false
    currentImageShow.value = false
  }
}
const documentMousemoveHandle = (event: MouseEvent) => {
  if (progressBlockBeenClick.value) {
    clearTimeout(controlDisappearTimeout.value)
    const children = progressRef.value.children[0]
    if (children) children.style.height = '4px'
    progressBlockShow.value = true

    const progressClientRect = progressRef.value.getBoundingClientRect()
    const progressLeft = progressClientRect.left
    const progressRight = progressClientRect.right
    const progressWidth = progressClientRect.width

    let true_document_offset_x: number = 0
    if (event.pageX >= progressLeft && event.pageX <= progressRight)
      true_document_offset_x = event.pageX - progressLeft
    else if (event.pageX > progressRight)
      true_document_offset_x = progressRight - progressLeft

    videoPreviewCurrentWidth.value = true_document_offset_x
    if (videoType.value === 'video') {
      getPreviewBase64Run()
      progressPreviewRender()
    }

    progressDoneRef.value.style.width = true_document_offset_x + 'px'
    progressBlockRef.value.style.left = (true_document_offset_x - 12) + 'px'

    videoDragCurrentTime.value = (true_document_offset_x / progressWidth) * videoDuration.value
  }
}
const documentMouseupHandle = (event: MouseEvent) => {
  document.removeEventListener('mousemove', documentMousemoveHandle)
  document.removeEventListener('mouseup', documentMouseupHandle)
  documentMousemoveHandle(event)
  progressMouseleaveHandle()
  controlDisappear()
  // videoMouseleaveHandle()
  progressBlockBeenClick.value = false
  currentImageShow.value = false
  if (videoRef.value) {
    videoRef.value.currentTime = videoDragCurrentTime.value
    videoDragCurrentTime.value = 0
    if (videoRef.value.paused) videoRef.value.play()
    videoPlayState.value = true
  }
}

const { run: getPreviewBase64Run } = useThrottleFn(() => {
  getPreviewBase64(videoPreviewCurrentWidth.value).then((urlData) => {
    if (typeof urlData === 'string') currentImage.value = urlData
  })
}, 250)

const getPreviewBase64 = (currentWidth: number) => {
  return new Promise(resolve => {
    const progressClientRect = progressRef.value.getBoundingClientRect()
    const progressWidth = progressClientRect.width
    let video = videoPreviewRef.value
    video.currentTime = videoDuration.value * currentWidth / progressWidth
    video.addEventListener('seeked', () => {
      let canvas = document.createElement("canvas"),
          width = video.width,
          height = video.width * video.videoHeight / video.videoWidth
      canvas.width = width
      canvas.height = height
      canvas.getContext("2d")?.drawImage(video, 0, 0, width, height)
      resolve(canvas.toDataURL('image/jpeg'))
    })
  })
}

const chooseEpisode = (index: number) => {
  url.value = episodeList.value[index].url
  videoUninstall()
  videoInit(videoRef.value)
  videoInit(videoPreviewRef.value, 'preview')
}
</script>

<template>
  <div class="feather-video-container" ref="containerRef">
    <video
        class="feather-video"
        ref="videoRef"
        :src="url"
    ></video>
    <video class="feather-video-preview" :src="url" ref="videoPreviewRef" preload="none" crossOrigin="anonymous" width="960" height="540"></video>
    <vue-danmaku
        class="feather-video-danmaku"
        ref="danmaku"
        :danmus="danmus"
        :autoplay="false"
        :speeds="120"
        :randomChannel="true"
        :top="8"
    />
    <transition name="fade">
      <div class="feather-video-control" ref="controlRef" v-show="controlShow">
        <div class="video-control-mask"></div>
        <div class="video-progress-box" ref="progressRef" v-show="videoType === 'video'">
          <div class="video-progress">
            <div class="video-progress-has-done" ref="progressDoneRef"></div>
            <div class="video-progress-has-buffered" ref="progressBufferedRef"></div>
            <div class="video-progress-not-yet"></div>
<!--            <div class="video-progress-block" ref="progressBlockRef" v-show="progressBlockShow"></div>-->
            <img class="video-progress-block" src="./images/feather-icon-48.png" ref="progressBlockRef" v-show="progressBlockShow" alt="block" />
            <img class="video-progress-preview" :src="currentImage" ref="progressPreviewRef" alt="video-progress-preview" v-show="currentImage && currentImageShow">
            <div class="video-progress-preview-time" ref="progressPreviewTimeRef" v-show="currentImage && currentImageShow"></div>
          </div>
        </div>
        <div class="video-action">
          <div class="video-play">
            <div class="video-play-icon">
              <img src="./icons/pause.png" alt="pause" @click="togglePlay" v-if="videoPlayState">
              <img src="./icons/play.png" alt="play" @click="togglePlay" v-else>
            </div>
            <div class="video-time" v-if="videoType === 'video'">
              {{ `${secondFormat(videoCurrentTime)} / ${secondFormat(videoDuration)}` }}
            </div>
            <div class="video-live flex flex-row justify-center items-center" v-if="videoType === 'live'">Live</div>
          </div>
          <div class="video-space">
          </div>
          <div class="video-setting">
            <div class="video-action-icon">
              <el-tooltip
                  class="box-item"
                  effect="dark"
                  content="关闭弹幕"
                  placement="top"
                  :teleported="false"
                  v-if="videoDanmuShow"
              >
                <img style="width: 28px; height: 30px;" src="./icons/danmu-off.png" alt="danmu" @click="toggleDanmu">
              </el-tooltip>
              <el-tooltip
                  class="box-item"
                  effect="dark"
                  content="打开弹幕"
                  placement="top"
                  :teleported="false"
                  v-else
              >
                <img style="width: 28px; height: 30px;" src="./icons/danmu-on.png" alt="danmu" @click="toggleDanmu">
              </el-tooltip>
            </div>
            <el-popover
                width="160px"
                trigger="hover"
                effect="dark"
                placement="top"
                :show-arrow="false"
                :teleported="false"
                popper-class="volume-popover"
                v-if="episodeList && episodeList.length > 0"
            >
              <template #reference>
                <div class="video-action-icon">
                  <img src="./icons/list.png" alt="list" @click="toggleVolume">
                </div>
              </template>
              <div class="selection-list">
                <div v-bind:class="`selection-item flex flex-row justify-center ${url === item.url ? 'selection-item-select' : ''}`" v-for="(item, index) in episodeList" :key="index" @click="chooseEpisode(index)">{{ item.name }}</div>
              </div>
            </el-popover>
            <el-popover
                width="75px"
                trigger="hover"
                effect="dark"
                placement="top"
                :show-arrow="false"
                :teleported="false"
                popper-class="volume-popover"
            >
              <template #reference>
                <div class="video-action-icon">
                  <div class="multiple" v-if="currentSpeed === 1">倍速</div>
                  <div class="multiple" v-else>{{ currentSpeed }}x</div>
                </div>
              </template>
              <div class="speed-list">
                <div
                    class="speed-item flex flex-row justify-center"
                    v-bind:style="{color: currentSpeed === speed ? 'var(--el-color-primary)' : '#d8eaf5'}"
                    v-for="speed in speedList"
                    :key="speed"
                    @click="speedSelect(speed)"
                >
                  {{ speed }}x
                </div>
              </div>
            </el-popover>
            <el-popover
                width="20px"
                trigger="hover"
                effect="dark"
                placement="top"
                :show-arrow="false"
                :teleported="false"
                popper-class="volume-popover"
            >
              <template #reference>
                <div class="video-action-icon" v-if="videoVolumeState">
                  <img src="./icons/volume-on.png" alt="volume" @click="toggleVolume">
                </div>
                <div class="video-action-icon" v-else>
                  <img src="./icons/volume-off.png" alt="volume" @click="toggleVolume">
                </div>
              </template>
              <el-slider
                  v-model="videoVolumeValue"
                  vertical
                  height="120px"
                  size="small"
                  @input="volumeChange"
                  @change="volumeChange"
              />
            </el-popover>
            <el-popover
                width="120px"
                trigger="hover"
                effect="dark"
                placement="top"
                :show-arrow="false"
                :teleported="false"
                popper-class="volume-popover"
            >
              <template #reference>
                <div class="video-action-icon">
                  <img src="./icons/setting.png" alt="volume">
                </div>
              </template>
              <div class="setting-list">
                <div class="setting-item">
                  <el-switch
                      v-model="videoLoop"
                      inactive-text="洗脑循环"
                      size="small"
                  />
                </div>
                <div class="setting-item">
                  <el-switch
                      v-model="videoReverse"
                      inactive-text="镜像画面"
                      size="small"
                  />
                </div>
              </div>
            </el-popover>
            <div class="video-action-icon">
              <el-tooltip
                  class="box-item"
                  effect="dark"
                  content="返回原画"
                  placement="top"
                  :teleported="false"
                  v-if="videoPictureInPicture"
              >
                <img src="./icons/pip-back.png" alt="PictureInPicture" @click="togglePictureInPicture">
              </el-tooltip>
              <el-tooltip
                  class="box-item"
                  effect="dark"
                  content="开启画中画"
                  placement="top"
                  :teleported="false"
                  v-else
              >
                <img src="./icons/PictureInPicture.png" alt="PictureInPicture" @click="togglePictureInPicture">
              </el-tooltip>
            </div>
            <div class="video-action-icon">
              <el-tooltip
                  class="box-item"
                  effect="dark"
                  content="退出全屏"
                  placement="top-end"
                  :teleported="false"
                  v-if="videoFullscreen"
              >
                <img src="./icons/not-fullscreen.png" alt="fullscreen" @click="toggleFullscreen">
              </el-tooltip>
              <el-tooltip
                  class="box-item"
                  effect="dark"
                  content="进入全屏"
                  placement="top-end"
                  :teleported="false"
                  v-else
              >
                <img src="./icons/fullscreen.png" alt="fullscreen" @click="toggleFullscreen">
              </el-tooltip>
            </div>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<style scoped lang="less">
.feather-video-container {
  margin: 24px;
  width: calc(100% - 48px);
  position: relative;
  background-color: #010101;
  border-radius: 8px;
  overflow: hidden;
  .feather-video {
    width: 100%;
    height: 100%;
  }
  .feather-video-preview {
    position: absolute;
    top: -1000000;
    left: -1000000;
    display: none;
  }
  .feather-video-danmaku {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    user-select: none;
    pointer-events: none;
    :deep(.move) {
      //pointer-events: auto;
      height: 36px !important;
      line-height: 36px !important;
      font-size: 18px !important;
      font-weight: 600 !important;
    }
  }
  .feather-video-control {
    width: 100%;
    padding: 0 12px;
    position: absolute;
    left: 0;
    bottom: 0;
    height: 60px;
    user-select: none;
    //z-index: 49;
    .video-control-mask {
      height: 180px;
      width: 100%;
      position: absolute;
      bottom: 0;
      left: 0;
      background: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAC0CAYAAABVEkZPAAAACXBIWXMAAAsTAAALEwEAmpwYAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAACaSURBVHgBlZLbDsQgCETLYP//i5UVtkYhuG59IFzODGnKdZ0eQWMSaFfST2TL0bmHXS+bhoBzD2+ybMAJgp0WiXb2ghWDmYFaa8ZBH6O1Nv1KwrGB32zAnLg8nGXDau4ti4z60+yGiFhZNAvOIdzKQbmy9MIX2V/t3HM+HbZSNAt3JcgOzrQmc/A6cIhEgwH/v9L1xG8j77coPnsUOUD2dBlAAAAAAElFTkSuQmCC) repeat-x bottom;
      z-index: 50;
      pointer-events: none;
    }
    .video-progress-box {
      position: absolute;
      height: 20px;
      left: 16px;
      width: calc(100% - 32px);
      bottom: 48px !important;
      z-index: 52;
      cursor: pointer;
      display: flex;
      flex-direction: row;
      align-items: center;
      .video-progress {
        position: relative;
        height: 2px;
        width: 100%;
        background-color: rgba(148, 149, 151, .45);
        cursor: pointer;
        display: flex;
        flex-direction: row;
        border-radius: 1px;
        .video-progress-has-done {
          position: absolute;
          left: 0;
          height: 100%;
          z-index: 54;
          background-color: var(--el-color-primary);
        }
        .video-progress-has-buffered {
          position: absolute;
          left: 0;
          height: 100%;
          z-index: 53;
          background-color: #a4a7a9;
        }
        .video-progress-block {
          position: absolute;
          top: -10px;
          left: -10px;
          cursor: pointer;
          z-index: 55;
          width: 24px;
          height: 24px;
        }
        .video-progress-preview {
          position: absolute;
          bottom: 24px;
          width: 160px;
          height: 90px;
          border-radius: 5px;
        }
        .video-progress-preview-time {
          position: absolute;
          bottom: 26px;
          padding: 2px 4px;
          border-radius: 4px;
          color: #fff;
          background-color: rgba(0, 0, 0, 0.8);
          font-size: 12px;
          font-weight: 500;
        }
      }
    }
    .video-action {
      position: absolute;
      height: 36px;
      margin: 8px 0 0 0;
      z-index: 52;
      left: 12px;
      width: calc(100% - 24px);
      bottom: 8px;
      display: flex;
      flex-direction: row;
      justify-content: space-between;
      align-items: center;
      .video-play {
        display: flex;
        flex-direction: row;
        align-items: center;
        .video-play-icon {
          cursor: pointer;
          img {
            width: 24px;
            height: 24px;
          }
        }
        .video-time {
          margin-left: 18px;
          font-size: 13px;
          color: #fff;
        }
        .video-live {
          margin-left: 18px;
          width: 60px;
          height: 24px;
          border-radius: 6px;
          background-color: #9147ff;
          color: #fff;
          font-size: 16px;
          font-weight: 600;
        }
      }
      .video-space {}
      .video-setting {
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: flex-end;
        .video-action-icon {
          margin: 0 12px;
          cursor: pointer;
          img {
            width: 24px;
            height: 24px;
          }
          .multiple {
            width: 42px;
            max-width: 42px;
            color: #e9f0f4;
            font-size: 13px;
            font-weight: 600;
            text-align: center;
            &:hover {
              font-size: 14px;
            }
          }
        }
      }
    }
  }
  .fade-leave-from,
  .fade-enter-to {
    opacity: 1;
  }
  .fade-leave-active,
  .fade-enter-active {
    transition: all .6s;
  }
  .fade-leave-to,
  .fade-enter-from {
    opacity: 0;
  }
}
</style>

