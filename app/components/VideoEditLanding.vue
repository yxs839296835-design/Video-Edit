<script setup lang="ts">
import { computed, ref } from 'vue'
import { videoEditLocale } from '../locales/video-edit'

const t = videoEditLocale.zh
const activeFaq = ref<number | null>(0)
const isWorkspaceActive = ref(false)

const uploadCards = [
  {
    icon: '/video-edit-assets/images/online-edit/upload-file.svg',
    title: t.uploadStepTitle,
    description: t.uploadStepDescription,
  },
  {
    icon: '/video-edit-assets/images/online-edit/export-video.svg',
    title: t.editStepTitle,
    description: t.editStepDescription,
  },
  {
    icon: '/video-edit-assets/images/online-edit/edit-video.svg',
    title: t.exportStepTitle,
    description: t.exportStepDescription,
  },
]

const steps = [
  {
    icon: '/video-edit-assets/images/editor/use-step-icon1.svg',
    title: t.uploadStepTitle,
    description: t.uploadStepDescription,
  },
  {
    icon: '/video-edit-assets/images/editor/use-step-icon2.svg',
    title: t.editStepTitle,
    description: t.editStepDescription,
  },
  {
    icon: '/video-edit-assets/images/editor/use-step-icon3.svg',
    title: t.exportStepTitle,
    description: t.exportStepDescription,
  },
]

const features = [
  {
    icon: '/video-edit-assets/icons/editor/main-feature-icon1.svg',
    image: '/video-edit-assets/images/editor/main-feature-img1.png',
    title: t.feature1Title,
    description: t.feature1Description,
  },
  {
    icon: '/video-edit-assets/icons/editor/main-feature-icon2.svg',
    image: '/video-edit-assets/images/editor/main-feature-img2.png',
    title: t.feature2Title,
    description: t.feature2Description,
  },
  {
    icon: '/video-edit-assets/icons/editor/main-feature-icon3.svg',
    image: '/video-edit-assets/images/editor/main-feature-img3.png',
    title: t.feature3Title,
    description: t.feature3Description,
  },
  {
    icon: '/video-edit-assets/icons/editor/main-feature-icon4.svg',
    image: '/video-edit-assets/images/editor/main-feature-img4.png',
    title: t.feature4Title,
    description: t.feature4Description,
  },
  {
    icon: '/video-edit-assets/icons/editor/main-feature-icon5.svg',
    image: '/video-edit-assets/images/editor/main-feature-img5.png',
    title: t.feature5Title,
    description: t.feature5Description,
  },
]

const scenes = [
  {
    icon: '/video-edit-assets/icons/editor/use-scene-icon1.svg',
    title: t.scene1Title,
    description: t.scene1Description,
  },
  {
    icon: '/video-edit-assets/icons/editor/use-scene-icon2.svg',
    title: t.scene2Title,
    description: t.scene2Description,
  },
  {
    icon: '/video-edit-assets/icons/editor/use-scene-icon3.svg',
    title: t.scene3Title,
    description: t.scene3Description,
  },
  {
    icon: '/video-edit-assets/icons/editor/use-scene-icon4.svg',
    title: t.scene4Title,
    description: t.scene4Description,
  },
  {
    icon: '/video-edit-assets/icons/editor/use-scene-icon5.svg',
    title: t.scene5Title,
    description: t.scene5Description,
  },
  {
    icon: '/video-edit-assets/icons/editor/use-scene-icon6.svg',
    title: t.scene6Title,
    description: t.scene6Description,
  },
]

const faqJsonLd = computed(() => ({
  '@context': 'https://schema.org',
  '@type': 'FAQPage',
  mainEntity: t.faqs.map((faq) => ({
    '@type': 'Question',
    name: faq.question,
    acceptedAnswer: {
      '@type': 'Answer',
      text: faq.answer,
    },
  })),
}))

useHead({
  title: t.metaTitle,
  meta: [
    { name: 'description', content: t.metaDescription },
    { property: 'og:title', content: t.metaTitle },
    { property: 'og:description', content: t.metaDescription },
    { property: 'og:type', content: 'website' },
    { name: 'twitter:card', content: 'summary_large_image' },
  ],
  script: [
    {
      type: 'application/ld+json',
      children: JSON.stringify(faqJsonLd.value),
    },
  ],
})

