<script setup lang="ts">
import { ref } from 'vue'
import { CdxButton, CdxIcon } from '@wikimedia/codex'
import { cdxIconEllipsis } from '@wikimedia/codex-icons'

import gameImageUrl from './assets/fun-space-game.png'
import playIconUrl from './assets/fun-space-play.svg'
/** Cover frames of the real reels, saved locally since Instagram's CDN URLs expire. */
import video1Url from './assets/fun-space-video-1.jpg' // instagram.com/reel/Df5i5tABWTJ
import video2Url from './assets/fun-space-video-2.jpg' // instagram.com/reel/Df4_jzKtHmM
import video3Url from './assets/fun-space-video-3.jpg' // instagram.com/reel/DfxL4ECx6TB

/** Placeholder games from the Figma "For you tab" frame (node 14044:5673). */
const games = [
  { id: 1, title: 'Game #1' },
  { id: 2, title: 'Game #2' },
  { id: 3, title: 'Game #3' },
]

interface SocialVideo {
  id: number
  /** The reel's cover frame. */
  image: string
  /** Instagram reel shortcode, played through Instagram's public embed. */
  reel: string
}

/** Social Videos page (Figma node 14044:5711). */
const videos: SocialVideo[] = [
  { id: 1, image: video1Url, reel: 'Df5i5tABWTJ' },
  { id: 2, image: video2Url, reel: 'Df4_jzKtHmM' },
  { id: 3, image: video3Url, reel: 'DfxL4ECx6TB' },
]

const pages = [
  { id: 'games', label: 'Games' },
  { id: 'videos', label: 'Social Videos' },
  { id: 'comic', label: 'Comic strip' },
] as const

const pagesEl = ref<HTMLElement | null>(null)
const activePage = ref(0)

function onPagesScroll(): void {
  const el = pagesEl.value
  if (!el || el.clientWidth === 0) return
  activePage.value = Math.round(el.scrollLeft / el.clientWidth)
}

function goToPage(index: number): void {
  const el = pagesEl.value
  if (el) el.scrollTo({ left: index * el.clientWidth, behavior: 'smooth' })
}

/** A mouse drag that swipes pages also ends in a click — don't treat that as a tap on a card. */
let pointerStart: { x: number; y: number } | null = null
const TAP_SLOP = 6

function onPointerDown(event: PointerEvent): void {
  pointerStart = { x: event.clientX, y: event.clientY }
}

function wasTap(event: MouseEvent): boolean {
  if (!pointerStart) return true
  return (
    Math.abs(event.clientX - pointerStart.x) < TAP_SLOP &&
    Math.abs(event.clientY - pointerStart.y) < TAP_SLOP
  )
}

/** The page opens the reel in a bottom sheet over the whole app frame (see ReelSheet). */
const emit = defineEmits<{
  'play-reel': [shortcode: string]
}>()

function play(video: SocialVideo, event: MouseEvent): void {
  if (wasTap(event)) emit('play-reel', video.reel)
}
</script>

<template>
  <section
    class="fun-space"
    aria-labelledby="fun-space-title"
  >
    <div
      ref="pagesEl"
      class="fun-space__pages"
      data-swipe-x
      @scroll="onPagesScroll"
      @pointerdown="onPointerDown"
    >
      <div
        v-for="(page, pageIndex) in pages"
        :key="page.id"
        class="fun-space__page"
        :aria-hidden="pageIndex !== activePage"
      >
        <div class="fun-space__title-row">
          <h2
            :id="pageIndex === 0 ? 'fun-space-title' : undefined"
            class="fun-space__title"
          >
            Fun Space
          </h2>
          <CdxButton
            weight="quiet"
            aria-label="More options"
          >
            <CdxIcon :icon="cdxIconEllipsis" />
          </CdxButton>
        </div>

        <ul
          v-if="page.id === 'games'"
          class="fun-space__games"
        >
          <li
            v-for="game in games"
            :key="game.id"
            class="fun-space__card"
          >
            <div class="fun-space__card-body">
              <p class="fun-space__card-title">
                {{ game.title }}
              </p>
              <CdxButton class="fun-space__play">
                Play today’s game
              </CdxButton>
            </div>
            <img
              class="fun-space__card-image"
              :src="gameImageUrl"
              alt=""
              draggable="false"
            >
          </li>
        </ul>

        <div
          v-else-if="page.id === 'videos'"
          class="fun-space__section"
        >
          <h3 class="fun-space__section-title">
            Social Videos
          </h3>
          <ul class="fun-space__videos">
            <li
              v-for="video in videos"
              :key="video.id"
            >
              <button
                type="button"
                class="fun-space__video"
                :aria-label="`Play video ${video.id}`"
                @click="play(video, $event)"
              >
                <img
                  class="fun-space__video-image"
                  :src="video.image"
                  alt=""
                  draggable="false"
                >
                <img
                  class="fun-space__video-play"
                  :src="playIconUrl"
                  width="24"
                  height="24"
                  alt=""
                  draggable="false"
                >
              </button>
            </li>
          </ul>
        </div>

        <!-- Comic strip: placeholder until there's a design / comic to show. -->
        <div
          v-else
          class="fun-space__section"
        >
          <h3 class="fun-space__section-title">
            Comic strip
          </h3>
          <div class="fun-space__comic" />
        </div>
      </div>
    </div>

    <!-- Same look as the Figma "Pagination for you" dots (8px, 12px apart), one per page. -->
    <div class="fun-space__pagination">
      <button
        v-for="(page, pageIndex) in pages"
        :key="page.id"
        type="button"
        class="fun-space__dot"
        :class="{ 'fun-space__dot--active': pageIndex === activePage }"
        :aria-label="`${page.label}, page ${pageIndex + 1} of ${pages.length}`"
        :aria-current="pageIndex === activePage ? 'true' : undefined"
        @click="goToPage(pageIndex)"
      />
    </div>
  </section>
