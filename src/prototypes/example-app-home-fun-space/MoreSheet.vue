<script setup lang="ts">
import { nextTick, onUnmounted, ref, watch } from 'vue'

import iconContributions from './assets/more-contributions.svg'
import iconDonate from './assets/more-donate.svg'
import iconEdit from './assets/more-edit.svg'
import iconFunSpace from './assets/more-fun-space.svg'
import iconPlaces from './assets/more-places.svg'
import iconSettings from './assets/more-settings.svg'
import iconTalk from './assets/more-talk.svg'
import iconUser from './assets/more-user.svg'
import iconWatchlist from './assets/more-watchlist.svg'

/**
 * Android "More" menu — a Material 3 modal bottom sheet.
 * Figma: Fun Space file, node 14045:10253 ("From the community tab").
 */

export type MoreSheetItemId =
  | 'user'
  | 'contributions'
  | 'edit'
  | 'talk'
  | 'watchlist'
  | 'fun-space'
  | 'places'
  | 'settings'
  | 'donate'

interface MoreSheetItem {
  id: MoreSheetItemId
  label: string
  icon: string
  badge?: string
}

const items: MoreSheetItem[] = [
  { id: 'user', label: 'CreativeMind', icon: iconUser },
  { id: 'contributions', label: 'Contributions', icon: iconContributions },
  { id: 'edit', label: 'Edit', icon: iconEdit },
  { id: 'talk', label: 'Talk', icon: iconTalk },
  { id: 'watchlist', label: 'Watchlist', icon: iconWatchlist },
  { id: 'fun-space', label: 'Fun space', icon: iconFunSpace, badge: 'New' },
  { id: 'places', label: 'Places', icon: iconPlaces },
  { id: 'settings', label: 'Settings', icon: iconSettings },
  { id: 'donate', label: 'Donate', icon: iconDonate },
]

const open = defineModel<boolean>('open', { default: false })

const emit = defineEmits<{
  select: [id: MoreSheetItemId]
}>()

const sheetEl = ref<HTMLElement | null>(null)
let returnFocusTo: HTMLElement | null = null

function close(): void {
  open.value = false
}

function onSelect(id: MoreSheetItemId): void {
  emit('select', id)
  close()
}

function onKeydown(event: KeyboardEvent): void {
  if (event.key === 'Escape') close()
}

watch(open, async (isOpen) => {
  if (isOpen) {
    returnFocusTo = document.activeElement instanceof HTMLElement ? document.activeElement : null
    window.addEventListener('keydown', onKeydown)
    await nextTick()
    // preventScroll: the sheet is still off-frame (translateY(100%)) here, so a plain
    // focus() scrolls the app frame up to reveal it and the content visibly jumps.
    sheetEl.value?.querySelector<HTMLElement>('button')?.focus({ preventScroll: true })
  } else {
    window.removeEventListener('keydown', onKeydown)
    returnFocusTo?.focus({ preventScroll: true })
    returnFocusTo = null
  }
})

onUnmounted(() => {
  window.removeEventListener('keydown', onKeydown)
})
</script>

<template>
  <Transition name="more-sheet">
    <div
      v-if="open"
      class="more-sheet"
    >
      <div
        class="more-sheet__scrim"
        aria-hidden="true"
        @click="close"
      />
      <div
        ref="sheetEl"
        class="more-sheet__sheet"
        data-theme="light"
        role="dialog"
        aria-modal="true"
        aria-label="More"
      >
        <ul class="more-sheet__list">
          <li
            v-for="item in items"
            :key="item.id"
          >
            <button
              type="button"
              class="more-sheet__item"
              @click="onSelect(item.id)"
            >
              <span
                class="more-sheet__icon"
                :style="{ '--more-sheet-icon': `url(&quot;${item.icon}&quot;)` }"
                aria-hidden="true"
              />
              <span class="more-sheet__label">{{ item.label }}</span>
              <span
                v-if="item.badge"
                class="more-sheet__badge"
              >{{ item.badge }}</span>
            </button>
          </li>
        </ul>
      </div>
    </div>
  </Transition>
</template>

