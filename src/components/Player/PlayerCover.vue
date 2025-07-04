<template>
  <div class="player-cover-container">
    <Transition name="fade" mode="out-in">
      <div
        :key="`cover_pic--${music.getPlaySongData?.album?.pic ?? defaultCover}`"
        :class="[
          'pic',
          !music.getPlayState ? 'pause' : '',
          music.getLoadingState ? 'loading' : '',
        ]"
      >
        <img
          class="album"
          :src="
            music.getPlaySongData && music.getPlaySongData.album
          ? music.getPlaySongData.album.picUrl.replace(/^http:/, 'https:') +
          '?param=1024y1024'
          : '/images/pic/default.png'
          "
          alt="cover"
        />
      </div>
    </Transition>
    <div class="controls">
      <div class="song-info">
        <div class="text">
          <span class="name text-hidden">
            {{
              music.getPlaySongData
                ? music.getPlaySongData.name
                : $t("other.noSong")
            }}
          </span>
          <span
            v-if="music.getPlaySongData"
            class="artists text-hidden"
            @click="
              routerJump('/artist', {
                id: music.getPlaySongData.artist[0].id,
              })
            "
          >
            {{ music.getPlaySongData.artist[0].name }}
            </span>
        </div>
        <n-icon v-if="music.getPlaySongData && user.userLogin" 
         class="like" size="20" 
         :component="music.getSongIsLike(music.getPlaySongData.id)
          ? FavoriteRound
          : FavoriteBorderRound" 
         @click.stop="
          music.getSongIsLike(music.getPlaySongData.id)
            ? music.changeLikeList(music.getPlaySongData.id, false)
            : music.changeLikeList(music.getPlaySongData.id, true)" 
        />
      </div>
      <div class="progress-bar">
        <div class="progress-bar-content">
          <span class="time-text">{{
            music.getPlaySongTime.songTimePlayed
          }}</span>
          <div class="slider-container">
            <div v-if="qualityText" class="quality-badge">
              {{ qualityText }}
            </div>
            <vue-slider
              v-model="music.getPlaySongTime.barMoveDistance"
              @drag-start="music.setPlayState(false)"
              @drag-end="sliderDragEnd"
              @click.stop="
            songTimeSliderUpdate(music.getPlaySongTime.barMoveDistance)
              "
              :tooltip="'none'"
            />
          </div>
          <span class="time-text">{{
            music.getPlaySongTime.songTimeDuration
          }}</span>
        </div>
      </div>
      <div class="buttons">
        <n-icon
          :style="
            music.getPersonalFmMode
          ? 'opacity: 0.2;pointer-events: none;'
          : null
          "
          class="button-icon"
          :component="
            music.getPlaySongMode === 'random' ? ShuffleOne : PlayCycle
          "
          @click="
            music.setPlaySongMode(
              music.getPlaySongMode === 'random' ? 'normal' : 'random'
            )
          "
        />
        <n-icon
          v-if="!music.getPersonalFmMode"
          class="button-icon"
          :component="IconRewind"
          @click.stop="music.setPlaySongIndex('prev')"
        />
        <n-icon
          v-else
          class="button-icon dislike"
          :style="!user.userLogin ? 'opacity: 0.2;pointer-events: none;' : null"
          :component="ThumbDownRound"
          @click="music.setFmDislike(music.getPersonalFmData.id)"
        />
        <div class="play-state">
          <n-button text :focusable="false" :loading="music.getLoadingState">
            <template #icon>
              <n-icon
                :component="music.getPlayState ? IconPause : IconPlay"
                @click.stop="music.setPlayState(!music.getPlayState)"
              />
            </template>
          </n-button>
        </div>
        <n-icon
          class="button-icon"
          :component="IconForward"
          @click.stop="music.setPlaySongIndex('next')"
        />
        <n-icon
          class="button-icon"
          :component="PlayOnce"
          :style="
            music.getPlaySongMode === 'single' ? 'color: var(--primary-color)' : ''
          "
          @click="
            music.setPlaySongMode(
              music.getPlaySongMode === 'single' ? 'normal' : 'single'
            )
          "
        />
      </div>
      <div class="volume-control">
        <n-icon
          class="button-icon"
          :component="VolumeOffRound"
        />
        <vue-slider
          :tooltip="'none'"
          :min="0"
          :max="1"
          :interval="0.001"
          v-model="persistData.playVolume"
        />
        <n-icon
          class="button-icon"
          :component="VolumeUpRound"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import {
  MoreHorizRound,
  ThumbDownRound,
  VolumeOffRound,
  VolumeUpRound,
  FavoriteRound,
  FavoriteBorderRound,
} from "@vicons/material";
import { computed, onMounted } from "vue";
import IconForward from "./icons/IconForward.vue";
import IconRewind from "./icons/IconRewind.vue";
import IconPlay from "./icons/IconPlay.vue";
import IconPause from "./icons/IconPause.vue";
import { PlayCycle, PlayOnce, ShuffleOne } from "@icon-park/vue-next";
import { musicStore, userStore, settingStore } from "@/store";
import { storeToRefs } from "pinia";
import { useRouter } from "vue-router";
import { setSeek } from "@/utils/Player";
import VueSlider from "vue-slider-component";
import "vue-slider-component/theme/default.css";
import defaultCover from "/images/pic/default.png?url";
import gsap from "gsap";

