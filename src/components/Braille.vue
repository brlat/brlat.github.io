<template>
  <p v-if="isFileClose" class="is-size-2 is-size-4-mobile">{{ t('placeholder') }}</p>
  <div v-else class="besbody is-size-3 is-size-5-mobile" role="document">
    <article>
      <section v-for="(page,pno) in bes.body" :key="pno" class="columns page">
        <div class="column yomi">
          <template v-for="(line,lno) in page" :key="lno">
            <hr v-if="line === '@HR@'">
            <p v-else-if="line.length === 0"><br /></p>
            <p v-else>{{ tenji2kana(line) }}</p>
          </template>
        </div>
      </section>
    </article>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import * as tenji from 'tenji'
// OIcon removed; not needed for tests
import { splitbraille, type ParsedBraille } from '@/modules/brailleParser'
import { unicode2brf } from '../modules/brf2unicode'
import { decodeUEB } from '../modules/uebDecoder'
import { useI18n } from '../modules/i18n'

const { t } = useI18n()

const props = withDefaults(defineProps<{
  braille?: string;
  isBrf?: boolean;
}>(), {
  braille: '',
  isBrf: false
})

const isFileClose = computed(() => props.braille.length === 0)
const bes = computed((): ParsedBraille => splitbraille(props.braille))

function toKatakana(value: string): string {
  return value.replace(/[\u3041-\u3096\u309D-\u309F]/g, char =>
    String.fromCharCode(char.charCodeAt(0) + 0x60)
  )
}

function tenji2kana(str: string | false): string {
  if (str === false) return ''
  let line = str
  if (line.slice(0, 4) === '@H1@') line = line.slice(4)
  if (line.slice(0, 4) === '@H2@') line = line.slice(4)
  if (line.slice(0, 4) === '@HR@') return '<hr />'
  if (line.length === 0) return '<br />'
  if (props.isBrf) {
    return decodeUEB(unicode2brf(line))
  }
  return toKatakana(tenji.fromTenji(line))
}
</script>

<style>
.page {
  position: relative;
  padding: 2rem 0 2rem 0;
  border-bottom: 2px solid #999;
}
.yomi {
  font-size: inherit;
  white-space: pre-wrap;
}
</style>
