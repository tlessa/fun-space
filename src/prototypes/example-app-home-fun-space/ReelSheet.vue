<script setup lang="ts">
import { computed, nextTick, onUnmounted, ref, watch } from 'vue'

/**
 * Material 3 modal bottom sheet that plays an Instagram reel.
 * Dismiss per M3: swipe down on the drag handle, tap the scrim, tap the
 * handle (single-detent sheet, so tapping it closes), or press Escape.
 */

/** Reel shortcode to play; `null` closes the sheet. */
const reel = defineModel<string | null>('reel', { default: null })

const open = computed(() => reel.value !== null)
/** Kept through the leave transition so the iframe doesn't blank mid-slide. */
const shownReel = ref<string | null>(null)
const embedUrl = computed(() =>
  shownReel.value ? `https://www.instagram.com/reel/${shownReel.value}/embed/` : '',
)

const sheetEl = ref<HTMLElement | null>(null)
const handleEl = ref<HTMLElement | null>(null)
let returnFocusTo: HTMLElement | null = null

function close(): void {
  reel.value = null
}

function onKeydown(event: KeyboardEvent): void {
  if (event.key === 'Escape') close()
}

watch(reel, async (value, previous) => {
  if (value !== null) {
    shownReel.value = value
    if (previous !== null) return
    returnFocusTo = document.activeElement instanceof HTMLElement ? document.activeElement : null
    window.addEventListener('keydown', onKeydown)
    await nextTick()
    // preventScroll: the sheet is still off-frame (translateY(100%)) here, so a plain
    // focus() scrolls the app frame up to reveal it and the content visibly jumps.
    handleEl.value?.focus({ preventScroll: true })
  } else {
    // dragOffset is reset in onAfterLeave so a drag-dismiss exits from where it was let go.
    window.removeEventListener('keydown', onKeydown)
    returnFocusTo?.focus({ preventScroll: true })
    returnFocusTo = null
  }
})

onUnmounted(() => {
  window.removeEventListener('keydown', onKeydown)
})

/* Drag to dismiss. The reel iframe swallows pointer events, so the drag
   lives on the handle area at the top of the sheet. */
const DISMISS_FRACTION = 0.25
const DISMISS_VELOCITY = 0.5 // px per ms — a quick flick closes even if short
const TAP_SLOP = 6

const dragOffset = ref(0)
const dragging = ref(false)
let drag: { pointerId: number; startY: number; lastY: number; lastTime: number; velocity: number } | null = null

function onHandlePointerDown(event: PointerEvent): void {
  if (event.button !== 0) return
  drag = { pointerId: event.pointerId, startY: event.clientY, lastY: event.clientY, lastTime: event.timeStamp, velocity: 0 }
  dragging.value = true
  ;(event.currentTarget as HTMLElement).setPointerCapture(event.pointerId)
}

function onHandlePointerMove(event: PointerEvent): void {
  if (!drag || event.pointerId !== drag.pointerId) return
  const dt = event.timeStamp - drag.lastTime
  if (dt > 0) drag.velocity = (event.clientY - drag.lastY) / dt
  drag.lastY = event.clientY
  drag.lastTime = event.timeStamp
  // Downward only — the sheet is already at its single, full-height detent.
  dragOffset.value = Math.max(0, event.clientY - drag.startY)
}

function onHandlePointerUp(event: PointerEvent): void {
  if (!drag || event.pointerId !== drag.pointerId) return
  const distance = event.clientY - drag.startY
  const height = sheetEl.value?.offsetHeight ?? 1
  const { velocity } = drag
  drag = null
  dragging.value = false

  if (Math.abs(distance) < TAP_SLOP) {
    close()
  } else if (distance > height * DISMISS_FRACTION || velocity > DISMISS_VELOCITY) {
    close()
  } else {
    dragOffset.value = 0 // settle back open
  }
}

function onAfterLeave(): void {
  shownReel.value = null
  dragOffset.value = 0
}

function onHandleKeydown(event: KeyboardEvent): void {
  if (event.key === 'Enter' || event.key === ' ') {
    event.preventDefault()
    close()
  }
}

const sheetStyle = computed(() => {
  if (dragging.value) return { transform: `translateY(${dragOffset.value}px)`, transition: 'none' }
  if (dragOffset.value) return { transform: `translateY(${dragOffset.value}px)` }
  return undefined
})

/** Scrim fades as the sheet is dragged away. */
const scrimStyle = computed(() => {
  const height = sheetEl.value?.offsetHeight
  if (!dragOffset.value || !height) return undefined
  return { opacity: String(Math.max(0, 1 - dragOffset.value / height)), transition: dragging.value ? 'none' : undefined }
})
</script>

