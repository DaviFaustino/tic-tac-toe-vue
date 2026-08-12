<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'

const players = ['X', 'O']
const wallpaperEndpoint = import.meta.env.VITE_WALLPAPER_ENDPOINT || 'https://picsum.photos/1920/1080'
const wallpaperCacheName = 'tic-tac-toe-wallpaper-v1'
const wallpaperCacheUrl = `${window.location.origin}/cached-wallpaper`
const wallpaperLocalStorageKey = 'tic-tac-toe-wallpaper'
const winningLines = [
  [0, 1, 2],
  [3, 4, 5],
  [6, 7, 8],
  [0, 3, 6],
  [1, 4, 7],
  [2, 5, 8],
  [0, 4, 8],
  [2, 4, 6],
]

const board = ref(Array(9).fill(''))
const currentPlayer = ref(players[0])
const focusedControl = ref(0)
const squareButtons = ref([])
const resetButton = ref(null)
const wallpaperButton = ref(null)
const wallpaperUrl = ref('')
const wallpaperStatus = ref('')
const isWallpaperLoading = ref(false)
let wallpaperObjectUrl = ''

const remoteKeyNames = {
  ArrowLeft: 'ArrowLeft',
  Left: 'ArrowLeft',
  ArrowUp: 'ArrowUp',
  Up: 'ArrowUp',
  ArrowRight: 'ArrowRight',
  Right: 'ArrowRight',
  ArrowDown: 'ArrowDown',
  Down: 'ArrowDown',
  Enter: 'Enter',
  OK: 'Enter',
  Accept: 'Enter',
}

const remoteKeyCodes = {
  13: 'Enter',
  37: 'ArrowLeft',
  38: 'ArrowUp',
  39: 'ArrowRight',
  40: 'ArrowDown',
}

const winningLine = computed(() =>
  winningLines.find(([a, b, c]) => {
    const mark = board.value[a]
    return mark && mark === board.value[b] && mark === board.value[c]
  }),
)

const winner = computed(() => (winningLine.value ? board.value[winningLine.value[0]] : ''))
const isDraw = computed(() => !winner.value && board.value.every(Boolean))
const gameOver = computed(() => Boolean(winner.value) || isDraw.value)

const statusMessage = computed(() => {
  if (winner.value) return `${winner.value} wins!`
  if (isDraw.value) return "It's a draw."
  return `${currentPlayer.value}'s turn`
})

const gameClasses = computed(() => ({
  'has-wallpaper': Boolean(wallpaperUrl.value),
}))

const wallpaperStyle = computed(() =>
  wallpaperUrl.value ? { '--wallpaper-image': `url("${wallpaperUrl.value}")` } : {},
)

function playSquare(index) {
  if (board.value[index] || gameOver.value) return

  board.value[index] = currentPlayer.value
  currentPlayer.value = currentPlayer.value === players[0] ? players[1] : players[0]
}

function handleSquareClick(index) {
  focusedControl.value = index
  playSquare(index)
}

function resetGame() {
  board.value = Array(9).fill('')
  currentPlayer.value = players[0]
  focusSquare(0)
}

function setSquareRef(element, index) {
  if (element) squareButtons.value[index] = element
}

function focusCurrentControl() {
  nextTick(() => {
    if (focusedControl.value === 'reset') {
      resetButton.value?.focus()
      return
    }

    if (focusedControl.value === 'wallpaper') {
      wallpaperButton.value?.focus()
      return
    }

    squareButtons.value[focusedControl.value]?.focus()
  })
}

function focusSquare(index) {
  focusedControl.value = index
  focusCurrentControl()
}

function focusResetButton() {
  focusedControl.value = 'reset'
  focusCurrentControl()
}

function focusWallpaperButton() {
  focusedControl.value = 'wallpaper'
  focusCurrentControl()
}

function moveSquareFocus(key) {
  const row = Math.floor(focusedControl.value / 3)
  const column = focusedControl.value % 3

  if (key === 'ArrowLeft') focusSquare(row * 3 + Math.max(column - 1, 0))
  if (key === 'ArrowRight') focusSquare(row * 3 + Math.min(column + 1, 2))
  if (key === 'ArrowUp') focusSquare(Math.max(row - 1, 0) * 3 + column)
  if (key === 'ArrowDown') {
    if (row === 2) focusResetButton()
    else focusSquare((row + 1) * 3 + column)
  }
}

function moveResetFocus(key) {
  if (key === 'ArrowUp') focusSquare(7)
  if (key === 'ArrowDown') focusWallpaperButton()
}

function moveWallpaperFocus(key) {
  if (key === 'ArrowUp') focusResetButton()
}

function normalizeRemoteKey(event) {
  return remoteKeyNames[event.key] || remoteKeyCodes[event.keyCode] || remoteKeyCodes[event.which] || ''
}

