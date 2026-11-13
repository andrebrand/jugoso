<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import wordData from './data/words.json'

type Syllable = {
  label: string
  link: string
}

type Word = {
  syllable: Syllable[]
}

type SelectionItem = {
  word: string
  syllableIndex: number
}

const words: Record<string, Word> = wordData
const wordEntries = Object.entries(words)

function findSyllable(item: SelectionItem) {
  return words[item.word]?.syllable[item.syllableIndex]
}

function decodeSelection(hash: string): { items: SelectionItem[]; error: string } {
  if (!hash.startsWith('#mix=')) return { items: [], error: '' }

  const token = hash.slice(5)
  try {
    if (!token || token.length > 4096 || !/^[A-Za-z0-9_-]+$/.test(token)) {
      throw new Error('Invalid shared mix')
    }

    const base64 = token.replace(/-/g, '+').replace(/_/g, '/')
    const binary = atob(base64.padEnd(Math.ceil(base64.length / 4) * 4, '='))
    const bytes = Uint8Array.from(binary, (character) => character.charCodeAt(0))
    const parsed: unknown = JSON.parse(new TextDecoder().decode(bytes))

    if (!Array.isArray(parsed) || parsed.length > 80) throw new Error('Invalid shared mix')

    const items: SelectionItem[] = []
    for (const entry of parsed) {
      if (
        !Array.isArray(entry) ||
        typeof entry[0] !== 'string' ||
        !Number.isInteger(entry[1]) ||
        !Object.prototype.hasOwnProperty.call(words, entry[0]) ||
        !words[entry[0]]?.syllable[entry[1]]
      ) {
        throw new Error('Invalid shared mix')
      }

      items.push({ word: entry[0], syllableIndex: entry[1] })
    }

    return { items, error: '' }
  } catch {
    return { items: [], error: 'This shared mix link is invalid. Choose syllables to start a new one.' }
  }
}

function encodeSelection(items: SelectionItem[]) {
  const serialized = JSON.stringify(items.map(({ word, syllableIndex }) => [word, syllableIndex]))
  const bytes = new TextEncoder().encode(serialized)
  let binary = ''
  for (const byte of bytes) binary += String.fromCharCode(byte)
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/g, '')
}

const initialState = decodeSelection(window.location.hash)
const selection = ref<SelectionItem[]>(initialState.items)
const sharedRevealPending = ref(initialState.items.length > 0)
const playbackIndex = ref(0)
const draggedSelectionIndex = ref<number | null>(null)
const isPlaying = ref(false)
const playbackMessage = ref(initialState.error)
const videoElement = ref<HTMLVideoElement | null>(null)

const activeSelection = computed(() => selection.value[playbackIndex.value])
const activeSyllable = computed(() =>
  activeSelection.value ? findSyllable(activeSelection.value) : undefined,
)
const selectedSyllables = computed(() =>
  selection.value.flatMap((item, index) => {
    const syllable = findSyllable(item)
    return syllable ? [{ ...item, syllable, position: index + 1 }] : []
  }),
)
const syllableCount = computed(() =>
  selection.value.reduce((count, item) => count + (findSyllable(item) ? 1 : 0), 0),
)

function wordName(word: string) {
  return word.charAt(0).toUpperCase() + word.slice(1)
}

function syncSelectionToUrl() {
  const hash = selection.value.length ? `#mix=${encodeSelection(selection.value)}` : ''
  window.history.replaceState(window.history.state, '', `${window.location.pathname}${window.location.search}${hash}`)
}

async function playCurrentClip(waitForRender = false) {
  if (waitForRender) await nextTick()
  const player = videoElement.value
  if (!player) {
    isPlaying.value = false
    playbackMessage.value = 'The video player is not available. Try again.'
    return
  }

  try {
    await player.play()
    playbackMessage.value = ''
  } catch (error) {
    isPlaying.value = false
    playbackMessage.value =
      error instanceof DOMException && error.name === 'NotAllowedError'
        ? 'Playback was blocked. Press play to hear this shared mix.'
        : 'This clip could not be played. Try again or choose another syllable.'
  }
}

async function playMix() {
  if (!selection.value.length) return
  const waitForRender = playbackIndex.value !== 0
  playbackIndex.value = 0
  playbackMessage.value = ''
  isPlaying.value = true
  await playCurrentClip(waitForRender)
}