function scrollToEditor() {
  document.querySelector('#video-editor')?.scrollIntoView({ behavior: 'smooth', block: 'center' })
}
</script>

<template>
  <div class="video-edit-page" :class="{ 'workspace-active': isWorkspaceActive }">
    <header class="site-header">
      <a class="brand" href="/" aria-label="Video Edit">
        <span class="brand-mark">V</span>
        <span>{{ t.brand }}</span>
      </a>
    </header>

    <main>
      <section class="hero-section" :class="{ 'workspace-hero': isWorkspaceActive }">
        <img
          class="hero-bg"
          src="/video-edit-assets/images/editor/hero-bg.png"
          alt=""
          aria-hidden="true"
        >
        <div class="hero-inner">
          <div v-if="!isWorkspaceActive" class="hero-copy">
            <h1 v-html="t.heroTitle" />
            <p>{{ t.heroDescription }}</p>
          </div>

          <BrowserVideoEditor
            id="video-editor"
            :upload-cards="uploadCards"
            @workspace-change="isWorkspaceActive = $event"
          />
        </div>
      </section>

      <template v-if="!isWorkspaceActive">
      <section id="tools" class="step-section">
        <div class="section-heading">
          <h2>{{ t.stepsTitle }}</h2>
          <p>{{ t.stepsDescription }}</p>
        </div>
        <ul class="step-list">
          <li v-for="(step, index) in steps" :key="step.title">
            <img :src="step.icon" :alt="step.title" draggable="false">
            <div class="step-title">
              <span>{{ index + 1 }}</span>
              <h3>{{ step.title }}</h3>
            </div>
            <p>{{ step.description }}</p>
          </li>
        </ul>
      </section>

      <section id="features" class="feature-section">
        <div class="section-heading">
          <h2>{{ t.featuresTitle }}</h2>
          <p>{{ t.featuresDescription }}</p>
        </div>
        <ul class="feature-list">
          <li
            v-for="(feature, index) in features"
            :key="feature.title"
            :class="{ reverse: index % 2 === 1 }"
          >
            <div class="feature-copy">
              <div class="feature-title">
                <img :src="feature.icon" :alt="feature.title" draggable="false">
                <h3>{{ feature.title }}</h3>
              </div>
              <p>{{ feature.description }}</p>
            </div>
            <img class="feature-image" :src="feature.image" :alt="feature.title" loading="lazy">
          </li>
        </ul>
      </section>

      <section class="scene-section">
        <div class="section-heading">
          <h2>{{ t.sceneTitle }}</h2>
          <p>{{ t.sceneDescription }}</p>
        </div>
        <ul class="scene-list">
          <li v-for="scene in scenes" :key="scene.title">
            <img class="hover-bg" src="/video-edit-assets/icons/ai-youtube/drama-advantage-bg.svg" alt="" aria-hidden="true">
            <img class="hover-rec" src="/video-edit-assets/icons/ai-youtube/drama-advantage-rec.svg" alt="" aria-hidden="true">
            <div class="scene-title">
              <img :src="scene.icon" :alt="scene.title" draggable="false">
              <h3>{{ scene.title }}</h3>
            </div>
            <p>{{ scene.description }}</p>
          </li>
        </ul>
      </section>

      <section id="faq" class="faq-section">
        <div class="section-heading">
          <h2>{{ t.faqTitle }}</h2>
          <p>{{ t.faqDescription }}</p>
        </div>
        <ul class="faq-list">
          <li v-for="(faq, index) in t.faqs" :key="faq.question">
            <button
              type="button"
              :class="{ active: activeFaq === index }"
              @click="activeFaq = activeFaq === index ? null : index"
            >
              <span>{{ faq.question }}</span>
              <span class="faq-arrow">⌄</span>
            </button>
            <p v-show="activeFaq === index">{{ faq.answer }}</p>
          </li>
        </ul>
      </section>

      <section class="comment-section">
        <div class="section-heading">
          <h2>{{ t.commentsTitle }}</h2>
        </div>
        <ul class="comment-list">
          <li v-for="comment in t.comments" :key="comment.name">
            <p>{{ comment.comment }}</p>
            <div>
              <img :src="comment.avatar" :alt="comment.name" loading="lazy">
              <span>
                <strong>{{ comment.name }}</strong>
                <small>{{ comment.profession }}</small>
              </span>
            </div>
          </li>
        </ul>
      </section>

      <section id="cta" class="cta-section">
        <img
          class="cta-bg"
          src="/video-edit-assets/images/ai-translate/ai-translate-last-bg.png"
          alt=""
          aria-hidden="true"
        >
        <div class="cta-content">
          <h2>{{ t.ctaTitle }}</h2>
          <ul>
            <li v-for="point in t.ctaPoints" :key="point">
              <img src="/video-edit-assets/icons/text-to-speech-scene/last-narrow.svg" alt="" aria-hidden="true">
              <span>{{ point }}</span>
            </li>
          </ul>
          <button class="gradient-button cta-button" type="button" @click="scrollToEditor">
            {{ t.startEditing }}
            <img src="/video-edit-assets/icons/editor/star.svg" alt="" aria-hidden="true">
          </button>
        </div>
      </section>
      </template>
    </main>

    <footer v-if="!isWorkspaceActive" class="site-footer">
      <div>
        <a class="brand" href="/" aria-label="Video Edit">
          <span class="brand-mark">V</span>
          <span>{{ t.brand }}</span>
        </a>
        <p>{{ t.footerDescription }}</p>
      </div>
      <nav aria-label="Footer navigation">
        <a href="#tools">{{ t.footerProduct }}</a>
        <a href="#features">{{ t.footerCompany }}</a>
        <a href="#faq">{{ t.footerSupport }}</a>
      </nav>
    </footer>
  </div>