const router = useRouter();
const music = musicStore();
const user = userStore();
const setting = settingStore();
const { persistData } = storeToRefs(music);

// 音质标签
const qualityText = computed(() => {
  const level = setting.songLevel;
  const qualityMap = {
    standard: "标准",
    higher: "较高",
    exhigh: "极高",
    lossless: "无损",
    hires: "Hi-Res",
  };
  return qualityMap[level] || null;
});

// 歌曲进度条更新
const sliderDragEnd = () => {
  songTimeSliderUpdate(music.getPlaySongTime.barMoveDistance);
  music.setPlayState(true);
};
const songTimeSliderUpdate = (val) => {
  if (typeof $player !== "undefined" && music.getPlaySongTime?.duration) {
    const currentTime = (music.getPlaySongTime.duration / 100) * val;
    setSeek($player, currentTime);
  }
};

// 页面跳转
const routerJump = (url, query) => {
  music.setBigPlayerState(false);
  router.push({
    path: url,
    query,
  });
};

// GSAP 动画
onMounted(() => {
  const buttons = document.querySelectorAll(".button-icon, .play-state");
  buttons.forEach((button) => {
    // 悬停动画
    button.addEventListener("mouseenter", () => {
      gsap.to(button, {
        scale: 1.1,
        duration: 0.2,
        ease: "power1.out",
      });
    });
    button.addEventListener("mouseleave", () => {
      gsap.to(button, {
        scale: 1,
        duration: 0.2,
        ease: "power1.inOut",
      });
    });
    // 点击动画
    button.addEventListener("mousedown", () => {
      gsap.to(button, {
        scale: 0.9,
        duration: 0.1,
        ease: "power1.in",
      });
    });
    button.addEventListener("mouseup", () => {
      gsap.to(button, {
        scale: 1.1,
        duration: 0.2,
        ease: "power1.out",
      });
    });
  });
});
</script>

<style lang="scss" scoped>
.player-cover-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2rem;
  .pic {
    position: relative;
    width: 50vh;
    height: 50vh;
    min-width: 250px;
    min-height: 250px;
    max-width: 360px;
    max-height: 360px;
    border-radius: 12px;
    transition: transform 0.5s ease-out, filter 0.5s ease-out;
    @media (max-width: 870px) {
      width: 40vh;
      height: 40vh;
    }
    &.pause {
      transform: scale(0.95);
    }
    &.loading {
      transform: scale(0.95);
      filter: grayscale(0.8);
    }
    .album {
      width: 100%;
      height: 100%;
      border-radius: 12px;
    }
  }
  .controls {
    width: 100%;
    max-width: 360px;
    display: flex;
    flex-direction: column;
    gap: 1.25rem;
    .song-info {
      display: flex;
      justify-content: space-between;
      align-items: center;
      color: var(--main-cover-color);
      .text {
        display: flex;
        flex-direction: column;
        gap: 0.25rem;
        .name {
          font-size: 1.5rem;
          font-weight: 600;
        }
        .artists {
          font-size: 1rem;
          opacity: 0.7;
          cursor: pointer;
            &:hover {
            opacity: 1;
          }
        }
      }
      .more-button {
        font-size: 1.75rem;
        cursor: pointer;
        opacity: 0.7;
        transition: opacity 0.2s ease;
        &:hover {
          opacity: 1;
        }
      }
    }
    .progress-bar {
      width: 100%;
      .progress-bar-content {
      display: flex;
      align-items: center;
        gap: 1rem;
        color: var(--main-cover-color);
        .time-text {
          font-size: 0.75rem;
          opacity: 0.7;
          min-width: 30px;
        }
        .slider-container {
          width: 100%;
          position: relative;
          .quality-badge {
            position: absolute;
            top: -20px;
            left: 50%;
            transform: translateX(-50%);
            background-color: rgba(255, 255, 255, 0.1);
            color: var(--main-cover-color);
            opacity: 0.8;
            font-size: 0.7rem;
            padding: 2px 6px;
            border-radius: 4px;
            white-space: nowrap;
          }
        }
      }
    }
    .buttons {
      display: flex;
      justify-content: space-around;
      align-items: center;
      .play-state {
        .n-button {
          font-size: 3rem;
          color: var(--main-cover-color);
        }
      }
      .button-icon {
        font-size: 1.5rem;
        color: var(--main-cover-color);
        opacity: 0.8;
        cursor: pointer;
        transition: opacity 0.2s ease, transform 0.1s ease-out;
        &:hover {
          opacity: 1;
        }
      }
    }
    .volume-control {
      display: flex;
      align-items: center;
      gap: 1rem;
      .button-icon {
        font-size: 1.25rem;
        color: var(--main-cover-color);
        opacity: 0.7;
      }
    }
    .vue-slider {
      width: 100% !important;
        cursor: pointer;
      :deep(.vue-slider-rail) {
        height: 5px;
        background-color: rgba(255, 255, 255, 0.2);
        border-radius: 2.5px;
        .vue-slider-process {
          background-color: var(--main-cover-color);
        }
        .vue-slider-dot {
          display: none !important;
        }
      }
    }
  }
}
</style>
