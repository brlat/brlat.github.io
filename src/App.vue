<template>
  <div id="app">
    <header>
      <nav class="navbar is-dark" :title="t('title')" id="main-header">
        <div class="navbar-brand">
          <h1 class="navbar-item">{{ t('title') }}</h1>
        </div>

        <div id="navbarMenu" class="navbar-menu is-active">
          <div class="navbar-start">
            <div class="navbar-item">
              <label for="file">{{ t('selectFile') }}</label>
              <input type="file" id="file" name="file" accept=".bes,.brf,.brl,.bse" @change="onFileChange" ref="fileInput" />
            </div>
            <div class="navbar-item">
              <button class="button is-small is-light" id="closeFile" :disabled="isFileClosed" @click="onFileClose">{{ t('closeFile') }}</button>
            </div>
          </div>
          <div class="navbar-end">
            <p class="navbar-item"><hr></p>
          </div>
        </div>
      </nav>
    </header>

    <div class="container is-fluid">
        <main class="section">
          <Braille :braille="bes" :isBrf="isBrf"></Braille>
        </main>
    </div>

    <footer class="footer">
      <div class="content has-text-centered">
        <hr>
        <p>
          これは、
          <a href="https://github.com/shunito/bes-viewer">UniBraille Viewer by Shunsuke Ito</a>
          を変更して
          <a href="https://github.com/brlat/brlat.github.io/">brlat</a> が作成しました。
        </p>
        <p>
          ライセンスは
          <a href="https://opensource.org/license/mit">MIT</a> です。
        </p>
      </div>
    </footer>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import Braille from './components/Braille.vue'
import bes2unicode from './modules/bes2unicode'
import brf2unicode from './modules/brf2unicode'
import { useI18n } from './modules/i18n'

const { t, locale } = useI18n()
locale.value = 'ja'

const openFile = ref(false)
const isBrf = ref(false)
const str = ref('')
const fileInput = ref<HTMLInputElement | null>(null)

const bes = computed(() => str.value)
const isFileClosed = computed(() => !openFile.value)


const onFileChange = (event: Event) => {
  const input = event.target as HTMLInputElement
  const files = input.files
  if (!files || !files.length) return

  const file = files[0]
  const ext = file.name.toLowerCase().split('.').pop()
  const isBrfFile = ext === 'brf' || ext === 'brl' || ext === 'bse'

  const reader = new FileReader()
  reader.onloadend = (theFile) => {
    const target = theFile.target as FileReader
    if (target && target.readyState === FileReader.DONE) {
      openFile.value = true
      isBrf.value = isBrfFile
      if (isBrfFile) {
        const text = target.result as string
        str.value = brf2unicode(text)
      } else {
        const result = target.result as ArrayBuffer
        const arr = new Uint8Array(result)
        str.value = bes2unicode(arr)
      }
    }
  }

  if (isBrfFile) {
    reader.readAsText(file)
  } else {
    reader.readAsArrayBuffer(file)
  }
}

const onFileClose = () => {
  str.value = ''
  openFile.value = false
  isBrf.value = false
  if (fileInput.value) fileInput.value.value = ''
}

const onGetBesUrl = async (url: string) => {
  const isBrfUrl = url.toLowerCase().endsWith('.brf')
  try {
    const response = await fetch(url, { method: 'GET' })
    isBrf.value = isBrfUrl
    if (isBrfUrl) {
      const text = await response.text()
      str.value = brf2unicode(text)
    } else {
      const buf = await response.arrayBuffer()
      str.value = bes2unicode(new Uint8Array(buf))
    }
    openFile.value = true
  } catch (error) {
    console.error(error)
  }
}

onMounted(() => {
  const params = new URL(window.location.href).searchParams
  const targetUrl = params.get('url')
  if (targetUrl && targetUrl.length > 5) {
    const ext = targetUrl.slice(-4).toLowerCase()
    if (ext === '.bes' || ext === '.brf') {
      onGetBesUrl(targetUrl)
    }
  }
})
</script>

<style>
.footer a {
  text-decoration: underline;
}
</style>
