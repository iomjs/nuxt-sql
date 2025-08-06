<template lang="pug">
.sql-prettifier
  .row.m8.alert.alert-danger(v-if='err' v-text='err')
  .row.m8.float-right
    button.m2(@click.preventDefault.capture='format') Format
    button.m2(@click.preventDefault.capture='minify') Minify
    button.m2(@click.preventDefault.capture='filter') Filter
    button.m2(@click.preventDefault.capture='clear') Clear
    button.m2(@click.preventDefault.capture='decodeBase64Gzip') Decode Base64+Gzip
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
    tr
      th Line Break Every (chars):
      td: input.m2(type='number' v-model.number='lineBreakInterval' min='100' max='100000' placeholder='10000')
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
      lineBreakInterval: 10000,
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
        keywordPosition: 'standard',
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
      let { value, lineBreakInterval } = this
      value = value || ''
      value = value.replace(whitespaceRegEx, ' ')
      value = value.replace(leftparenthesesRegEx, '(')
      value = value.replace(rightparenthesesRegEx, ')')
      
      // Add double newlines at specified intervals, being careful not to break words or quoted strings
      if (lineBreakInterval > 0) {
        const result = []
        let currentPos = 0
        let lastBreakPos = 0
        let inQuotes = false
        
        for (let i = 0; i < value.length; i++) {
          const char = value[i]
          if (char === '\'' || char === '"') {
            inQuotes = !inQuotes
          }
          
          if (!inQuotes && i - lastBreakPos >= lineBreakInterval) {
            // Find the next whitespace to break on
            let breakPos = i
            while (breakPos < value.length && !/\s/.test(value[breakPos])) {
              breakPos++
            }
            
            if (breakPos < value.length) {
              result.push(value.substring(currentPos, breakPos + 1) + '\n\n')
              currentPos = breakPos + 1
              lastBreakPos = currentPos
              i = breakPos // Skip ahead to the break position
            } else {
              break // No more whitespace found, we're done
            }
          }
        }
        
        // Add the remaining part of the string
        if (currentPos < value.length) {
          result.push(value.substring(currentPos))
        }
        
        value = result.join('')
      } else {
        value = value + '\n'
      }
      
      this.value = value
    },
    filter () {
      this.pushHistory()
      let { value } = this
      value = value || ''
      value = value.replace(/\\'/g, '\'')
      value = value.replace(/"/g, '')
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
    async decodeBase64Gzip() {
      this.pushHistory()
      
      // Check for Compression Streams API support
      if (typeof DecompressionStream === 'undefined') {
        this.err = 'Your browser does not support the required Compression Streams API. Please use a modern browser like Chrome 80+, Edge 79+, or Opera 67+.'
        return
      }
      
      try {
        this.err = null
        
        // Check if the input is empty
        if (!this.value.trim()) {
          this.err = 'Please paste a base64-encoded gzipped string into the text area.'
          return
        }
        
        // Decode base64
        const base64Data = this.value.replace(/^data:.*?;base64,/, '')
        let binaryString
        try {
          binaryString = atob(base64Data)
        } catch (e) {
          this.err = 'Invalid base64 input. Please provide a valid base64-encoded string.'
          return
        }
        
        // Convert binary string to Uint8Array
        const len = binaryString.length
        const bytes = new Uint8Array(len)
        for (let i = 0; i < len; i++) {
          bytes[i] = binaryString.charCodeAt(i)
        }
        
        // Decompress using Compression Streams API
        const decompressionStream = new DecompressionStream('gzip')
        const decompressedStream = new Response(
          new Blob([bytes]).stream().pipeThrough(decompressionStream)
        )
        
        const decompressed = await decompressedStream.text()
        this.value = decompressed
      } catch (err) {
        console.error('Failed to decode base64 and gunzip:', err)
        this.err = 'Failed to decode. The input might not be a valid gzipped base64 string or there was an error during decompression.'
      }
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
