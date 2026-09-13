<script setup>
import { computed, onMounted, onUnmounted, reactive, ref, watch } from 'vue'

const STORAGE_KEY = 'climatewise-learning-interests'
const USERS_STORAGE_KEY = 'climatewise-users'
const CURRENT_USER_STORAGE_KEY = 'climatewise-current-user'
const RATINGS_STORAGE_KEY = 'climatewise-ratings'
const MIN_PASSWORD_LENGTH = 6
const roles = ['member', 'admin']

const learningResources = [
  {
    id: 1,
    title: 'Climate Basics',
    type: 'Guide',
    audience: 'Students',
    description: 'Plain-language lessons about greenhouse gases, carbon footprints, and climate action.',
  },
  {
    id: 2,
    title: 'Melbourne Local Impacts',
    type: 'Article',
    audience: 'Community',
    description: 'Short local examples about heatwaves, storms, transport, and urban sustainability.',
  },
  {
    id: 3,
    title: 'Classroom Action Plan',
    type: 'Worksheet',
    audience: 'Teachers',
    description: 'A simple activity template for planning school or community climate actions.',
  },
]

const resourceCount = computed(() => learningResources.length)

const audienceCount = computed(() => {
  return new Set(learningResources.map((resource) => resource.audience)).size
})

const topics = [
  'Climate basics',
  'Local impacts',
  'Sustainability choices',
  'Community action',
]

const registeredUsers = ref([])
const currentUser = ref(null)
const isAdmin = computed(() => currentUser.value?.role === 'admin')
const currentPage = ref(window.location.hash)
const ratings = ref([])
const ratingForm = reactive({ score: '', comment: '' })
const ratingMessage = ref('')
const myRating = computed(() => ratings.value.find((rating) => rating.userId === currentUser.value?.id))
const averageRating = computed(() => ratings.value.length
  ? (ratings.value.reduce((sum, rating) => sum + rating.score, 0) / ratings.value.length).toFixed(1)
  : null)
const authMode = ref('login')
const registerAttempted = ref(false)
const loginAttempted = ref(false)
const registerSuccessMessage = ref('')
const loginErrorMessage = ref('')
const authBusy = ref(true)
const authReady = ref(false)

const registerForm = reactive({
  fullName: '',
  emailAddress: '',
  password: '',
  confirmPassword: '',
  role: 'member',
})

const loginForm = reactive({
  emailAddress: '',
  password: '',
  role: 'member',
})

const interestForm = reactive({
  fullName: '',
  emailAddress: '',
  postcode: '',
  topic: topics[0],
  learningGoal: '',
})

const touched = reactive({
  fullName: false,
  emailAddress: false,
  postcode: false,
  topic: false,
  learningGoal: false,
})

const submitAttempted = ref(false)
const savedInterests = ref([])
const emailPattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
const postcodePattern = /^\d{4}$/

const registerErrors = computed(() => {
  const errors = contactErrors(registerForm)
  const emailAddress = normalizeEmail(registerForm.emailAddress)
  const password = registerForm.password
  const confirmPassword = registerForm.confirmPassword

  if (registeredUsers.value.some((user) => user.emailAddress === emailAddress)) {
    errors.emailAddress = 'This email is already registered.'
  }
  if (!roles.includes(registerForm.role)) errors.role = 'Choose Member or Admin.'

  if (!password) {
    errors.password = 'Password is required.'
  } else if (password.length < MIN_PASSWORD_LENGTH) {
    errors.password = `Password must be at least ${MIN_PASSWORD_LENGTH} characters.`
  } else if (password.length > 128) {
    errors.password = 'Use 128 characters or fewer.'
  }

  if (!confirmPassword) {
    errors.confirmPassword = 'Confirm your password.'
  } else if (confirmPassword !== password) {
    errors.confirmPassword = 'Passwords must match.'
  }

  return errors
})

const loginErrors = computed(() => {
  const errors = {}
  const emailAddress = normalizeEmail(loginForm.emailAddress)

  if (!emailAddress) {
    errors.emailAddress = 'Email address is required.'
  } else if (!validEmail(emailAddress)) {
    errors.emailAddress = 'Enter a valid email address (up to 254 characters).'
  }

  if (!loginForm.password) {
    errors.password = 'Password is required.'
  }
  if (!roles.includes(loginForm.role)) errors.role = 'Choose Member or Admin.'

  return errors
})