</template>

<style scoped>
:global(html) {
  background: #000;
  scroll-behavior: smooth;
}

:global(body) {
  margin: 0;
  background: #000;
  color: #fff;
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
    "Microsoft YaHei", sans-serif;
}

:global(*) {
  box-sizing: border-box;
}

.video-edit-page {
  min-height: 100vh;
  overflow: hidden;
  background: #000;
  color: #fff;
}

.video-edit-page.workspace-active {
  height: 100vh;
}

.workspace-active main {
  height: calc(100vh - 72px);
  overflow: hidden;
}

.site-header {
  position: sticky;
  top: 0;
  z-index: 50;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 24px;
  height: 72px;
  padding: 0 clamp(20px, 5vw, 72px);
  border-bottom: 1px solid rgb(255 255 255 / 8%);
  background: rgb(0 0 0 / 78%);
  backdrop-filter: blur(24px);
}

.brand {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  color: #fff;
  font-size: 20px;
  font-weight: 800;
  text-decoration: none;
}

.brand-mark {
  display: grid;
  width: 34px;
  height: 34px;
  place-items: center;
  border-radius: 10px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #0b1020;
}

.site-footer nav a {
  color: rgb(255 255 255 / 72%);
  font-size: 15px;
  text-decoration: none;
}

.site-footer nav a:hover {
  color: #fff;
}

button,
label {
  font: inherit;
}

.gradient-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 188px;
  min-height: 56px;
  padding: 0 24px;
  border: 0;
  border-radius: 999px;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #091018;
  cursor: pointer;
  font-weight: 800;
  text-decoration: none;
  box-shadow: 0 18px 46px rgb(128 250 163 / 22%);
}

.hero-section {
  position: relative;
  min-height: 940px;
  padding: 90px 0 76px;
  background: #000;
}

.hero-section.workspace-hero {
  height: calc(100vh - 72px);
  min-height: 0;
  padding: 16px 0;
  overflow: hidden;
}

.hero-bg,
.cta-bg {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  pointer-events: none;
}

.hero-inner {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 70px;
  width: min(92vw, 1320px);
  margin: 0 auto;
}

.workspace-hero .hero-inner {
  gap: 0;
  height: 100%;
  width: min(96vw, 1880px);
}

.hero-copy {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
  text-align: center;
}

.hero-copy h1 {
  max-width: 1200px;
  margin: 0;
  font-size: clamp(32px, 5vw, 54px);
  line-height: 1.16;
  font-weight: 800;
}

.hero-copy :deep(i),
.section-heading :deep(i),
.gradient-text {
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  font-style: normal;
}

.hero-copy p {
  max-width: 1100px;
  margin: 0;
  color: rgb(255 255 255 / 80%);
  font-size: clamp(16px, 2.1vw, 24px);
  line-height: 1.45;
}

.step-section,
.feature-section,
.scene-section,
.faq-section,
.comment-section {
  position: relative;
  background: #000;
  color: #fff;
}

.step-section {
  padding: clamp(60px, 15vh, 150px) 0 80px;
}