<style scoped>
/* Covers the whole app frame, bottom nav included. The positioned ancestor
   is `.app-chrome-wrapper` (made `position: relative` by the page). */
.more-sheet {
  position: absolute;
  inset: 0;
  z-index: 100;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  /* The sliding sheet must never add scrollable overflow to the app frame. */
  overflow: clip;
}

.more-sheet__scrim {
  position: absolute;
  inset: 0;
  background-color: rgba(0, 0, 0, 0.3);
}

.more-sheet__sheet {
  position: relative;
  max-height: calc(100% - 72px);
  overflow-y: auto;
  padding-bottom: var(--spacing-25, 4px);
  border: 0.5px solid var(--border-color-subtle, #eaecf0);
  border-bottom: 0;
  border-radius: 16px 16px 0 0;
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
}

.more-sheet__list {
  margin: 0;
  padding: 0;
  list-style: none;
}

.more-sheet__item {
  display: flex;
  align-items: center;
  gap: var(--spacing-75, 12px);
  width: 100%;
  min-height: 48px;
  box-sizing: border-box;
  padding: var(--spacing-75, 12px) var(--spacing-100, 16px);
  border: 0;
  border-top: 0.5px solid var(--border-color-subtle, #eaecf0);
  background: transparent;
  color: inherit;
  font: inherit;
  font-size: 16px;
  line-height: 24px;
  text-align: start;
  cursor: pointer;
}

.more-sheet__item:hover {
  background-color: var(--background-color-interactive-subtle--hover, rgba(0, 0, 0, 0.04));
}

.more-sheet__item:active {
  background-color: var(--background-color-interactive-subtle--active, rgba(0, 0, 0, 0.08));
}

.more-sheet__item:focus-visible {
  outline: 2px solid var(--outline-color-progressive--focus, #36c);
  outline-offset: -2px;
}

/* The Figma SVGs, painted with the text colour so they follow light/dark. */
.more-sheet__icon {
  flex-shrink: 0;
  width: 24px;
  height: 24px;
  background-color: var(--color-subtle, #54595d);
  mask: var(--more-sheet-icon) center / 24px 24px no-repeat;
}

.more-sheet__label {
  min-width: 0;
}

.more-sheet__badge {
  display: inline-flex;
  align-items: center;
  height: 20px;
  padding-inline: var(--spacing-25, 4px);
  border-radius: 100px;
  background-color: rgba(51, 102, 204, 0.05);
  color: var(--color-progressive, #36c);
  font-size: 12px;
  font-weight: 500;
  line-height: 16px;
  letter-spacing: 0.4px;
}

/* Material 3 motion: enter with emphasized-decelerate, exit with
   emphasized-accelerate; the scrim fades alongside the sheet. */
.more-sheet-enter-active,
.more-sheet-enter-active .more-sheet__scrim,
.more-sheet-enter-active .more-sheet__sheet {
  transition-duration: 400ms;
  transition-timing-function: cubic-bezier(0.05, 0.7, 0.1, 1);
}

.more-sheet-leave-active,
.more-sheet-leave-active .more-sheet__scrim,
.more-sheet-leave-active .more-sheet__sheet {
  transition-duration: 200ms;
  transition-timing-function: cubic-bezier(0.3, 0, 0.8, 0.15);
}

.more-sheet__scrim {
  transition-property: opacity;
}

.more-sheet__sheet {
  transition-property: transform;
}

.more-sheet-enter-from .more-sheet__scrim,
.more-sheet-leave-to .more-sheet__scrim {
  opacity: 0;
}

.more-sheet-enter-from .more-sheet__sheet,
.more-sheet-leave-to .more-sheet__sheet {
  transform: translateY(100%);
}

@media (prefers-reduced-motion: reduce) {
  .more-sheet-enter-active,
  .more-sheet-enter-active .more-sheet__scrim,
  .more-sheet-enter-active .more-sheet__sheet,
  .more-sheet-leave-active,
  .more-sheet-leave-active .more-sheet__scrim,
  .more-sheet-leave-active .more-sheet__sheet {
    transition-duration: 1ms;
  }
}
</style>
