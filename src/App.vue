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
              <input type="file" id="file" name="file" accept=".bes,.BES" @change="onFileChange" ref="fileInput" />
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
          <a href="https://github.com/shunito/bes-viewer">Shunsuke Ito の UniBraille Viewer</a>
          を <a href="https://note.com/gesund_fumika">文佳</a> と、 <a href="https://github.com/brlat/brlat.github.io/">brlat</a> が改変・公開したものです。
        </p>
        <p>
          ライセンス：<a href="/LICENSE.txt">MIT License（著作権表示とライセンス全文）</a>
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


// 文字列の最終調整（カッコの調整）
function finalizeText(text: string): string {
  const finalizedText = text
    .replace(/\u2810\u2836/, '\u2800') // 第２カッコ開き
    .replace(/\u2836\u2802/, '\u2800') // 第２カッコ閉じ
    .replace(/\u2830\u2836/, '\u2800') // 二重カッコ開き
    .replace(/\u2836\u2806/, '\u2800') // 二重かっこ閉じ
    .replace(/^\u2800{2}\u2836{2}\u2800/, '\u2800\u2800') // 第１段落挿入符開き
    .replace(/\u2800\u2836{2}$/, '') // 行頭の点訳挿入符
    .replace(/^\u2836{1,2}/, '') // 行移しで行頭に来た第１カッコまたは点訳挿入符
    .replace(/\u2836{1,2}/g, '\u2800') // 第１カッコまたは点訳挿入符
    .replace(/\u2824\u2824/g, '\u2821\u2811\u2800') // 波線を「カラ」に変換
    .replace(/\u2810\u2800/g, '\u2800') // 中点を削除
    .replace(/\u2802{3}/gu, '\u2832\u2800\u2800') // 点線を句点に変換

  // 数字の直後のスペースにハイフンを付ける（スペースを挟んだ後の数字化防止）
  const targetChars = "[\u2801\u2803\u2809\u2819\u2811\u280b\u281b\u2813\u280a\u281a]"
  const regex = new RegExp(`(\\u283c${targetChars}+)\\u2800`, "gu")

  return finalizedText.replace(regex, "$1\u2824\u2800")
}

