<script setup>
import { ref, nextTick, onBeforeUnmount, onMounted } from 'vue'
import { readAccounts, saveAccounts, readSession, saveSession, clearSession } from './storage.js'

const mode = ref('signin')
const name = ref('')
const email = ref('')
const password = ref('')
const confirmPassword = ref('')
const accounts = ref(readAccounts())
const savedEmail = readSession()
const profile = ref(accounts.value.find(account => account.email === savedEmail) ?? null)
const busy = ref(false)
const error = ref('')
const notice = ref('')
const shake = ref(false)
const card = ref(null)
const mouseX = ref(0)
const mouseY = ref(0)
const currentTime = ref('')
let timer
let clockTimer

function updateClock() {
  const now = new Date()
  currentTime.value = now.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', second: '2-digit', hour12: false })
}

function handleMouseMove(e) {
  mouseX.value = ((e.clientX / window.innerWidth) - 0.5) * 8
  mouseY.value = ((e.clientY / window.innerHeight) - 0.5) * 8
}

onMounted(() => {
  updateClock()
  clockTimer = setInterval(updateClock, 1000)
  window.addEventListener('mousemove', handleMouseMove)
})

async function showError(message) {
  error.value = message
  shake.value = false
  await nextTick()
  void card.value?.offsetWidth
  shake.value = true
}
function switchMode(nextMode) {
  if (busy.value) return
  mode.value = nextMode
  password.value = ''
  confirmPassword.value = ''
  error.value = ''
  notice.value = ''
  shake.value = false
}
async function submitForm() {
  if (busy.value || profile.value) return
  error.value = ''
  notice.value = ''
  const value = email.value.trim().toLowerCase()
  if (mode.value === 'signup' && name.value.trim().length < 2) {
    return showError('NAME MUST BE ≥ 2 CHARACTERS.')
  }
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
    return showError('INVALID EMAIL FORMAT.')
  }
  if (password.value.length < 4) {
    return showError('PASSWORD MUST BE ≥ 4 CHARACTERS.')
  }
  const existing = accounts.value.find(account => account.email === value)
  if (mode.value === 'signup') {
    if (password.value !== confirmPassword.value) {
      return showError('PASSWORDS DO NOT MATCH.')
    }
    if (existing) return showError('EMAIL EXISTS. SIGN IN INSTEAD.')
  } else if (!existing) {
    return showError('NO ACCOUNT FOUND. SIGN UP FIRST.')
  }
  busy.value = true
  await new Promise(resolve => { timer = setTimeout(resolve, 1200) })
  if (mode.value === 'signup') {
    const account = { name: name.value.trim(), email: value }
    accounts.value = [...accounts.value, account]
    const saved = saveAccounts(accounts.value)
    email.value = value
    mode.value = 'signin'
    notice.value = saved ? 'ACCOUNT CREATED. SIGN IN NOW.'
      : 'ACCOUNT CREATED FOR THIS TAB ONLY.'
  } else {
    const saved = saveSession(existing.email)
    profile.value = existing
    notice.value = saved ? '' : 'SESSION: THIS TAB ONLY.'
  }
  password.value = ''
  confirmPassword.value = ''
  shake.value = false
  busy.value = false
}
function logout() {
  const removed = clearSession()
  profile.value = null
  mode.value = 'signin'
  password.value = ''
  confirmPassword.value = ''
  error.value = ''
  shake.value = false
  notice.value = removed ? '' : 'STORAGE CLEAR FAILED.'
}
function focusPanel(element) { element.querySelector('[data-heading]')?.focus() }
onBeforeUnmount(() => {
  clearTimeout(timer)
  clearInterval(clockTimer)
  window.removeEventListener('mousemove', handleMouseMove)
})
</script>