</template>

<style scoped>
/* Fills one story slot of the For you feed. ForYouFeed's header + tabs
   float over the top, so content starts below them. */
.fun-space {
  position: relative;
  height: 100%;
  overflow: hidden;
  background-color: #196551;
  background-image: linear-gradient(to top, rgba(0, 0, 0, 0) 68.2%, rgba(0, 0, 0, 0.6) 90.6%);
  color: #f8f9fa;
}

/* Horizontal pager — same scroll-snap pattern as the feed's story slides. */
.fun-space__pages {
  display: flex;
  height: 100%;
  overflow-x: scroll;
  scroll-snap-type: x mandatory;
  scrollbar-width: none;
}

.fun-space__pages::-webkit-scrollbar {
  display: none;
}

.fun-space__page {
  display: flex;
  flex: 0 0 100%;
  flex-direction: column;
  gap: var(--spacing-100, 16px);
  box-sizing: border-box;
  padding: 132px var(--spacing-100, 16px) 0;
  scroll-snap-align: start;
}

.fun-space__title-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--spacing-50, 8px);
}

.fun-space__title {
  margin: 0;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-family: var(--font-family-serif);
  font-size: var(--font-size-xx-large, 1.5rem);
  font-weight: var(--font-weight-normal, 400);
  line-height: 32px;
  color: #f8f9fa;
}

.fun-space__title-row .cdx-button {
  flex-shrink: 0;
  color: #f8f9fa;
}

.fun-space__games {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin: 0;
  padding: 0;
  list-style: none;
}

.fun-space__card {
  display: flex;
  justify-content: space-between;
  gap: var(--spacing-100, 16px);
  height: 140px;
  box-sizing: border-box;
  padding: var(--spacing-100, 16px);
  border-radius: 16px;
  background-color: #202122;
  overflow: hidden;
}

.fun-space__card-body {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 21px;
  min-width: 0;
}

.fun-space__card-title {
  margin: 0;
  color: #eaecf0;
  font-size: var(--font-size-medium, 0.875rem);
  line-height: 20px;
  letter-spacing: 0.25px;
}

/* Outlined pill — only the delta from Codex's default button. */
.fun-space__play.cdx-button {
  min-height: 0;
  padding: 10px var(--spacing-100, 16px);
  border: var(--border-width-base, 1px) solid #f8f9fa;
  border-radius: 100px;
  background-color: transparent;
  color: #f8f9fa;
  font-size: var(--font-size-medium, 0.875rem);
  font-weight: 500;
  line-height: 20px;
  letter-spacing: 0.1px;
}

.fun-space__play.cdx-button:hover {
  background-color: rgba(248, 249, 250, 0.08);
}

.fun-space__play.cdx-button:active {
  background-color: rgba(248, 249, 250, 0.16);
}

.fun-space__card-image {
  flex-shrink: 0;
  width: 100px;
  height: 100px;
  border-radius: 12px;
  object-fit: cover;
  box-shadow: 0 0 12px rgba(0, 0, 0, 0.15);
}

/* Social Videos page (Figma 14044:5711) */
.fun-space__section {
  display: flex;
  flex-direction: column;
  margin-top: var(--spacing-50, 8px);
}

.fun-space__section-title {
  margin: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: var(--font-size-large, 1rem);
  font-weight: 600;
  line-height: 24px;
  letter-spacing: 0.15px;
  color: #f8f9fa;
}

.fun-space__videos {
  display: flex;
  gap: var(--spacing-50, 8px);
  margin: 0;
  padding: 0;
  list-style: none;
}

.fun-space__video {
  position: relative;
  display: block;
  width: 102px;
  height: 149px;
  padding: 0;
  border: 0;
  border-radius: 8px;
  overflow: hidden;
  background: none;
  cursor: pointer;
}

.fun-space__video:focus-visible {
  outline: 2px solid #f8f9fa;
  outline-offset: 2px;
}

/* Reel covers are 9:16; cover-fill crops them to the card's 102×149. */
.fun-space__video-image {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  pointer-events: none;
}

/* 24px play icon (Figma's is 42px), tucked into the bottom-right corner. */
.fun-space__video-play {
  position: absolute;
  inset-inline-end: var(--spacing-50, 8px);
  bottom: var(--spacing-50, 8px);
  display: block;
}

/* Comic strip placeholder — 3:1 strip area until real content lands. */
.fun-space__comic {
  aspect-ratio: 3 / 1;
  border: 1px dashed rgba(248, 249, 250, 0.4);
  border-radius: 8px;
}

.fun-space__pagination {
  position: absolute;
  inset-inline: 0;
  bottom: var(--spacing-50, 8px);
  display: flex;
  justify-content: center;
  gap: 12px;
}

/* Inactive dot matches the Figma asset: #f8f9fa at 0.5 × 0.8 opacity. */
.fun-space__dot {
  width: 8px;
  height: 8px;
  padding: 0;
  border: 0;
  border-radius: 50%;
  background-color: #f8f9fa;
  opacity: 0.4;
  cursor: pointer;
  transition: opacity 200ms cubic-bezier(0.2, 0, 0, 1);
}

.fun-space__dot--active {
  opacity: 1;
}

.fun-space__dot:focus-visible {
  outline: 2px solid #f8f9fa;
  outline-offset: 2px;
}
</style>
