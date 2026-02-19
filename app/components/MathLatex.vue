<template>
  <span
    v-if="inline"
    class="math-latex-inline"
    v-html="renderedFormula"
  ></span>
  <div
    v-else
    class="math-latex-block my-4 flex justify-center"
    v-html="renderedFormula"
  ></div>
</template>

<script setup lang="ts">
import { computed } from 'vue';
import katex from 'katex';

const props = defineProps({
  formula: {
    type: String,
    required: true,
  },
  inline: {
    type: Boolean,
    default: true,
  },
  throwOnError: {
    type: Boolean,
    default: false,
  },
});

const renderedFormula = computed(() => {
  try {
    return katex.renderToString(props.formula, {
      displayMode: !props.inline,
      throwOnError: props.throwOnError,
    });
  } catch (e) {
    console.error('KaTeX error:', e);
    return props.formula;
  }
});
</script>

<style>
/* KaTeX styles are imported globally via nuxt.config.ts */
.math-latex-inline {
  display: inline-block;
  vertical-align: middle;
}
.math-latex-block {
  width: 100%;
  overflow-x: auto;
  overflow-y: hidden;
}
</style>