.section-heading {
  width: min(90vw, 1180px);
  margin: 0 auto;
  text-align: center;
}

.section-heading h2 {
  margin: 0;
  font-size: clamp(28px, 4.5vw, 48px);
  line-height: 1.2;
  font-weight: 800;
}

.section-heading p {
  max-width: 900px;
  margin: 20px auto 0;
  color: rgb(255 255 255 / 78%);
  font-size: clamp(16px, 1.9vw, 20px);
  line-height: 1.45;
}

.step-list {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: clamp(20px, 4vw, 60px);
  width: min(90vw, 1200px);
  margin: 48px auto 0;
  padding: 0;
  list-style: none;
}

.step-list li {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40px 36px;
  border-radius: 16px;
  background: rgb(36 44 69 / 62%);
  backdrop-filter: blur(40px);
}

.step-list img {
  width: 90px;
  height: 90px;
}

.step-title {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 24px;
}

.step-title span {
  position: relative;
  display: grid;
  width: 28px;
  height: 28px;
  place-items: center;
  border-radius: 999px;
  background: linear-gradient(90deg, rgb(214 251 114 / 50%), rgb(118 249 177 / 50%));
  color: #fff;
  font-size: 15px;
  font-weight: 800;
}

.step-title h3 {
  margin: 0;
  font-size: 22px;
}

.step-list p {
  margin: 16px 0 0;
  color: rgb(255 255 255 / 60%);
  font-size: 16px;
  line-height: 1.55;
}

.feature-section {
  padding: 80px 0 150px;
}

.feature-list {
  display: flex;
  flex-direction: column;
  gap: clamp(58px, 10vw, 180px);
  width: min(90vw, 1300px);
  margin: 90px auto 0;
  padding: 0;
  list-style: none;
}

.feature-list li {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 58px;
}

.feature-list li.reverse {
  flex-direction: row-reverse;
}

.feature-copy {
  max-width: 530px;
}

.feature-title {
  display: flex;
  align-items: center;
  gap: 12px;
}

.feature-title img {
  width: 54px;
  height: 54px;
}

.feature-title h3 {
  margin: 0;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  font-size: clamp(26px, 4vw, 42px);
  font-weight: 800;
}

.feature-copy p {
  margin: 24px 0 0;
  font-size: clamp(16px, 2vw, 20px);
  line-height: 1.8;
}

.feature-image {
  width: min(54vw, 698px);
  max-width: 100%;
}

.scene-section {
  padding: 0 0 70px;
}

.scene-list {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 30px;
  width: min(90vw, 1200px);
  margin: 60px auto 0;
  padding: 0;
  list-style: none;
}

.scene-list li {
  position: relative;
  min-height: 190px;
  overflow: hidden;
  padding: 30px 24px;
  border: 2px solid transparent;
  border-radius: 16px;
  background: rgb(185 236 255 / 12%);
  cursor: default;
  backdrop-filter: blur(40px);
}

.scene-list li:hover {
  border-color: #2a3e25;
}

.hover-bg,
.hover-rec {
  position: absolute;
  display: none;
  pointer-events: none;
}

.hover-bg {
  top: 16px;
  right: -36px;
}

.hover-rec {
  bottom: 0;
  left: 18px;
  transform: translateY(30%);
}

.scene-list li:hover .hover-bg,
.scene-list li:hover .hover-rec {
  display: block;
}

.scene-title {
  position: relative;
  z-index: 1;
  display: flex;
  align-items: center;
  gap: 8px;
}

.scene-title img {
  width: 30px;
  height: 30px;
}

.scene-title h3 {
  margin: 0;
  font-size: 20px;
  font-weight: 800;
}

.scene-list p {
  position: relative;
  z-index: 1;
  margin: 18px 0 0;
  color: rgb(255 255 255 / 60%);
  font-size: 14px;
  line-height: 1.72;
}

.faq-section {
  padding: 70px 0 110px;
}

.faq-list {
  width: min(90vw, 860px);
  margin: 46px auto 0;
  padding: 0;
  list-style: none;
}

.faq-list li {
  border-bottom: 1px solid rgb(255 255 255 / 20%);
}

.faq-list button {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  min-height: 78px;
  padding: 12px 0;
  border: 0;
  background: transparent;
  color: #fff;
  cursor: pointer;
  text-align: left;
  font-size: 20px;
  font-weight: 750;
}