<template>
  <Transition
    name="reel-sheet"
    @after-leave="onAfterLeave"
  >
    <div
      v-if="open"
      class="reel-sheet"
    >
      <div
        class="reel-sheet__scrim"
        :style="scrimStyle"
        aria-hidden="true"
        @click="close"
      />
      <div
        ref="sheetEl"
        class="reel-sheet__sheet"
        :style="sheetStyle"
        data-theme="light"
        role="dialog"
        aria-modal="true"
        aria-label="Social video"
      >
        <div
          ref="handleEl"
          class="reel-sheet__handle-area"
          role="button"
          tabindex="0"
          aria-label="Close video"
          @pointerdown="onHandlePointerDown"
          @pointermove="onHandlePointerMove"
          @pointerup="onHandlePointerUp"
          @pointercancel="onHandlePointerUp"
          @keydown="onHandleKeydown"
        >
          <span class="reel-sheet__handle" />
        </div>
        <iframe
          v-if="shownReel"
          class="reel-sheet__frame"
          :src="embedUrl"
          title="Wikipedia on Instagram"
          allow="autoplay; encrypted-media; picture-in-picture; clipboard-write"
          allowfullscreen
        />
      </div>
    </div>
  </Transition>
</template>

<style scoped>
/* Same containing block as MoreSheet: covers the whole app frame, bottom nav
   included (`.app-chrome-wrapper` is made `position: relative` by the page). */
.reel-sheet {
  position: absolute;
  inset: 0;
  z-index: 100;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  /* The sliding sheet must never add scrollable overflow to the app frame. */
  overflow: clip;
}

.reel-sheet__scrim {
  position: absolute;
  inset: 0;
  background-color: rgba(0, 0, 0, 0.32);
}

/* M3 modal sheet on compact screens: full width, 72dp top margin, 28dp top corners. */
.reel-sheet__sheet {
  position: relative;
  display: flex;
  flex-direction: column;
  height: calc(100% - 72px);
  border-radius: 28px 28px 0 0;
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
  overflow: hidden;
}

/* 48dp touch target around the 32×4dp M3 drag handle. */
.reel-sheet__handle-area {
  display: flex;
  flex-shrink: 0;
  align-items: center;
  justify-content: center;
  height: 48px;
  touch-action: none;
  cursor: grab;
}

.reel-sheet__handle-area:active {
  cursor: grabbing;
}

.reel-sheet__handle-area:focus-visible {
  outline: 2px solid var(--outline-color-progressive--focus, #36c);
  outline-offset: -4px;
  border-radius: 28px 28px 0 0;
}

.reel-sheet__handle {
  width: 32px;
  height: 4px;
  border-radius: 2px;
  background-color: var(--color-subtle, #54595d);
  opacity: 0.4;
}

.reel-sheet__frame {
  flex: 1;
  width: 100%;
  border: 0;
}

/* Material 3 motion, matching MoreSheet: enter emphasized-decelerate,
   exit emphasized-accelerate; the scrim fades alongside the sheet. */
.reel-sheet-enter-active,
.reel-sheet-enter-active .reel-sheet__scrim,
.reel-sheet-enter-active .reel-sheet__sheet {
  transition-duration: 400ms;
  transition-timing-function: cubic-bezier(0.05, 0.7, 0.1, 1);
}

.reel-sheet-leave-active,
.reel-sheet-leave-active .reel-sheet__scrim,
.reel-sheet-leave-active .reel-sheet__sheet {
  transition-duration: 200ms;
  transition-timing-function: cubic-bezier(0.3, 0, 0.8, 0.15);
}

.reel-sheet__scrim {
  transition: opacity 200ms cubic-bezier(0.2, 0, 0, 1);
}

/* Settling back open after a short drag uses M3 standard easing. */
.reel-sheet__sheet {
  transition: transform 300ms cubic-bezier(0.2, 0, 0, 1);
}

.reel-sheet-enter-from .reel-sheet__scrim,
.reel-sheet-leave-to .reel-sheet__scrim {
  opacity: 0 !important;
}

.reel-sheet-enter-from .reel-sheet__sheet,
.reel-sheet-leave-to .reel-sheet__sheet {
  transform: translateY(100%) !important;
}

@media (prefers-reduced-motion: reduce) {
  .reel-sheet__scrim,
  .reel-sheet__sheet,
  .reel-sheet-enter-active,
  .reel-sheet-enter-active .reel-sheet__scrim,
  .reel-sheet-enter-active .reel-sheet__sheet,
  .reel-sheet-leave-active,
  .reel-sheet-leave-active .reel-sheet__scrim,
  .reel-sheet-leave-active .reel-sheet__sheet {
    transition-duration: 1ms;
  }
}
</style>