const registerFormIsValid = computed(() => Object.keys(registerErrors.value).length === 0)

const loginFormIsValid = computed(() => Object.keys(loginErrors.value).length === 0)

const formErrors = computed(() => {
  const errors = contactErrors(interestForm)
  const postcode = interestForm.postcode.trim()
  const learningGoal = interestForm.learningGoal.trim()

  if (!postcode) {
    errors.postcode = 'Postcode is required.'
  } else if (!postcodePattern.test(postcode)) {
    errors.postcode = 'Enter a 4-digit postcode.'
  }

  if (!topics.includes(interestForm.topic)) {
    errors.topic = 'Choose one topic interest.'
  }

  if (!learningGoal) {
    errors.learningGoal = 'Learning goal is required.'
  } else if (learningGoal.length < 10) {
    errors.learningGoal = 'Write at least 10 characters.'
  } else if (learningGoal.length > 500) {
    errors.learningGoal = 'Use 500 characters or fewer.'
  }

  return errors
})

const formIsValid = computed(() => Object.keys(formErrors.value).length === 0)

const visibleInterests = computed(() => {
  if (!currentUser.value) return []
  if (isAdmin.value && currentPage.value === '#admin') return savedInterests.value
  return savedInterests.value.filter((interest) => interest.userId === currentUser.value.id
    || (!interest.userId && normalizeEmail(interest.emailAddress) === currentUser.value.emailAddress))
})
const savedInterestCount = computed(() => visibleInterests.value.length)

onMounted(async () => {
  window.addEventListener('hashchange', updatePage)
  try {
    await loadUsers()
    loadSavedInterests()
    loadRatings()
    loadCurrentUser()
    authReady.value = true
  } catch {
    loginErrorMessage.value = 'Unable to load accounts. Use HTTPS or localhost and allow browser storage, then reload.'
  } finally {
    authBusy.value = false
  }
})

onUnmounted(() => window.removeEventListener('hashchange', updatePage))

function updatePage() {
  currentPage.value = window.location.hash
}

watch(currentUser, () => {
  ratingForm.score = myRating.value?.score ?? ''
  ratingForm.comment = myRating.value?.comment ?? ''
  ratingMessage.value = ''
  resetInterestForm()
})

watch(ratings, (items) => localStorage.setItem(RATINGS_STORAGE_KEY, JSON.stringify(items)))

watch(
  savedInterests,
  (records) => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(records))
  },
  { deep: true },
)

function readList(key) {
  const stored = localStorage.getItem(key)
  try {
    const items = JSON.parse(stored || '[]')
    return Array.isArray(items) ? items : []
  } catch {
    return []
  }
}

async function loadUsers() {
  const users = []
  for (const user of readList(USERS_STORAGE_KEY)) {
    if (!user || !Number.isSafeInteger(user.id) || !validText(user.fullName)
      || !validEmail(user.emailAddress) || !roles.includes(user.role ?? 'member')) continue
    let credentials
    if (validText(user.salt, 32) && /^[a-f0-9]{32}$/.test(user.salt)
      && validText(user.passwordHash, 64) && /^[a-f0-9]{64}$/.test(user.passwordHash)) {
      credentials = { salt: user.salt, passwordHash: user.passwordHash }
    } else if (typeof user.password === 'string' && user.password.length > 0) {
      // Migrate C1 passwords before exposing accounts or writing them back to storage.
      credentials = await hashPassword(user.password)
    } else continue
    users.push({ id: user.id, fullName: user.fullName, emailAddress: normalizeEmail(user.emailAddress),
      role: user.role ?? 'member', createdAt: validText(user.createdAt) ? user.createdAt : '', ...credentials })
  }
  localStorage.setItem(USERS_STORAGE_KEY, JSON.stringify(users))
  registeredUsers.value = users
}