async function handleClipEnded() {
  if (playbackIndex.value < selection.value.length - 1) {
    playbackIndex.value += 1
    await playCurrentClip(true)
    return
  }

  isPlaying.value = false
  if (sharedRevealPending.value) {
    sharedRevealPending.value = false
    playbackMessage.value = 'Your shared mix is ready.'
  } else {
    playbackMessage.value = 'Mix complete. Play it again or build a new one.'
  }
}

function handleClipError() {
  isPlaying.value = false
  playbackMessage.value = 'This clip could not be loaded. Check the video link in the word catalog.'
}

function addSyllable(word: string, syllableIndex: number) {
  videoElement.value?.pause()
  selection.value = [...selection.value, { word, syllableIndex }]
  playbackIndex.value = selection.value.length - 1
  sharedRevealPending.value = false
  isPlaying.value = false
  playbackMessage.value = `${wordName(word)} · ${words[word]?.syllable[syllableIndex]?.label ?? ''} added to your mix.`
  syncSelectionToUrl()
}

function removeSyllable(index: number) {
  videoElement.value?.pause()
  selection.value = selection.value.filter((_, itemIndex) => itemIndex !== index)
  playbackIndex.value = Math.min(playbackIndex.value, Math.max(selection.value.length - 1, 0))
  sharedRevealPending.value = false
  isPlaying.value = false
  playbackMessage.value = selection.value.length ? 'Syllable removed from your mix.' : 'Mix cleared.'
  syncSelectionToUrl()
}

function moveSyllable(fromIndex: number, toIndex: number) {
  if (
    fromIndex === toIndex ||
    fromIndex < 0 ||
    toIndex < 0 ||
    fromIndex >= selection.value.length ||
    toIndex >= selection.value.length
  ) {
    return
  }

  const reordered = [...selection.value]
  const [item] = reordered.splice(fromIndex, 1)
  if (!item) return
  reordered.splice(toIndex, 0, item)
  selection.value = reordered

  if (playbackIndex.value === fromIndex) {
    playbackIndex.value = toIndex
  } else if (fromIndex < playbackIndex.value && toIndex >= playbackIndex.value) {
    playbackIndex.value -= 1
  } else if (fromIndex > playbackIndex.value && toIndex <= playbackIndex.value) {
    playbackIndex.value += 1
  }

  playbackMessage.value = 'Mix order updated.'
  syncSelectionToUrl()
}

function startDragging(event: DragEvent, index: number) {
  draggedSelectionIndex.value = index
  if (event.dataTransfer) {
    event.dataTransfer.effectAllowed = 'move'
    event.dataTransfer.setData('text/plain', String(index))
  }
}

function dropSyllable(event: DragEvent, targetIndex: number) {
  event.preventDefault()
  const fromIndex = Number(event.dataTransfer?.getData('text/plain'))
  if (Number.isInteger(fromIndex)) moveSyllable(fromIndex, targetIndex)
  draggedSelectionIndex.value = null
}

function clearMix() {
  videoElement.value?.pause()
  selection.value = []
  playbackIndex.value = 0
  sharedRevealPending.value = false
  isPlaying.value = false
  playbackMessage.value = 'Mix cleared. Pick syllables below to build a new one.'
  syncSelectionToUrl()
}

async function copyShareLink() {
  try {
    await navigator.clipboard.writeText(window.location.href)
    playbackMessage.value = 'Share link copied to clipboard.'
  } catch {
    playbackMessage.value = 'Could not copy the link. Copy it from your browser address bar instead.'
  }
}

function restoreFromUrl() {
  const state = decodeSelection(window.location.hash)
  videoElement.value?.pause()
  selection.value = state.items
  playbackIndex.value = 0
  isPlaying.value = false
  sharedRevealPending.value = state.items.length > 0
  playbackMessage.value = state.error

  if (sharedRevealPending.value) void playMix()
}

onMounted(() => {
  window.addEventListener('hashchange', restoreFromUrl)
  if (sharedRevealPending.value) void playMix()
})