// BESのユニコードを整形するメイン関数
const formatBrailleText = (text: string): string => {
  if (!text) return ''

  // ユニコードの整理
  const normalizedText = text
    .replace(/@LB@.@PB@/gu, '@LB@@PB@') // 改ページ記号を整理
    .replace(/@LB@@HR@@LB@@LB@/g, '@LB@') // 見出しタグを削除
    .replace(/\u2830\u2806/g, '\u2808\u281d\u2812') // 文中注記符を「チュー」に変換
    .replace(/\u2830\u283c([^\u2800]+)\u2806/gu, '\u2808\u281d\u2812\u2800\u283c$1') // 数字入り文中注記符を「チュー　数字」に変換
    .replace(/\u2800\u2814\u2814/g, '\u2800\u282a\u283f\u2810\u2833\u2819\u2833') // 第１星印を「コメジルシ」に変換
    .replace(/\u2800\u2822\u2822/g, '\u2800\u282e\u2833\u2810\u2833\u2819\u2833') // 第２星印を「ホシジルシ」に変換
    .replace(/\u2820\u2822/g, '\u2835\u2819\u2810\u2833\u2819\u2833') // 第３星印を「マルジルシ」に変換
    .replace(/\u282a\u2812\u2812\u2815/g, '\u2831\u282c\u2812\u2800\u280c\u2810\u2833\u2819\u2833') // 「<-->」を「サユー　ヤジルシ」に変換
    .replace(/\u282a\u2812\u2812/g, '\u2827\u2810\u2815\u2813\u2800\u280c\u2810\u2833\u2819\u2833') // 「<--」を「ヒダリ　ヤジルシ」に変換
    .replace(/\u2812\u2812\u2815/g, '') //  「-->」を削除

  // ----------------------------------------------------
  // 表紙・目次ページの終わりを判定
  // ----------------------------------------------------
  let startIndex = 0

  // 判定条件1: 先頭行が28マス目の数符⠼（\u283c）から始まっているか
  const startsWith28Num = /^\u2800{25,}\u283c/.test(normalizedText)

  if (!startsWith28Num) {
    // 判定条件2: 「@PB@ → スペース複数 → 数符⠼（\u283c）」の並びが現れる最初の位置を探す
    const match = normalizedText.match(/@PB@\u2800+\u283c/)
    if (match && match.index !== undefined) {
      startIndex = match.index
    }
  }

  // ----------------------------------------------------
  // 前半（表紙・目次ページ）と後半（それ以降）に分割
  // ----------------------------------------------------
  const result: string[] = []

  // 前半部分（startIndexより前）をそのままpush
  const beforeText = normalizedText.slice(0, startIndex)
  let pageFrameCount = 0
  let titleFrameCount = 0
  let indexOfTitleLine = 0
  let isFirstPage = true
  let isTocPage = false
  let isStart = true
  let isEnd = true

  // ----------------------------------------------------
  // 前半（表紙・目次ページ）の処理
  // ----------------------------------------------------
  if (beforeText) {
    const lines = beforeText.split(/@LB@/)
    if (lines.length === 0) return ''
    for (let i = 0; i < lines.length; i++) {
      let line = lines[i]

      // 1ページ目の終了を検出
      if (line.includes('@PB@')) {
        if (isFirstPage) {
          result.push('@PB@')
          isFirstPage = false
        }
        continue
      }

      // 1ページ目のみに適用
      if (isFirstPage) {
        // 標題紙の枠線の始まりと終わりを判別
        if (pageFrameCount < 2 && /([^\u2800])\1{27,}/u.test(line)) {
          pageFrameCount++
          continue
        }

        // 標題紙両サイドの枠線を削除
        if (pageFrameCount === 1) {
          line = Array.from(line)
            .slice(1, -1).join('')
        }

        line = line.replace(/^(\u2800)+|(\u2800)+$/g, '')

        // タイトル枠の始まりと終わりを判別
        if(titleFrameCount < 2 && /([^\u2800])\1{10,}/u.test(line)) {
          titleFrameCount++
          continue
        }

        // タイトルを一行に結合
        if (titleFrameCount === 1) {
          line = Array.from(line)
            .slice(1, -1).join('')
            .replace(/^(\u2800)+|(\u2800)+$/g, '')
          if (indexOfTitleLine !== 0) {
            result[result.length - 1] = result[result.length - 1] + '\u2800' + finalizeText(line)
            continue
          }
          indexOfTitleLine++
        }
      }

      // 目次ページ開始から前半終わりまで
      if (!isTocPage && line.includes('\u283e\u2829\u2810\u2833')) {
        result.push(line)
        isTocPage = true
        continue
      }

      // 文字列の末尾が完結しているか判定
      if (isTocPage) {
        if (line === '') {
          isStart = true
          isEnd = true
        } else if (line.charAt(28) === '\u283c') {
          const firstPart = line.slice(0,29)
          const secondPart = line.slice(29)
          const replacedFirstPart = firstPart
            .replace(/\u2800[\u2802\u2810]{2,}\u2800\u283c/u, '\u2800\u2800\u283c')
          line = replacedFirstPart + secondPart
          isEnd = true
        } else if (/\u2836\u2836\u281d\u2810\u281d\u2823\u2836\u2836/u.test(line)) {
          isEnd = true
        } else {
          isEnd = false
        }

        // 文字列の先頭の状態に応じて出力し、末尾の状態に応じて次行の先頭の状態を設定
        if (isStart) {
          if (!isEnd) isStart = false
        } else {
          line = line.replace(/^\u2800+/, '\u2800')
          result[result.length - 1] = result[result.length - 1] + finalizeText(line)
          if (isEnd) isStart = true
          continue
        }
      }

      if (line !== '') result.push(finalizeText(line))
    }
  }

  // ----------------------------------------------------
  // 後半の処理
  // ----------------------------------------------------
  const targetText = normalizedText.slice(startIndex)
  const convertedText = targetText
    .replace(/@LB@@PB@/g, '@LB@')
  const lines = convertedText.split(/@LB@/)
  if (lines.length === 0) return ''

  let currentParagraphIndent = -1
  let isInBlock = false
  const Chars = "[\u2813\u2816\u2836\u281b]"
  const Zorome = "([\\u2812\\u2802\\u2810])\\1{10,}"
  const regex2 = new RegExp(`${Chars}${Zorome}`, 'u')

  for (let i = 0; i < lines.length; i++) {
    let line = lines[i]

    // ページ番号行はスキップ（半角スペースが25個以上のあとに数符）
    if (/\u2800{25,}[\u283c]/.test(line)) {
      continue
    }

    // 枠線の行は空行に
    if (regex2.test(line) || /\u2812{10,}/.test(line)) {
      result.push('')
      if (/[\u2816\u2836]/.test(line)) {
        isInBlock = true
      } else if (/[\u2813\u281b]/.test(line)) {
        isInBlock = false
      }
      continue
    }

    // 空行の場合はそのまま結果に追加してインデント状態をリセット
    if (line.length === 0) {
      result.push('')
      currentParagraphIndent = -1
      continue
    }

    // 行頭のスペース数をカウント
    const spaceMatch = line.match(/^(\u2800+)/)
    const spaceCount = spaceMatch ? spaceMatch[1].length : 0

    // 最初の1行目の処理
    if (result.length === 0) {
      result.push(finalizeText(line))
      if (spaceCount >= 4) {
        currentParagraphIndent = spaceCount
      }
      continue
    }

    // インデント規則に基づいて改行を詰める
    const prevLine = result[result.length - 1]

    // 結合条件1: 次の行の行頭に \u2800 がない場合（改行削除してそのまま連結）
    if (spaceCount === 0) {
      // つなぎ目がくっつきすぎないよう、前行末尾に \u2800 がなければ補う
      const needsSpace = prevLine.length > 0 && !prevLine.endsWith('\u2800')
      result[result.length - 1] = prevLine + (needsSpace ? '\u2800' : '') + finalizeText(line)
      continue
    }

    // 結合条件2: 前行行頭の \u2800 が4マス以上あり、次行が「+2マス」のインデントで続く場合
    if (currentParagraphIndent >= 4 && spaceCount === currentParagraphIndent + 2) {
      // 行頭の \u2800 を 1マス分 に縮小して結合
      const trimmedContent = line.replace(/^\u2800+/, '\u2800')
      result[result.length - 1] = prevLine + finalizeText(trimmedContent)
      continue
    }

    // 上記の結合条件に当てはまらない場合（新しい段落の開始など）
    if (spaceCount >= 4 && spaceCount < 10) {
      if (isInBlock) {
        if (result[result.length - 1] !== '') {
          result.push('@LB@' + finalizeText(line))
        } else {
          result.push(finalizeText(line))
        }
      } else {
        result[result.length - 1] = result[result.length - 1] + '@PB@' + finalizeText(line)
      }
      currentParagraphIndent = spaceCount
    } else {
      currentParagraphIndent = -1
      result.push(finalizeText(line))
    }
  }

  return result.join('@LB@')
}

const onFileChange = (event: Event) => {
  const input = event.target as HTMLInputElement
  const files = input.files
  if (!files || !files.length) return

  const file = files[0]
  if (!file.name.toLowerCase().endsWith('.bes')) {
    window.alert('.bes点字データではありません')
    input.value = ''
    return
  }

  const reader = new FileReader()
  reader.onloadend = (theFile) => {
    const target = theFile.target as FileReader
    if (target && target.readyState === FileReader.DONE) {
      openFile.value = true
      isBrf.value = false
      const result = target.result as ArrayBuffer
      const arr = new Uint8Array(result)
      str.value = formatBrailleText(bes2unicode(arr))
    }
  }

  reader.readAsArrayBuffer(file)
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
      str.value = formatBrailleText(bes2unicode(new Uint8Array(buf)))
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