function loadCurrentUser() {
  const storedUser = localStorage.getItem(CURRENT_USER_STORAGE_KEY)

  if (!storedUser) {
    return
  }

  try {
    const parsedUser = JSON.parse(storedUser)
    const matchingUser = registeredUsers.value.find((user) => user.id === parsedUser?.id)

    if (matchingUser) {
      localStorage.setItem(CURRENT_USER_STORAGE_KEY, JSON.stringify({ id: matchingUser.id }))
      currentUser.value = matchingUser
    } else {
      localStorage.removeItem(CURRENT_USER_STORAGE_KEY)
    }
  } catch {
    localStorage.removeItem(CURRENT_USER_STORAGE_KEY)
  }
}

function loadSavedInterests() {
  savedInterests.value = readList(STORAGE_KEY).filter((item) => item && Number.isSafeInteger(item.id)
    && (item.userId === undefined || Number.isSafeInteger(item.userId))
    && validText(item.fullName) && validEmail(item.emailAddress)
    && typeof item.postcode === 'string' && postcodePattern.test(item.postcode)
    && topics.includes(item.topic) && validText(item.learningGoal) && validText(item.savedAt))
}

function validText(value, max = Infinity, min = 1) {
  return typeof value === 'string' && value.trim().length >= min && value.length <= max
}

function validEmail(value) {
  return validText(value, 254) && emailPattern.test(value.trim())
}

function contactErrors(form) {
  const errors = {}
  if (!validText(form.fullName, 80)) errors.fullName = 'Enter a name (1-80 characters).'
  if (!validEmail(form.emailAddress)) errors.emailAddress = 'Enter a valid email address (up to 254 characters).'
  return errors
}

function toHex(bytes) {
  return Array.from(new Uint8Array(bytes), (byte) => byte.toString(16).padStart(2, '0')).join('')
}

async function hashPassword(password, salt = toHex(crypto.getRandomValues(new Uint8Array(16)))) {
  const key = await crypto.subtle.importKey('raw', new TextEncoder().encode(password), 'PBKDF2', false, ['deriveBits'])
  const saltBytes = Uint8Array.from(salt.match(/../g), (byte) => parseInt(byte, 16))
  const bits = await crypto.subtle.deriveBits(
    { name: 'PBKDF2', hash: 'SHA-256', salt: saltBytes, iterations: 600000 }, key, 256,
  )
  return { salt, passwordHash: toHex(bits) }
}

function normalizeEmail(emailAddress) {
  return emailAddress.trim().toLowerCase()
}

function loadRatings() {
  const items = readList(RATINGS_STORAGE_KEY).filter((rating) => rating && Number.isInteger(rating.score)
    && rating.score >= 1 && rating.score <= 5 && validText(rating.comment ?? '', 100, 0)
    && registeredUsers.value.some((user) => user.id === rating.userId))
  ratings.value = [...new Map(items.map((rating) => [rating.userId, rating])).values()]
}

function submitRating() {
  if (!currentUser.value) return
  const score = ratingForm.score
  if (!Number.isInteger(score) || score < 1 || score > 5 || !validText(ratingForm.comment, 100, 0)) {
    ratingMessage.value = 'Choose 1 to 5 stars and keep comments within 100 characters.'
    return
  }
  const rating = { userId: currentUser.value.id, score, comment: ratingForm.comment.trim() }
  ratings.value = [...ratings.value.filter((item) => item.userId !== rating.userId), rating]
  ratingMessage.value = 'Your rating has been saved.'
}

function formattedDate() {
  return new Date().toLocaleDateString('en-AU', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  })
}

function switchAuthMode(mode) {
  if (authBusy.value || !authReady.value) return
  authMode.value = mode
  registerAttempted.value = false
  loginAttempted.value = false
  loginErrorMessage.value = ''

  if (mode === 'register') {
    registerSuccessMessage.value = ''
  }
}

async function registerUser() {
  if (authBusy.value || !authReady.value) return
  registerAttempted.value = true
  registerSuccessMessage.value = ''
  loginErrorMessage.value = ''

  if (!registerFormIsValid.value) {
    return
  }

  const newUser = {
    id: Date.now(),
    fullName: registerForm.fullName.trim(),
    emailAddress: normalizeEmail(registerForm.emailAddress),
    role: registerForm.role,
    createdAt: formattedDate(),
  }

  authBusy.value = true
  try {
    Object.assign(newUser, await hashPassword(registerForm.password))
    const users = [newUser, ...registeredUsers.value]
    localStorage.setItem(USERS_STORAGE_KEY, JSON.stringify(users))
    registeredUsers.value = users
    loginForm.emailAddress = newUser.emailAddress
    loginForm.password = ''
    loginForm.role = newUser.role
    registerSuccessMessage.value = 'Account created. You can now log in.'
    resetRegisterForm()
    authMode.value = 'login'
  } catch {
    loginErrorMessage.value = 'Unable to create your account. Use HTTPS or localhost and allow browser storage.'
  } finally {
    authBusy.value = false
  }
}