function handleRemoteKey(event) {
  const key = normalizeRemoteKey(event)
  if (!key) return

  event.preventDefault()

  if (key === 'Enter') {
    if (focusedControl.value === 'reset') resetGame()
    else if (focusedControl.value === 'wallpaper') fetchAndCacheWallpaper()
    else playSquare(focusedControl.value)

    focusCurrentControl()
    return
  }

  if (focusedControl.value === 'reset') moveResetFocus(key)
  else if (focusedControl.value === 'wallpaper') moveWallpaperFocus(key)
  else moveSquareFocus(key)
}

function supportsCacheStorage() {
  return 'caches' in window
}

function setWallpaperFromBlob(blob) {
  if (wallpaperObjectUrl) URL.revokeObjectURL(wallpaperObjectUrl)

  wallpaperObjectUrl = URL.createObjectURL(blob)
  wallpaperUrl.value = wallpaperObjectUrl
}

async function saveWallpaperResponse(response) {
  if (supportsCacheStorage()) {
    try {
      const cache = await caches.open(wallpaperCacheName)
      await cache.put(wallpaperCacheUrl, response.clone())
      return
    } catch {
      // Fall through to localStorage for engines that expose caches but reject writes.
    }
  }

  localStorage.setItem(wallpaperLocalStorageKey, await blobToDataUrl(await response.blob()))
}

async function loadCachedWallpaper() {
  if (supportsCacheStorage()) {
    try {
      const cache = await caches.open(wallpaperCacheName)
      const response = await cache.match(wallpaperCacheUrl)

      if (response) {
        setWallpaperFromBlob(await response.blob())
        wallpaperStatus.value = 'Using cached wallpaper.'
        return
      }
    } catch {
      // Fall back to localStorage below.
    }
  }

  const dataUrl = localStorage.getItem(wallpaperLocalStorageKey)
  if (!dataUrl) return

  wallpaperUrl.value = dataUrl
  wallpaperStatus.value = 'Using saved wallpaper.'
}

function blobToDataUrl(blob) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = () => resolve(reader.result)
    reader.onerror = () => reject(new Error('Could not read wallpaper image.'))
    reader.readAsDataURL(blob)
  })
}

async function fetchAndCacheWallpaper() {
  if (isWallpaperLoading.value) return

  isWallpaperLoading.value = true
  wallpaperStatus.value = 'Fetching wallpaper...'

  try {
    const response = await fetch(wallpaperEndpoint, { cache: 'no-store' })

    if (!response.ok) {
      throw new Error(`Wallpaper request failed with ${response.status}.`)
    }

    const contentType = response.headers.get('content-type') || ''
    if (!contentType.startsWith('image/')) {
      throw new Error('The wallpaper endpoint did not return an image.')
    }

    await saveWallpaperResponse(response.clone())
    setWallpaperFromBlob(await response.blob())
    wallpaperStatus.value = 'Wallpaper saved for reuse.'
  } catch (error) {
    wallpaperStatus.value = error.message || 'Could not save wallpaper.'
  } finally {
    isWallpaperLoading.value = false
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleRemoteKey, true)
  loadCachedWallpaper()
  focusCurrentControl()
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleRemoteKey, true)
  if (wallpaperObjectUrl) URL.revokeObjectURL(wallpaperObjectUrl)
})
</script>

<template>
  <main class="game" :class="gameClasses" :style="wallpaperStyle">
    <section class="panel" aria-labelledby="game-title">
      <p class="eyebrow">Vue Tic Tac Toe</p>
      <h1 id="game-title">Tic Tac Toe</h1>

      <div class="status" role="status" aria-live="polite">
        {{ statusMessage }}
      </div>

      <div class="board" aria-label="Tic tac toe board">
        <button
          v-for="(_, index) in board"
          :key="index"
          :ref="(element) => setSquareRef(element, index)"
          class="square"
          :class="{
            filled: board[index],
            selected: focusedControl === index,
            winner: winningLine?.includes(index),
          }"
          type="button"
          :aria-label="`Square ${index + 1}${board[index] ? `, ${board[index]}` : ''}`"
          :aria-disabled="Boolean(board[index]) || gameOver"
          :tabindex="focusedControl === index ? 0 : -1"
          @click="handleSquareClick(index)"
          @focus="focusedControl = index"
        >
          {{ board[index] }}
        </button>
      </div>

      <button
        ref="resetButton"
        class="reset"
        type="button"
        :tabindex="focusedControl === 'reset' ? 0 : -1"
        @click="resetGame"
        @focus="focusedControl = 'reset'"
      >
        Reset game
      </button>

      <button
        ref="wallpaperButton"
        class="wallpaper"
        type="button"
        :disabled="isWallpaperLoading"
        :tabindex="focusedControl === 'wallpaper' ? 0 : -1"
        @click="fetchAndCacheWallpaper"
        @focus="focusedControl = 'wallpaper'"
      >
        {{ isWallpaperLoading ? 'Saving wallpaper...' : 'Fetch wallpaper' }}
      </button>

      <p v-if="wallpaperStatus" class="wallpaper-status" role="status" aria-live="polite">
        {{ wallpaperStatus }}
      </p>
    </section>
  </main>