.faq-list button.active span:first-child {
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
}

.faq-arrow {
  display: grid;
  width: 26px;
  height: 26px;
  flex: 0 0 auto;
  place-items: center;
  border-radius: 999px;
  background: rgb(255 255 255 / 20%);
  transition: transform 0.25s ease;
}

.faq-list button.active .faq-arrow {
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  color: #000;
  transform: rotate(180deg);
}

.faq-list p {
  margin: 0;
  padding: 0 42px 22px 0;
  color: rgb(255 255 255 / 60%);
  font-size: 15px;
  line-height: 1.75;
}

.comment-section {
  padding: 0 0 110px;
}

.comment-list {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 22px;
  width: min(90vw, 1200px);
  margin: 56px auto 0;
  padding: 0;
  list-style: none;
}

.comment-list li {
  display: flex;
  min-height: 280px;
  flex-direction: column;
  justify-content: space-between;
  padding: 26px;
  border: 1px solid rgb(255 255 255 / 10%);
  border-radius: 18px;
  background: rgb(36 44 69 / 62%);
}

.comment-list p {
  margin: 0;
  color: rgb(255 255 255 / 72%);
  font-size: 15px;
  line-height: 1.72;
}

.comment-list div {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 26px;
}

.comment-list img {
  width: 46px;
  height: 46px;
  border-radius: 999px;
  object-fit: cover;
}

.comment-list span {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.comment-list strong {
  color: #fff;
  font-size: 15px;
}

.comment-list small {
  color: rgb(255 255 255 / 52%);
}

.cta-section {
  position: relative;
  min-height: 520px;
  padding: 120px 0;
  background: #000;
}

.cta-content {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  width: min(90vw, 1100px);
  margin: 0 auto;
  text-align: center;
}

.cta-content h2 {
  max-width: 980px;
  margin: 0;
  background: linear-gradient(90deg, #d6fb72, #76f9b1);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  font-size: clamp(32px, 5vw, 54px);
  font-weight: 900;
  line-height: 1.18;
}

.cta-content ul {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 20px 50px;
  margin: 30px 0 0;
  padding: 0;
  list-style: none;
}

.cta-content li {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 18px;
}

.cta-button {
  min-width: 270px;
  min-height: 64px;
  margin-top: 50px;
  font-size: 22px;
}

.cta-button img {
  width: 24px;
  height: 24px;
  margin-left: 6px;
}

.site-footer {
  display: flex;
  justify-content: space-between;
  gap: 32px;
  padding: 46px clamp(20px, 5vw, 72px);
  border-top: 1px solid rgb(255 255 255 / 10%);
  background: #030509;
}

.site-footer p {
  max-width: 420px;
  margin: 14px 0 0;
  color: rgb(255 255 255 / 55%);
  line-height: 1.7;
}

.site-footer nav {
  display: flex;
  gap: 26px;
  align-items: flex-start;
}

@media (max-width: 1023px) {
  .site-header {
    height: 64px;
  }

  .hero-section {
    min-height: auto;
    padding: 62px 0 50px;
  }

  .hero-section.workspace-hero {
    height: calc(100vh - 64px);
    min-height: 0;
    padding: 12px 0;
  }

  .hero-inner {
    gap: 42px;
  }

  .step-list,
  .scene-list,
  .comment-list {
    grid-template-columns: 1fr;
  }

  .feature-list {
    margin-top: 50px;
  }

  .feature-list li,
  .feature-list li.reverse {
    flex-direction: column;
    gap: 30px;
  }

  .feature-copy {
    max-width: 100%;
  }

  .feature-image {
    width: 100%;
  }

  .comment-list li {
    min-height: 220px;
  }

  .site-footer {
    flex-direction: column;
  }
}

@media (max-width: 640px) {
  .site-header {
    padding: 0 16px;
  }

  .brand {
    font-size: 17px;
  }

  .brand-mark {
    width: 30px;
    height: 30px;
    border-radius: 9px;
  }

  .step-list li {
    padding: 34px 26px;
  }

  .feature-section {
    padding-bottom: 80px;
  }

  .feature-title img {
    width: 44px;
    height: 44px;
  }

  .faq-list button {
    min-height: 68px;
    font-size: 17px;
  }

  .cta-content ul {
    flex-direction: column;
  }

  .site-footer nav {
    flex-wrap: wrap;
  }
}
</style>