async function loginUser() {
  if (authBusy.value || !authReady.value) return
  loginAttempted.value = true
  loginErrorMessage.value = ''

  if (!loginFormIsValid.value) {
    return
  }

  const emailAddress = normalizeEmail(loginForm.emailAddress)
  const matchedUser = registeredUsers.value.find((user) => {
    return user.emailAddress === emailAddress && user.role === loginForm.role
  })

  authBusy.value = true
  try {
    if (!matchedUser || (await hashPassword(loginForm.password, matchedUser.salt)).passwordHash !== matchedUser.passwordHash) {
      loginErrorMessage.value = 'Email, password or role is incorrect.'
      return
    }
    localStorage.setItem(CURRENT_USER_STORAGE_KEY, JSON.stringify({ id: matchedUser.id }))
    currentUser.value = matchedUser
    window.location.hash = 'home'
    updatePage()
    registerSuccessMessage.value = ''
    resetLoginForm()
  } catch {
    loginErrorMessage.value = 'Unable to log in. Use HTTPS or localhost and allow browser storage.'
  } finally {
    authBusy.value = false
  }
}

function logoutUser() {
  localStorage.removeItem(CURRENT_USER_STORAGE_KEY)
  currentUser.value = null
  authMode.value = 'login'
}

function resetRegisterForm() {
  registerForm.fullName = ''
  registerForm.emailAddress = ''
  registerForm.password = ''
  registerForm.confirmPassword = ''
  registerForm.role = 'member'
  registerAttempted.value = false
}

function resetLoginForm() {
  loginForm.emailAddress = ''
  loginForm.password = ''
  loginForm.role = 'member'
  loginAttempted.value = false
  loginErrorMessage.value = ''
}

function markTouched(field) {
  touched[field] = true
}

function shouldShowError(field) {
  return submitAttempted.value || touched[field]
}

function fieldState(field) {
  if (!shouldShowError(field)) {
    return ''
  }

  return formErrors.value[field] ? 'is-invalid' : 'is-valid'
}

function saveInterest() {
  if (!currentUser.value) return
  submitAttempted.value = true
  Object.keys(touched).forEach((field) => {
    touched[field] = true
  })

  if (!formIsValid.value) {
    return
  }

  savedInterests.value.unshift({
    id: Date.now(),
    userId: currentUser.value.id,
    fullName: interestForm.fullName.trim(),
    emailAddress: interestForm.emailAddress.trim(),
    postcode: interestForm.postcode.trim(),
    topic: interestForm.topic,
    learningGoal: interestForm.learningGoal.trim(),
    savedAt: formattedDate(),
  })

  resetInterestForm()
}

function resetInterestForm() {
  interestForm.fullName = ''
  interestForm.emailAddress = ''
  interestForm.postcode = ''
  interestForm.topic = topics[0]
  interestForm.learningGoal = ''
  submitAttempted.value = false
  Object.keys(touched).forEach((field) => {
    touched[field] = false
  })
}

function clearSavedInterests() {
  if (!isAdmin.value) return
  savedInterests.value = []
}
</script>