</template>

<style scoped>
:global(*) {
  box-sizing: border-box;
}

:global(body) {
  margin: 0;
  min-width: 320px;
  min-height: 100vh;
  color: #18202f;
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

:global(button) {
  font: inherit;
}

.game {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  padding: 2rem;
  background:
    radial-gradient(circle at top left, rgba(255, 196, 87, 0.2), transparent 32rem),
    linear-gradient(135deg, #f7fbff 0%, #eef4f7 48%, #f8f1e9 100%);
  background-position: center;
  background-size: cover;
  transition: background-image 240ms ease;
}

.game.has-wallpaper {
  background-image:
    linear-gradient(rgba(247, 251, 255, 0.66), rgba(248, 241, 233, 0.78)),
    var(--wallpaper-image);
}

.panel {
  width: 30rem;
  max-width: 100%;
  text-align: center;
}

.eyebrow {
  margin: 0 0 0.4rem;
  color: #4f6775;
  font-size: 0.8rem;
  font-weight: 800;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

h1 {
  margin: 0;
  color: #16202d;
  font-size: 4rem;
  font-size: clamp(2.5rem, 9vw, 4.75rem);
  line-height: 0.95;
}

.status {
  min-height: 3.1rem;
  margin: 1.4rem 0 1.2rem;
  padding: 0.8rem 1rem;
  border: 1px solid rgba(24, 32, 47, 0.12);
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.72);
  box-shadow: 0 16px 40px rgba(34, 46, 60, 0.08);
  color: #253447;
  font-size: 1.2rem;
  font-weight: 800;
}

.board {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  width: 24rem;
  max-width: 100%;
  margin: 0 auto;
}

.square {
  display: block;
  width: 7.2rem;
  height: 7.2rem;
  margin: 0.4rem;
  padding: 0;
  border: 2px solid #17202d;
  border-radius: 8px;
  background: #ffffff;
  color: #17202d;
  box-shadow: 0 10px 0 #17202d;
  cursor: pointer;
  font-size: 4.8rem;
  font-size: clamp(2.6rem, 16vw, 5.4rem);
  font-weight: 900;
  line-height: 7.2rem;
  text-align: center;
  transition:
    transform 160ms ease,
    box-shadow 160ms ease,
    background-color 160ms ease,
    color 160ms ease;
}

.square:not([aria-disabled='true']):hover {
  transform: translateY(3px);
  background: #fff8df;
  box-shadow: 0 7px 0 #17202d;
  outline: none;
}

.square:focus,
.square.selected {
  transform: translateY(3px);
  background: #fff8df;
  box-shadow: 0 7px 0 #17202d;
  outline: none;
}

.square[aria-disabled='true'] {
  cursor: default;
}

.square.filled {
  color: #0f6d73;
}

.square.winner {
  background: #ffe08a;
  color: #17202d;
}

.reset {
  margin-top: 1.6rem;
  border: 0;
  border-radius: 8px;
  background: #17202d;
  color: #ffffff;
  cursor: pointer;
  font-weight: 800;
  padding: 0.85rem 1.15rem;
  transition:
    transform 160ms ease,
    background-color 160ms ease;
}

.wallpaper {
  display: block;
  margin: 0.8rem auto 0;
  border: 2px solid #17202d;
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.82);
  color: #17202d;
  cursor: pointer;
  font-weight: 800;
  padding: 0.75rem 1.05rem;
  transition:
    transform 160ms ease,
    background-color 160ms ease,
    color 160ms ease;
}

.reset:hover,
.reset:focus,
.wallpaper:hover,
.wallpaper:focus {
  background: #0f6d73;
  color: #ffffff;
  outline: none;
  transform: translateY(-2px);
}

.wallpaper:disabled {
  cursor: wait;
  opacity: 0.72;
}

.wallpaper-status {
  min-height: 1.4rem;
  margin: 0.7rem 0 0;
  color: #253447;
  font-size: 0.9rem;
  font-weight: 700;
}

@media (max-width: 420px) {
  .game {
    padding: 1.2rem;
  }

  .board {
    width: 18rem;
  }

  .square {
    width: 5.4rem;
    height: 5.4rem;
    margin: 0.3rem;
    box-shadow: 0 7px 0 #17202d;
    font-size: 3.7rem;
    line-height: 5.4rem;
  }
}
</style>