onBeforeUnmount(() => {
  window.removeEventListener('hashchange', restoreFromUrl)
})
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Jugoso home" @click.prevent="clearMix">
        <span class="brand-mark" aria-hidden="true">j</span>
        <span>jugoso<span class="brand-period">.</span></span>
      </a>
      <div class="topbar-right">
        <span class="availability"><span class="availability-dot"></span> YOUR SPANISH SOUND LAB</span>
        <button
          v-if="selection.length"
          class="icon-button clear-top"
          type="button"
          aria-label="Clear mix"
          title="Clear mix"
          @click="clearMix"
        >
          <svg viewBox="0 0 24 24" aria-hidden="true">
            <path d="M4 7h16M10 11v6m4-6v6M6 7l1 13h10l1-13M9 7V4h6v3" />
          </svg>
        </button>
      </div>
    </header>

    <main>
      <section class="intro" aria-labelledby="page-title">
        <div class="eyebrow"><span class="eyebrow-line"></span> PRONUNCIATION, PIECE BY PIECE</div>
        <h1 id="page-title">Find your <span>Spanish flow.</span></h1>
        <p>Pick a sound. Make it yours. Build a little Spanish, one syllable at a time.</p>
      </section>

      <section class="studio" aria-label="Pronunciation studio">
        <div class="studio-topline">
          <div class="studio-title">
            <span class="sound-icon" aria-hidden="true">
              <svg viewBox="0 0 24 24">
                <path d="M4 10v4m4-8v12m4-15v18m4-13v8m4-11v14" />
              </svg>
            </span>
            <div>
              <h2>Your sound mix</h2>
              <p v-if="selection.length">{{ syllableCount }} {{ syllableCount === 1 ? 'syllable' : 'syllables' }} in your sequence</p>
              <p v-else>Choose syllables below to get started</p>
            </div>
          </div>
          <div class="studio-actions">
            <button
              v-if="selection.length && !sharedRevealPending"
              class="share-button"
              type="button"
              @click="copyShareLink"
            >
              <svg viewBox="0 0 24 24" aria-hidden="true">
                <path d="M10 13a5 5 0 0 0 7.07 0l3-3A5 5 0 0 0 13 2.93l-1.72 1.71m2.72 6.36a5 5 0 0 0-7.07 0l-3 3A5 5 0 0 0 11 21.07l1.71-1.71" />
              </svg>
              Share mix
            </button>
            <span v-else-if="sharedRevealPending" class="shared-badge">
              <span class="availability-dot"></span> SHARED MIX
            </span>
          </div>
        </div>

        <div class="player-frame" :class="{ 'is-playing': isPlaying }">
          <video
            v-if="selection.length && activeSyllable"
            ref="videoElement"
            :key="`${activeSelection?.word}-${activeSelection?.syllableIndex}`"
            class="video-player"
            :src="activeSyllable.link"
            controls
            autoplay
            playsinline
            preload="auto"
            :aria-label="
              sharedRevealPending
                ? 'Shared pronunciation video'
                : `Pronunciation video for ${activeSyllable.label}`
            "
            @ended="handleClipEnded"
            @error="handleClipError"
          />
          <div v-else class="player-empty">
            <div class="waveform" aria-hidden="true">
              <span v-for="bar in 21" :key="bar" :style="{ '--bar': bar }"></span>
            </div>
            <p>Your next favorite sound starts here.</p>
            <span>SELECT A SYLLABLE TO PREVIEW IT</span>
          </div>
          <div v-if="sharedRevealPending && !isPlaying" class="shared-overlay">
            <button
              class="shared-overlay-icon"
              type="button"
              aria-label="Play shared mix"
              @click="playMix"
            >
              <svg viewBox="0 0 24 24"><path d="m9 6 10 6-10 6z" /></svg>
            </button>
            <p>Your shared mix is ready</p>
            <span>Play the sequence to reveal its syllables</span>
          </div>
        </div>

        <div class="player-controls">
          <div class="now-playing">
            <span class="now-playing-label">
              {{ sharedRevealPending ? 'SHARED MIX' : isPlaying ? 'NOW PLAYING' : 'UP NEXT' }}
            </span>
            <template v-if="activeSyllable && !sharedRevealPending">
              <span class="now-playing-word">{{ wordName(activeSelection?.word ?? '') }}</span>
              <span class="now-playing-separator">·</span>
              <span class="now-playing-label-text">{{ activeSyllable.label }}</span>
              <span v-if="selection.length > 1" class="clip-count">{{ playbackIndex + 1 }} / {{ selection.length }}</span>
            </template>
            <span v-else-if="sharedRevealPending" class="now-playing-muted">
              {{ isPlaying ? 'Playing your sequence' : 'Play to reveal the syllables' }}
            </span>
            <span v-else class="now-playing-muted">Waiting for your first syllable</span>
          </div>
          <button
            v-if="selection.length"
            class="play-button"
            type="button"
            @click="playMix"
          >
            <svg viewBox="0 0 24 24" aria-hidden="true"><path d="m8 5 12 7-12 7z" /></svg>
            {{ isPlaying ? 'Restart mix' : sharedRevealPending ? 'Play shared mix' : 'Play the mix' }}
          </button>
        </div>

        <div v-if="!sharedRevealPending && selectedSyllables.length" class="mix-tray" aria-label="Your selected syllables">
          <div class="mix-tray-heading">
            <span>YOUR MIX</span>
            <span class="mix-tray-hint">Tap <b>×</b> to remove</span>
          </div>
          <div class="mix-sequence">
            <template v-for="(item, index) in selectedSyllables" :key="`${item.word}-${item.syllableIndex}-${index}`">
              <span
                class="mix-chip"
                :class="{ 'is-dragging': draggedSelectionIndex === index }"
                draggable="true"
                @dragstart="startDragging($event, index)"
                @dragover.prevent
                @drop="dropSyllable($event, index)"
                @dragend="draggedSelectionIndex = null"
              >
                <span class="mix-chip-position">{{ String(index + 1).padStart(2, '0') }}</span>
                <span class="mix-chip-word">{{ item.word }}</span>
                <b>{{ item.syllable.label }}</b>
                <button
                  type="button"
                  class="mix-chip-move"
                  :disabled="index === 0"
                  :aria-label="`Move ${item.syllable.label} earlier in your mix`"
                  @click="moveSyllable(index, index - 1)"
                >←</button>
                <button
                  type="button"
                  class="mix-chip-move"
                  :disabled="index === selectedSyllables.length - 1"
                  :aria-label="`Move ${item.syllable.label} later in your mix`"
                  @click="moveSyllable(index, index + 1)"
                >→</button>
                <button
                  type="button"
                  :aria-label="`Remove ${item.syllable.label} from your mix`"
                  @click="removeSyllable(index)"
                >×</button>
              </span>
              <span v-if="index < selectedSyllables.length - 1" class="sequence-arrow" aria-hidden="true">→</span>
            </template>
          </div>
        </div>
      </section>

      <section class="library" aria-labelledby="library-title">
        <div class="library-heading">
          <div>
            <div class="eyebrow"><span class="eyebrow-line"></span> THE WORD LIBRARY</div>
            <h2 id="library-title">Pick your pieces<span class="brand-period">.</span></h2>
          </div>
          <span class="word-count">{{ wordEntries.length }} {{ wordEntries.length === 1 ? 'WORD' : 'WORDS' }}</span>
        </div>

        <div class="word-grid">
          <article v-for="[word, entry] in wordEntries" :key="word" class="word-card">
            <div class="word-card-heading">
              <div>
                <h3>{{ word }}</h3>
                <p>{{ entry.syllable.length }} syllables <span>·</span> Spanish</p>
              </div>
              <span class="word-card-spark" aria-hidden="true">✳</span>
            </div>
            <div class="syllable-list">
              <button
                v-for="(syllable, index) in entry.syllable"
                :key="`${word}-${index}`"
                type="button"
                class="syllable-button"
                :aria-label="`Add ${syllable.label}, syllable ${index + 1} of ${word}, to your mix`"
                @click="addSyllable(word, index)"
              >
                <span class="syllable-number">{{ String(index + 1).padStart(2, '0') }}</span>
                <span class="syllable-label">{{ syllable.label }}</span>
                <span class="syllable-add" aria-hidden="true">+</span>
              </button>
            </div>
          </article>
        </div>
      </section>
    </main>

    <footer class="footer">
      <span>MADE FOR THE LOVE OF WORDS</span>
      <span class="footer-note">Listen close. Speak boldly.</span>
    </footer>
    <div class="sr-only" role="status" aria-live="polite">{{ playbackMessage }}</div>
  </div>
</template>