<template>
  <main v-if="!currentUser" class="auth-shell">
    <section class="container py-4 py-lg-5">
      <div class="auth-layout">
        <fieldset class="auth-card" :disabled="authBusy || !authReady" :aria-busy="authBusy">
          <div class="auth-tabs" role="tablist" aria-label="Authentication options">
            <button
              class="auth-tab"
              :class="{ active: authMode === 'login' }"
              type="button"
              @click="switchAuthMode('login')"
            >
              Login
            </button>
            <button
              class="auth-tab"
              :class="{ active: authMode === 'register' }"
              type="button"
              @click="switchAuthMode('register')"
            >
              Register
            </button>
          </div>

          <div v-if="loginErrorMessage" class="auth-alert error" role="alert">
            {{ loginErrorMessage }}
          </div>

          <form v-if="authMode === 'login'" novalidate @submit.prevent="loginUser">
            <div class="section-heading">
              <p class="section-kicker">Welcome back</p>
              <h2 class="h4 fw-bold mb-0">Login</h2>
            </div>

            <div v-if="registerSuccessMessage" class="auth-alert success">
              {{ registerSuccessMessage }}
            </div>

            <div class="mb-3">
              <label for="loginEmailAddress" class="form-label">Email address</label>
              <input
                id="loginEmailAddress"
                v-model="loginForm.emailAddress"
                maxlength="254"
                class="form-control"
                :class="{ 'is-invalid': loginAttempted && Boolean(loginErrors.emailAddress) }"
                type="email"
                placeholder="name@example.com"
                required
                :aria-invalid="loginAttempted && Boolean(loginErrors.emailAddress)"
                aria-describedby="loginEmailAddressFeedback"
              >
              <div id="loginEmailAddressFeedback" class="invalid-feedback">
                {{ loginErrors.emailAddress }}
              </div>
            </div>

            <div class="mb-3">
              <label for="loginRole" class="form-label">Role</label>
              <select id="loginRole" v-model="loginForm.role" class="form-select"
                :aria-invalid="loginAttempted && Boolean(loginErrors.role)" aria-describedby="loginRoleFeedback">
                <option value="member">Member</option>
                <option value="admin">Admin</option>
              </select>
              <div v-if="loginAttempted && loginErrors.role" id="loginRoleFeedback" class="invalid-feedback d-block">
                {{ loginErrors.role }}
              </div>
            </div>

            <div class="mb-4">
              <label for="loginPassword" class="form-label">Password</label>
              <input
                id="loginPassword"
                v-model="loginForm.password"
                class="form-control"
                :class="{ 'is-invalid': loginAttempted && Boolean(loginErrors.password) }"
                type="password"
                placeholder="Enter your password"
                required
                :aria-invalid="loginAttempted && Boolean(loginErrors.password)"
                aria-describedby="loginPasswordFeedback"
              >
              <div id="loginPasswordFeedback" class="invalid-feedback">
                {{ loginErrors.password }}
              </div>
            </div>

            <div class="d-grid gap-2">
              <button class="btn btn-climate px-4" type="submit">{{ authBusy ? 'Please wait...' : 'Login' }}</button>
              <button class="btn btn-outline-secondary px-4" type="button" @click="resetLoginForm">
                Reset
              </button>
            </div>
          </form>

          <form v-else novalidate @submit.prevent="registerUser">
            <div class="section-heading">
              <p class="section-kicker">New account</p>
              <h2 class="h4 fw-bold mb-0">Register</h2>
            </div>

            <div class="mb-3">
              <label for="registerFullName" class="form-label">Full name</label>
              <input
                id="registerFullName"
                v-model="registerForm.fullName"
                maxlength="80"
                class="form-control"
                :class="{ 'is-invalid': registerAttempted && Boolean(registerErrors.fullName) }"
                type="text"
                placeholder="e.g. Mia Chen"
                required
                :aria-invalid="registerAttempted && Boolean(registerErrors.fullName)"
                aria-describedby="registerFullNameFeedback"
              >
              <div id="registerFullNameFeedback" class="invalid-feedback">
                {{ registerErrors.fullName }}
              </div>
            </div>

            <div class="mb-3">
              <label for="registerEmailAddress" class="form-label">Email address</label>
              <input
                id="registerEmailAddress"
                v-model="registerForm.emailAddress"
                maxlength="254"
                class="form-control"
                :class="{ 'is-invalid': registerAttempted && Boolean(registerErrors.emailAddress) }"
                type="email"
                placeholder="name@example.com"
                required
                :aria-invalid="registerAttempted && Boolean(registerErrors.emailAddress)"
                aria-describedby="registerEmailAddressFeedback"
              >
              <div id="registerEmailAddressFeedback" class="invalid-feedback">
                {{ registerErrors.emailAddress }}
              </div>
            </div>

            <div class="mb-3">
              <label for="registerRole" class="form-label">Role</label>
              <select id="registerRole" v-model="registerForm.role" class="form-select"
                :aria-invalid="registerAttempted && Boolean(registerErrors.role)" aria-describedby="registerRoleFeedback">
                <option value="member">Member</option>
                <option value="admin">Admin</option>
              </select>
              <div v-if="registerAttempted && registerErrors.role" id="registerRoleFeedback" class="invalid-feedback d-block">
                {{ registerErrors.role }}
              </div>
            </div>

            <div class="mb-3">
              <label for="registerPassword" class="form-label">Password</label>
              <input
                id="registerPassword"
                v-model="registerForm.password"
                maxlength="128"
                class="form-control"
                :class="{ 'is-invalid': registerAttempted && Boolean(registerErrors.password) }"
                type="password"
                placeholder="At least 6 characters"
                required
                :aria-invalid="registerAttempted && Boolean(registerErrors.password)"
                aria-describedby="registerPasswordFeedback"
              >
              <div id="registerPasswordFeedback" class="invalid-feedback">
                {{ registerErrors.password }}
              </div>
            </div>

            <div class="mb-4">
              <label for="registerConfirmPassword" class="form-label">Confirm password</label>
              <input
                id="registerConfirmPassword"
                v-model="registerForm.confirmPassword"
                maxlength="128"
                class="form-control"
                :class="{ 'is-invalid': registerAttempted && Boolean(registerErrors.confirmPassword) }"
                type="password"
                placeholder="Repeat your password"
                required
                :aria-invalid="registerAttempted && Boolean(registerErrors.confirmPassword)"
                aria-describedby="registerConfirmPasswordFeedback"
              >
              <div id="registerConfirmPasswordFeedback" class="invalid-feedback">
                {{ registerErrors.confirmPassword }}
              </div>
            </div>

            <div class="d-grid gap-2">
              <button class="btn btn-climate px-4" type="submit">{{ authBusy ? 'Please wait...' : 'Create account' }}</button>
              <button
                class="btn btn-outline-secondary px-4"
                type="button"
                @click="resetRegisterForm"
              >
                Reset
              </button>
            </div>
          </form>
        </fieldset>
      </div>
    </section>
  </main>

  <main v-else class="app-shell">
    <header class="site-header">
      <div class="container py-3">
        <div class="d-flex flex-column flex-lg-row align-items-lg-center justify-content-between gap-3">
          <div class="d-flex align-items-center gap-3">
            <div class="brand-mark" aria-hidden="true">CW</div>
            <div>
              <p class="small text-uppercase fw-bold text-teal mb-1">
                Version 1
              </p>
              <h1 class="h3 fw-bold mb-0">ClimateWise Melbourne</h1>
            </div>
          </div>

          <section class="account-strip" aria-label="Current user">
            <div class="account-summary">
              <span>Signed in as</span>
              <strong>{{ currentUser.fullName }}</strong>
              <span>{{ isAdmin ? 'Admin' : 'Member' }}</span>
            </div>
            <button class="btn btn-climate btn-sm" type="button" @click="logoutUser">
              Logout
            </button>
          </section>
        </div>
      </div>
    </header>

    <section class="container py-4 py-lg-5">
      <nav class="d-flex gap-3 mb-4" aria-label="Main navigation">
        <a href="#home" :aria-current="currentPage !== '#admin' ? 'page' : null">Learning hub</a>
        <a v-if="isAdmin" href="#admin" :aria-current="currentPage === '#admin' ? 'page' : null">Admin</a>
      </nav>
      <p v-if="currentPage === '#admin' && !isAdmin" role="alert">Access denied. Admin only.</p>
      <div v-if="currentPage !== '#admin'" class="row g-4 align-items-stretch">
        <div class="col-12 col-xl-5">
          <section class="intro-panel h-100">
            <p class="section-kicker">Climate literacy hub</p>
            <h2 class="display-title">Learn climate action in a local way</h2>
            <p class="intro-copy">
              ClimateWise Mel supports students, teachers, and community members with
              beginner-friendly resources about climate change, sustainability, and local action.
            </p>

            <img
              class="feature-image"
              src="/climatewise-melbourne.png"
              alt="Melbourne skyline with climate learning board"
            >

            <div class="row g-3 mt-2">
              <div class="col-6">
                <div class="metric-box">
                  <span>{{ resourceCount }}</span>
                  <small>Resources</small>
                </div>
              </div>
              <div class="col-6">
                <div class="metric-box">
                  <span>{{ audienceCount }}</span>
                  <small>Audiences</small>
                </div>
              </div>
            </div>
          </section>
        </div>

        <div class="col-12 col-xl-7">
          <section class="content-panel h-100">
            <div class="section-heading">
              <p class="section-kicker">Dynamic resource list</p>
              <h2 class="h4 fw-bold mb-0">Featured learning resources</h2>
            </div>

            <div class="row g-3">
              <div
                v-for="resource in learningResources"
                :key="resource.id"
                class="col-12 col-md-4"
              >
                <article class="resource-card h-100">
                  <div class="resource-type">{{ resource.type }}</div>
                  <h3 class="h6 fw-bold">{{ resource.title }}</h3>
                  <p>{{ resource.description }}</p>
                  <span>{{ resource.audience }}</span>
                </article>
              </div>
            </div>
          </section>
        </div>
      </div>

      <div class="row g-4 mt-1">
        <div v-if="currentPage !== '#admin'" class="col-12">
          <section class="border-top pt-4" aria-labelledby="ratingHeading">
            <h2 id="ratingHeading" class="h4 fw-bold">Rate ClimateWise</h2>
            <p aria-live="polite">
              {{ averageRating ? `${averageRating} / 5` : 'No ratings yet.' }}
              <span v-if="ratings.length">({{ ratings.length }} {{ ratings.length === 1 ? 'rating' : 'ratings' }})</span>
            </p>
            <form class="row g-3 align-items-end" novalidate @submit.prevent="submitRating">
              <div class="col-12 col-sm-3">
                <label for="ratingScore" class="form-label">Your rating</label>
                <select id="ratingScore" v-model.number="ratingForm.score" class="form-select" required>
                  <option disabled value="">Choose stars</option>
                  <option v-for="score in 5" :key="score" :value="score">{{ score }} / 5</option>
                </select>
              </div>
              <div class="col-12 col-sm-6">
                <label for="ratingComment" class="form-label">Comment (optional)</label>
                <input id="ratingComment" v-model="ratingForm.comment" class="form-control" maxlength="100">
              </div>
              <div class="col-12 col-sm-3">
                <button class="btn btn-climate w-100" type="submit">{{ myRating ? 'Update rating' : 'Submit rating' }}</button>
              </div>
            </form>
            <p class="small mt-2 mb-0" role="status">{{ ratingMessage }}</p>
          </section>
        </div>

        <div v-if="currentPage !== '#admin'" class="col-12">
          <form class="form-panel" novalidate @submit.prevent="saveInterest">
            <div class="section-heading">
              <p class="section-kicker">Community interest form</p>
              <h2 class="h4 fw-bold mb-0">Save learning interest</h2>
            </div>

            <div class="mb-3">
              <label for="fullName" class="form-label">Full name</label>
              <input
                id="fullName"
                v-model="interestForm.fullName"
                maxlength="80"
                class="form-control"
                :class="fieldState('fullName')"
                type="text"
                placeholder="e.g. Mia Chen"
                required
                :aria-invalid="shouldShowError('fullName') && Boolean(formErrors.fullName)"
                aria-describedby="fullNameFeedback"
                @blur="markTouched('fullName')"
              >
              <div id="fullNameFeedback" class="invalid-feedback">
                {{ formErrors.fullName }}
              </div>
            </div>

            <div class="mb-3">
              <label for="emailAddress" class="form-label">Email address</label>
              <input
                id="emailAddress"
                v-model="interestForm.emailAddress"
                maxlength="254"
                class="form-control"
                :class="fieldState('emailAddress')"
                type="email"
                placeholder="name@example.com"
                required
                :aria-invalid="shouldShowError('emailAddress') && Boolean(formErrors.emailAddress)"
                aria-describedby="emailAddressFeedback"
                @blur="markTouched('emailAddress')"
              >
              <div id="emailAddressFeedback" class="invalid-feedback">
                {{ formErrors.emailAddress }}
              </div>
            </div>

            <div class="row g-3">
              <div class="col-12 col-md-5">
                <label for="postcode" class="form-label">Postcode</label>
                <input
                  id="postcode"
                  v-model="interestForm.postcode"
                  class="form-control"
                  :class="fieldState('postcode')"
                  inputmode="numeric"
                  placeholder="3000"
                  maxlength="4"
                  pattern="[0-9]{4}"
                  required
                  :aria-invalid="shouldShowError('postcode') && Boolean(formErrors.postcode)"
                  aria-describedby="postcodeFeedback"
                  @blur="markTouched('postcode')"
                >
                <div id="postcodeFeedback" class="invalid-feedback">
                  {{ formErrors.postcode }}
                </div>
              </div>

              <div class="col-12 col-md-7">
                <label for="topic" class="form-label">Topic interest</label>
                <select
                  id="topic"
                  v-model="interestForm.topic"
                  class="form-select"
                  :class="fieldState('topic')"
                  required
                  :aria-invalid="shouldShowError('topic') && Boolean(formErrors.topic)"
                  aria-describedby="topicFeedback"
                  @blur="markTouched('topic')"
                >
                  <option v-for="topic in topics" :key="topic">
                    {{ topic }}
                  </option>
                </select>
                <div id="topicFeedback" class="invalid-feedback">
                  {{ formErrors.topic }}
                </div>
              </div>
            </div>

            <div class="mt-3">
              <label for="learningGoal" class="form-label">Learning goal</label>
              <textarea
                id="learningGoal"
                v-model="interestForm.learningGoal"
                maxlength="500"
                class="form-control"
                :class="fieldState('learningGoal')"
                rows="4"
                placeholder="e.g. I want to learn simple actions for my school."
                required
                minlength="10"
                :aria-invalid="shouldShowError('learningGoal') && Boolean(formErrors.learningGoal)"
                aria-describedby="learningGoalFeedback"
                @blur="markTouched('learningGoal')"
              ></textarea>
              <div id="learningGoalFeedback" class="invalid-feedback">
                {{ formErrors.learningGoal }}
              </div>
            </div>

            <div class="d-grid d-sm-flex gap-2 mt-4">
              <button class="btn btn-climate px-4" type="submit">Save interest</button>
              <button
                class="btn btn-outline-secondary px-4"
                type="button"
                @click="resetInterestForm"
              >
                Reset
              </button>
            </div>
          </form>
        </div>

        <div v-if="currentPage !== '#admin' || isAdmin" class="col-12">
          <section class="records-panel">
            <div class="d-flex flex-column flex-md-row justify-content-between gap-3">
              <div>
                <p class="section-kicker">{{ currentPage === '#admin' ? 'Admin' : 'Learning interests' }}</p>
                <h2 class="h4 fw-bold mb-0">{{ currentPage === '#admin' ? 'Saved community interests' : 'My learning interests' }}</h2>
              </div>

              <div class="records-actions">
                <span class="record-count">{{ savedInterestCount }} saved</span>
                <button
                  v-if="isAdmin && currentPage === '#admin'"
                  class="btn btn-outline-secondary btn-sm"
                  type="button"
                  :disabled="savedInterestCount === 0"
                  @click="clearSavedInterests"
                >
                  Clear
                </button>
              </div>
            </div>

            <div v-if="savedInterestCount === 0" class="empty-state mt-3">
              <h3 class="h6 fw-bold">No saved interests yet.</h3>
              <p class="mb-0">Submit the form to add a new record to this dynamic list.</p>
            </div>

            <div v-else class="table-responsive mt-3">
              <table class="table align-middle records-table mb-0">
                <thead>
                  <tr>
                    <th scope="col">Name</th>
                    <th scope="col">Email</th>
                    <th scope="col">Postcode</th>
                    <th scope="col">Topic</th>
                    <th scope="col">Learning goal</th>
                    <th scope="col">Saved</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="interest in visibleInterests" :key="interest.id">
                    <td>{{ interest.fullName }}</td>
                    <td>{{ interest.emailAddress }}</td>
                    <td>{{ interest.postcode }}</td>
                    <td>{{ interest.topic }}</td>
                    <td>{{ interest.learningGoal }}</td>
                    <td>{{ interest.savedAt }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </section>
        </div>
      </div>
    </section>
  </main>
</template>