<template>
  <main class="shell">
    <!-- DECORATIVE GRID OVERLAY -->
    <div class="grid-overlay" aria-hidden="true">
      <div class="grid-line" v-for="n in 12" :key="n"></div>
    </div>

    <!-- BRUTALIST HEADER BAR -->
    <header class="top-bar">
      <a class="brand" href="#" @click.prevent>FORM<span class="brand-slash">/</span>LAB</a>
      <div class="top-bar-right">
        <span class="clock">{{ currentTime }}</span>
        <span class="top-label">EDITORIAL BRUTALISM</span>
      </div>
    </header>

    <!-- LEFT PANEL — EDITORIAL STATEMENT -->
    <aside class="intro">
      <div class="intro-inner">
        <div class="issue-marker" aria-hidden="true">
          <span class="issue-no">Nº 01</span>
          <span class="issue-line"></span>
          <span class="issue-year">2026</span>
        </div>
        <div class="intro-copy">
          <p class="section-tag">AUTHENTICATION INTERFACE</p>
          <h1>
            <span class="h1-line h1-line-1">Raw</span>
            <span class="h1-line h1-line-2">Form.</span>
            <span class="h1-line h1-line-3" :style="{ transform: `translate(${mouseX}px, ${mouseY}px)` }">Pure</span>
            <span class="h1-line h1-line-4">Function.</span>
          </h1>
          <p class="manifesto">No decoration. No pretense.<br>Only the interface, stripped<br>to its essential structure.</p>
        </div>
        <div class="intro-bottom">
          <div class="scroll-indicator" aria-hidden="true">
            <span class="scroll-line"></span>
            <span class="scroll-text">SCROLL</span>
          </div>
          <p class="intro-footer">WEB SERVICE DESIGN<br><span class="footer-sub">ENGLISH SECTION — FALL 2026</span></p>
        </div>
      </div>
    </aside>

    <!-- RIGHT PANEL — FORM WORKSPACE -->
    <section class="workspace" aria-label="Sign in and sign up practice">
      <div class="workspace-header">
        <p class="demo-label"><span class="demo-dot"></span>LOCAL DEMO</p>
        <p class="workspace-coords" aria-hidden="true">47.3769° N, 8.5417° E</p>
      </div>

      <Transition name="panel" mode="out-in" @after-enter="focusPanel">
        <!-- SIGN IN / SIGN UP FORM -->
        <section v-if="!profile" :key="mode" class="card-shell" :class="{ 'signup-shell': mode === 'signup' }">
          <div ref="card" class="card" :class="{ 'is-error': shake }">
            <div class="card-number" aria-hidden="true">{{ mode === 'signup' ? '02' : '01' }}</div>
            <p class="eyebrow">{{ mode === 'signup' ? '// START HERE' : '// WELCOME BACK' }}</p>
            <h2 data-heading tabindex="-1">{{ mode === 'signup' ? 'Sign up.' : 'Sign in.' }}</h2>
            <p class="description">{{ mode === 'signup' ? 'Create a demo profile in this browser.' : 'Use the email from your demo sign-up.' }}</p>

            <form @submit.prevent="submitForm" novalidate :aria-busy="busy">
              <fieldset :disabled="busy">
                <div v-if="mode === 'signup'" class="field">
                  <label for="name">
                    <span class="label-index">01</span>
                    DEMO NAME
                  </label>
                  <input id="name" v-model="name" type="text" autocomplete="off" placeholder="Alex" maxlength="60" :aria-invalid="!!error" aria-describedby="form-error">
                </div>
                <div class="field">
                  <label for="email">
                    <span class="label-index">{{ mode === 'signup' ? '02' : '01' }}</span>
                    EMAIL
                  </label>
                  <input id="email" v-model="email" type="email" autocomplete="off" placeholder="alex@example.com" maxlength="100" :aria-invalid="!!error" aria-describedby="form-error">
                </div>
                <div class="field">
                  <label for="password">
                    <span class="label-index">{{ mode === 'signup' ? '03' : '02' }}</span>
                    DEMO PASSWORD
                  </label>
                  <input id="password" v-model="password" type="password" autocomplete="off" placeholder="Any 4+ characters" maxlength="100" :aria-invalid="!!error" aria-describedby="password-help form-error">
                  <small id="password-help">{{ mode === 'signup' ? '→ Never saved. Repeat below.' : '→ Any 4+ characters. No check.' }}</small>
                </div>
                <div v-if="mode === 'signup'" class="field">
                  <label for="confirm-password">
                    <span class="label-index">04</span>
                    CONFIRM PASSWORD
                  </label>
                  <input id="confirm-password" v-model="confirmPassword" type="password" autocomplete="off" placeholder="Repeat the value" maxlength="100" :aria-invalid="!!error" aria-describedby="form-error">
                </div>

                <p id="form-error" class="error-message" role="alert">{{ error }}</p>

                <button class="primary" type="submit" :disabled="busy" id="submit-button">
                  <span v-if="busy" class="spinner" aria-hidden="true"></span>
                  <span class="btn-text">{{ busy ? (mode === 'signup' ? 'CREATING...' : 'OPENING...') : (mode === 'signup' ? 'CREATE ACCOUNT' : 'SIGN IN') }}</span>
                  <span v-if="!busy" class="btn-arrow" aria-hidden="true">↗</span>
                </button>
              </fieldset>
            </form>

            <div class="switch-section">
              <div class="switch-line" aria-hidden="true"></div>
              <p class="switch-copy">{{ mode === 'signup' ? 'Already registered?' : 'No account yet?' }}
                <button class="text-button" type="button" :disabled="busy" @click="switchMode(mode === 'signup' ? 'signin' : 'signup')">{{ mode === 'signup' ? 'SIGN IN →' : 'SIGN UP →' }}</button>
              </p>
            </div>
          </div>
        </section>

        <!-- WELCOME / PROFILE VIEW -->
        <section v-else key="welcome" class="card-shell">
          <div class="card welcome">
            <div class="card-number" aria-hidden="true">00</div>
            <p class="eyebrow">// YOUR SPACE</p>
            <h2 data-heading tabindex="-1">Hello,<br>{{ profile.name }}.</h2>
            <p class="description">Profile loaded successfully.</p>
            <dl class="profile-details">
              <dt>REGISTERED EMAIL</dt>
              <dd>{{ profile.email }}</dd>
            </dl>
            <p class="reload-hint">↻ Reload to persist. Sign out to return.</p>
            <button class="primary" type="button" @click="logout" id="signout-button">
              <span class="btn-text">SIGN OUT</span>
              <span class="btn-arrow" aria-hidden="true">↗</span>
            </button>
          </div>
        </section>
      </Transition>

      <div class="workspace-footer">
        <p class="notice" role="status">{{ notice }}</p>
        <p class="demo-note">// UI PRACTICE ONLY. NO REAL AUTH.</p>
      </div>
    </section>

    <!-- BOTTOM STRIP -->
    <footer class="bottom-strip" aria-hidden="true">
      <span>BRUTALIST DESIGN SYSTEM</span>
      <span>VERSION 1.0.0</span>
      <span>NO FRAMEWORKS. NO BORDERS.</span>
      <span>2026</span>
    </footer>
  </main>
</template>
