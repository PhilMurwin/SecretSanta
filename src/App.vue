<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import QRCode from 'qrcode'

type Assignment = { giver: string; recipient: string }
type SavedDrawing = { participants: string[]; assignments: Assignment[]; createdAt: string }

const STORAGE_KEY = 'secret-santa-drawing-v1'
const defaultNames = ['Steve', 'Patricia', 'Lisa', 'Shirley']
const participants = ref<string[]>([...defaultNames])
const drawing = ref<SavedDrawing | null>(null)
const error = ref('')
const copiedName = ref('')
const giftRecipient = ref<string | null>(null)
const invalidGift = ref(false)
const isDarkMode = ref(false)

const qrCodes = ref<Record<string, string>>({})
const qrError = ref('')
const isRevealPage = computed(() => new URLSearchParams(window.location.search).has('gift'))
const participantCount = computed(() => participants.value.filter(name => name.trim()).length)
const distributionRows = computed(() => {
  if (!drawing.value) return []
  return drawing.value.assignments.map(assignment => ({
    name: assignment.giver,
    code: encodeName(assignment.recipient),
  }))
})

function toggleTheme() {
  isDarkMode.value = !isDarkMode.value
}

function encodeName(value: string): string {
  const bytes = new TextEncoder().encode(value)
  let binary = ''
  bytes.forEach(byte => { binary += String.fromCharCode(byte) })
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/g, '')
}

function decodeName(value: string): string {
  const normalized = value.replace(/-/g, '+').replace(/_/g, '/')
  const padded = normalized + '='.repeat((4 - normalized.length % 4) % 4)
  const binary = atob(padded)
  const bytes = Uint8Array.from(binary, char => char.charCodeAt(0))
  return new TextDecoder().decode(bytes)
}

function shuffle<T>(items: T[]): T[] {
  const result = [...items]
  for (let i = result.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[result[i], result[j]] = [result[j], result[i]]
  }
  return result
}

function createAssignments(names: string[]): Assignment[] {
  // Rejection sampling gives a randomized permutation with no self-assignments.
  for (let attempt = 0; attempt < 1000; attempt++) {
    const recipients = shuffle(names)
    if (recipients.every((recipient, index) => recipient !== names[index])) {
      return names.map((giver, index) => ({ giver, recipient: recipients[index] }))
    }
  }

  // Guaranteed fallback: a randomly rotated shuffled list cannot assign anyone to themselves.
  const shuffled = shuffle(names)
  const offset = 1 + Math.floor(Math.random() * (shuffled.length - 1))
  return shuffled.map((giver, index) => ({
    giver,
    recipient: shuffled[(index + offset) % shuffled.length],
  }))
}

async function generateDrawing() {
  error.value = ''
  const cleaned = participants.value.map(name => name.trim())
  if (cleaned.length < 2 || cleaned.some(name => !name)) {
    error.value = 'Enter at least two participant names.'
    return
  }
  const normalized = cleaned.map(name => name.toLocaleLowerCase())
  if (new Set(normalized).size !== cleaned.length) {
    error.value = 'Participant names must be unique.'
    return
  }

  drawing.value = {
    participants: cleaned,
    assignments: createAssignments(cleaned),
    createdAt: new Date().toISOString(),
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(drawing.value))

  qrError.value = ''
  try {
    await createQrCodes()
  } catch {
    qrError.value = 'The drawing was saved, but QR codes could not be generated.'
  }
}

function clearDrawing() {
  if (!window.confirm('Discard this drawing and start over?')) return
  drawing.value = null
  qrCodes.value = {}
  qrError.value = ''
  localStorage.removeItem(STORAGE_KEY)
  error.value = ''
}

function participantUrl(code: string): string {
  return `${window.location.origin}${import.meta.env.BASE_URL}?gift=${encodeURIComponent(code)}`
}

async function generateQrCode(url: string): Promise<string> {
  return QRCode.toDataURL(url, {
    width: 180,
    margin: 1,
    errorCorrectionLevel: 'M',
  })
}

async function createQrCodes() {
  const entries = await Promise.all(
    distributionRows.value.map(async row => [
      row.name,
      await generateQrCode(participantUrl(row.code)),
    ] as const)
  )

  qrCodes.value = Object.fromEntries(entries)
}

async function copyLink(name: string, code: string) {
  await navigator.clipboard.writeText(participantUrl(code))
  copiedName.value = name
  window.setTimeout(() => { if (copiedName.value === name) copiedName.value = '' }, 1800)
}

function printSheet() {
  window.print()
}


watch(isDarkMode, (dark) => {
  document.documentElement.classList.toggle('dark', dark)
  localStorage.setItem('secret-santa-theme', dark ? 'dark' : 'light')
})

