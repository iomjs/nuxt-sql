<!-- Please remove this file from your project -->
<template lang="pug">
.sql-prettifier
  .row.m8.alert.alert-danger(v-if='err' v-text='err')
  .row.m8.float-right
    button.m2(@click.preventDefault.capture='format') Format
    button.m2(@click.preventDefault.capture='minify') Minify
    button.m2(@click.preventDefault.capture='clear') Clear
    button.m2(:disabled='history.length === 0' :class='{disabled: history.length == 0}' @click.preventDefault.capture='undo') Undo
  .row.m8
    textarea(
      v-model='value'
      :lang='textareaConfig.lang'
      :rows='textareaConfig.rows'
      :cols='textareaConfig.cols'
    )
  table
    tr
      th Syntax Formatter:
      td: select.m2(v-model='formatterConfig.lang')
        option(v-for="option in $options.formatterOptions.lang" :value='option.value' v-text="option.text")
    tr
      th Lines Between Queries:
      td: select.m2(v-model='formatterConfig.linesBetweenQueries')
        option(v-for="option in $options.formatterOptions.linesBetweenQueries" :value='option.value' v-text="option.text")
    tr
      th Indent:
      td: select.m2(v-model='formatterConfig.indent')
        option(v-for="option in $options.formatterOptions.indent" :value='option.value' v-text="option.text")
    tr
      th Keyword Position:
      td: select.m2(v-model='formatterConfig.keywordPosition')
        option(v-for="option in $options.formatterOptions.keywordPosition" :value='option.value' v-text="option.text")
  //- .row: .col: pre {{ JSON.stringify(formatterConfig, null, 2) }}
</template>

<script>
import * as prettierSql from 'prettier-sql'
// import pick from 'lodash/pick'

const whitespaceRegEx = /\s+/gim
const leftparenthesesRegEx = /\(\s+/gim
const rightparenthesesRegEx = /\s+\)/gim

export default {
  name: 'SqlPrettifier',
  data () {
    return {
      err: '',
      value: '',
      history: [],
      maxHistory: 10,
      textareaConfig: {
        lang: 'pgsql',
        rows: 20,
        cols: 90,
      },
      formatterConfig: {
        indent: '    ',
        lang: 'sql',
        linesBetweenQueries: 2,
        uppercase: true,
        keywordPosition: 'tenSpaceLeft',
        // newline: {
        //  "mode": "always" | "itemCount" | "lineWidth" | "hybrid" | "never",
        //  "itemCount"?: number // only used if newline.mode is itemCount or hybrid
        // },
        // breakBeforeBooleanOperator: bool,
        // aliasAs": "always" | "select" | "never",
        // tabulateAlias: boolean,
        commaPosition: 'after', // "before" | "after" | "tabular",
        parenOptions: {
          openParenNewline: false,
          closeParenNewline: false,
        },
        // lineWidth: number,
        denseOperators: true,
        semicolonNewline: true,
      },
    }
  },
  formatterOptions: Object.freeze({
    keywordPosition: ['standard', 'tenSpaceLeft', 'tenSpaceRight'].map(valueToValueTextObj),
    lang: ['db2', 'n1ql', 'pl/sql', 'sql'].map(valueToValueTextObj),
    indent: [
      { value: '', text: '0 spaces' },
      { value: '  ', text: '2 spaces' },
      { value: '    ', text: '4 spaces' },
      { value: '      ', text: '6 spaces' },
      { value: '        ', text: '8 spaces' },
      { value: '\t', text: 'tab' },
    ],
    linesBetweenQueries: [0, 1, 2, 3, 4].map(valueToValueTextObj),
  }),
  // aceOptions: Object.freeze({
  //   lang: ['pgsql', 'mysql', 'sql', 'sqlserver'],
  // }),
  methods: {
    pushHistory () {
      const { value, history, maxHistory } = this
      if (history[0] !== value) {
        if (history.unshift(value) > maxHistory) {
          history.length = maxHistory
        }
        this.history = history
      }
    },
    minify () {
      this.pushHistory()
      let { value } = this
      value = value || ''
      value = value.replace(whitespaceRegEx, ' ')
      value = value.replace(leftparenthesesRegEx, '(')
      value = value.replace(rightparenthesesRegEx, ')')
      value = value + '\n'
      this.value = value
    },
    clear () {
      this.pushHistory()
      this.value = ''
    },
    format () {
      this.pushHistory()
      try {
        this.err = null
        const options = {
          ...this.formatterConfig,
        }
        let { value } = this
        value = prettierSql.format(value, options) + '\n'
        this.value = value
      } catch (err) {
        // eslint-disable-next-line
        console.error(err)
        // eslint-disable-next-line
        this.err = new String(err)
      }
    },
    undo () {
      const { history } = this
      this.value = history.shift()
      this.history = history
    },
    onEditorChange (newValue) {
      // eslint-disable-next-line
      console.log('onEditorChange', { newValue })
      this.value = newValue.detail
    },
  },
}

function valueToValueTextObj (s) {
  return { value: s, text: s }
}
</script>
