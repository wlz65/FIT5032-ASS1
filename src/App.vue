<script setup>
import { computed, reactive } from 'vue'

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

const interestForm = reactive({
  fullName: '',
  emailAddress: '',
})
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
                Version 1 - BR A and B 
              </p>
              <h1 class="h3 fw-bold mb-0">ClimateWise Melbourne</h1>
            </div>
          </div>
          <div class="header-note">
            A basic Vue application for the Assignment 1 A and B requirements.
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
              This version focuses only on the required A and B features.
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
        <div class="col-12 col-lg-7">
          <form class="form-panel">
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
                type="text"
                placeholder="e.g. Mia Chen"
              >
            </div>

            <div class="mb-0">
              <label for="emailAddress" class="form-label">Email address</label>
              <input
                id="emailAddress"
                v-model="interestForm.emailAddress"
                class="form-control"
                type="email"
                placeholder="name@example.com"
              >
            </div>
          </form>
        </div>
      </div>
    </section>
  </main>
</template>