onMounted(async () => {
  const savedTheme = localStorage.getItem('secret-santa-theme')
  isDarkMode.value = savedTheme === 'dark'
  document.documentElement.classList.toggle('dark', isDarkMode.value)

  if (isRevealPage.value) {
    try {
      const code = new URLSearchParams(window.location.search).get('gift')
      if (!code) throw new Error('Missing code')
      giftRecipient.value = decodeName(code)
      if (!giftRecipient.value.trim()) throw new Error('Empty name')
    } catch {
      invalidGift.value = true
    }
    return
  }

  try {
    const saved = localStorage.getItem(STORAGE_KEY)
    if (saved) drawing.value = JSON.parse(saved) as SavedDrawing
  } catch {
    localStorage.removeItem(STORAGE_KEY)
  }

  if (drawing.value) {
    try {
      await createQrCodes()
    } catch {
      qrError.value = 'QR codes could not be generated.'
    }
  }
})
</script>

<template>
  <main v-if="isRevealPage" class="reveal-page">
    <section v-if="!invalidGift" class="reveal-card">
      <div class="ornament">🎁</div>
      <p class="eyebrow">YOUR SECRET SANTA ASSIGNMENT</p>
      <h1>You're gifting to</h1>
      <p class="recipient">{{ giftRecipient }}</p>
      <p class="keep-secret">Keep it a secret. Happy gifting!</p>
    </section>
    <section v-else class="reveal-card">
      <div class="ornament">🎄</div>
      <h1>We couldn't read that link.</h1>
      <p class="keep-secret">Ask the organizer for your Secret Santa link again.</p>
    </section>
  </main>

  <main v-else class="app-shell">
    <header class="site-header">
      <div class="brand-mark">🎄</div>
      <div class="brand-title">
        <p class="eyebrow">A LITTLE HOLIDAY MYSTERY</p>
        <h1>Secret Santa</h1>
      </div>
      <button
        class="theme-toggle"
        type="button"
        @click="toggleTheme"
        :aria-label="isDarkMode ? 'Switch to light mode' : 'Switch to dark mode'"
      >
        {{ isDarkMode ? '☀️' : '🌙' }}
      </button>
    </header>

    <section v-if="!drawing" class="panel">
      <div class="section-heading">
        <div>
          <p class="eyebrow">THE ORGANIZER</p>
          <h2>Who's participating?</h2>
        </div>
        <span class="count">{{ participantCount }} people</span>
      </div>

      <p class="intro">Add everyone who's joining. We'll make sure nobody draws their own name.</p>

      <div class="participant-list">
        <div v-for="(name, index) in participants" :key="index" class="participant-row">
          <span class="person-icon">👤</span>
          <input v-model="participants[index]" :aria-label="`Participant ${index + 1}`" placeholder="Participant name" maxlength="80" />
          <button class="icon-button" type="button" aria-label="Remove participant" :disabled="participants.length <= 2" @click="participants.splice(index, 1)">×</button>
        </div>
      </div>

      <button class="text-button" type="button" @click="participants.push('')">＋ Add another person</button>
      <p v-if="error" class="error-message">{{ error }}</p>
      <button class="primary-button" type="button" @click="generateDrawing">Generate Secret Santa drawing <span>→</span></button>
      <p class="fine-print">The drawing is saved in this browser so you can return to it later.</p>
    </section>

    <section v-else class="panel">
      <div class="success-heading">
        <div class="success-icon">✓</div>
        <div>
          <p class="eyebrow">DRAWING COMPLETE</p>
          <h2>Your Secret Santas are ready.</h2>
        </div>
      </div>
      <p class="intro">Nobody was assigned themselves. Each person has one unique link. Print this sheet and give each person only their own link.</p>

      <div class="link-list">
        <div v-for="row in distributionRows" :key="row.name" class="link-row">
          <div class="link-person">
            <strong>{{ row.name }}</strong>
            <span>{{ participantUrl(row.code) }}</span>
          </div>

          <button class="copy-button" type="button" @click="copyLink(row.name, row.code)">
            {{ copiedName === row.name ? 'Copied!' : 'Copy link' }}
          </button>
        </div>
      </div>

      <div class="action-row">
        <button class="primary-button" type="button" @click="printSheet">Print distribution sheet <span>↗</span></button>
        <button class="secondary-button" type="button" @click="clearDrawing">Start over</button>
      </div>
      <p class="fine-print">Keep the printed sheet private. Anyone with a person's link can see that person's recipient.</p>
    </section>

    <footer>Made for keeping the holiday surprises secret.</footer>

    <section v-if="drawing" class="print-sheet" aria-hidden="true">
      <h1>Secret Santa 2026</h1>
      <p class="print-subtitle">Give each person only their own link. Keep this sheet private.</p>
      <div v-for="row in distributionRows" :key="row.name" class="print-entry">
        <h2>{{ row.name }}</h2>
        <p>{{ participantUrl(row.code) }}</p>
        <img
          v-if="qrCodes[row.name]"
          :src="qrCodes[row.name]"
          :alt="`QR code for ${row.name}`"
          class="print-qr"
        />
      </div>
    </section>
  </main>
</template>
