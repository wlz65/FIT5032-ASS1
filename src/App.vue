<script setup>
import { computed, onMounted, reactive, ref, watch } from 'vue'

const STORAGE_KEY = 'climatewise-learning-interests'

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

const formErrors = computed(() => {
  const errors = {}
  const fullName = interestForm.fullName.trim()
  const emailAddress = interestForm.emailAddress.trim()
  const postcode = interestForm.postcode.trim()
  const learningGoal = interestForm.learningGoal.trim()

  if (!fullName) {
    errors.fullName = 'Full name is required.'
  }

  if (!emailAddress) {
    errors.emailAddress = 'Email address is required.'
  } else if (!emailPattern.test(emailAddress)) {
    errors.emailAddress = 'Enter a valid email address.'
  }

  if (!postcode) {
    errors.postcode = 'Postcode is required.'
  } else if (!postcodePattern.test(postcode)) {
    errors.postcode = 'Enter a 4-digit postcode.'
  }

  if (!interestForm.topic) {
    errors.topic = 'Choose one topic interest.'
  }

  if (!learningGoal) {
    errors.learningGoal = 'Learning goal is required.'
  } else if (learningGoal.length < 10) {
    errors.learningGoal = 'Write at least 10 characters.'
  }

  return errors
})

const formIsValid = computed(() => Object.keys(formErrors.value).length === 0)

const savedInterestCount = computed(() => savedInterests.value.length)

onMounted(() => {
  const storedInterests = localStorage.getItem(STORAGE_KEY)

  if (!storedInterests) {
    return
  }

  try {
    const parsedInterests = JSON.parse(storedInterests)

    if (Array.isArray(parsedInterests)) {
      savedInterests.value = parsedInterests
    }
  } catch {
    localStorage.removeItem(STORAGE_KEY)
  }
})

watch(
  savedInterests,
  (records) => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(records))
  },
  { deep: true },
)

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
  submitAttempted.value = true
  Object.keys(touched).forEach((field) => {
    touched[field] = true
  })

  if (!formIsValid.value) {
    return
  }

  savedInterests.value.unshift({
    id: Date.now(),
    fullName: interestForm.fullName.trim(),
    emailAddress: interestForm.emailAddress.trim(),
    postcode: interestForm.postcode.trim(),
    topic: interestForm.topic,
    learningGoal: interestForm.learningGoal.trim(),
    savedAt: new Date().toLocaleDateString('en-AU', {
      day: '2-digit',
      month: 'short',
      year: 'numeric',
    }),
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
  savedInterests.value = []
}
</script>

<template>
  <main class="app-shell">
    <header class="site-header">
      <div class="container py-3">
        <div class="d-flex flex-column flex-md-row align-items-md-center justify-content-between gap-3">
          <div class="d-flex align-items-center gap-3">
            <div class="brand-mark" aria-hidden="true">CW</div>
            <div>
              <p class="small text-uppercase fw-bold text-teal mb-1">
                Version 1 
              </p>
              <h1 class="h3 fw-bold mb-0">ClimateWise Melbourne</h1>
            </div>
          </div>
          <div class="header-note">
            A basic Vue application for the Assignment 1.
          </div>
        </div>
      </div>
    </header>

    <section class="container py-4 py-lg-5">
      <div class="row g-4 align-items-stretch">
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
        <div class="col-12">
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

        <div class="col-12">
          <section class="records-panel">
            <div class="d-flex flex-column flex-md-row justify-content-between gap-3">
              <div>
                <p class="section-kicker">Local storage data</p>
                <h2 class="h4 fw-bold mb-0">Saved community interests</h2>
              </div>

              <div class="records-actions">
                <span class="record-count">{{ savedInterestCount }} saved</span>
                <button
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
                  <tr v-for="interest in savedInterests" :key="interest.id">
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
