<script setup>
import { ref, computed } from 'vue'
import { evaluate } from 'mathjs'

const expression = ref('')
const result = ref('')
const justEvaluated = ref(false)

const buttons = [
  ['AC', '+/-', '%', '/'],
  ['7', '8', '9', '*'],
  ['4', '5', '6', '-'],
  ['1', '2', '3', '+'],
  ['0', '.', '=']
]

function handleClick(value) {
  const operators = ['+', '-', '*', '/']
  const lastChar = expression.value.slice(-1)

  // AC
  if (value === 'AC') {
    expression.value = ''
    result.value = ''
    justEvaluated.value = false
    return
  }

  // =
  if (value === '=') {
    try {
      result.value = evaluate(expression.value).toString()
      justEvaluated.value = true
    } catch {
      result.value = 'Error'
    }
    return
  }

  // After = clicked
  if (justEvaluated.value) {
    if (/[0-9.]/.test(value)) {
      expression.value = value
    } else if (operators.includes(value)) {
      expression.value = result.value + value
    } else if (value === '+/-') {
      expression.value = '-(' + result.value + ')'
    }
    result.value = ''
    justEvaluated.value = false
    return
  }

  // Prevent multiple leading zeros
  if (value === '0') {
    const match = expression.value.match(/(?:^|[+\-*/(])0$/)
    if (match) return
  }

  // Decimal
  if (value === '.') {
    const lastNumberMatch = expression.value.match(/([^\+\-\*\/\(\)]+)$/)
    const lastNumber = lastNumberMatch ? lastNumberMatch[1] : ''
    if (lastNumber.includes('.')) return
    if (expression.value === '' || /[+\-*/(]$/.test(expression.value)) {
      expression.value += '0'
    }
  }

  // +/-
  if (value === '+/-') {
    if (!expression.value) {
      expression.value = '-'
      return
    }

    const match = expression.value.match(/(?:\(-)?(\d+(\.\d+)?)\)?$/)
    if (match) {
      const fullMatch = match[0]
      const numberValue = match[1]
      const start = match.index
      const before = expression.value.slice(0, start)
      const after = expression.value.slice(start + fullMatch.length)

      if (numberValue === '0') return

      if (fullMatch.startsWith('(-')) {
        expression.value = before + numberValue + after
      } else {
        expression.value = before + '(-' + fullMatch + ')' + after
      }
    }
    return
  }

  // Percentage
  if (value === '%') {
    if (!expression.value) {
      expression.value = '(0/100)'
      return
    }

    if (/[+\-*/]$/.test(lastChar)) {
      expression.value += '(0/100)'
      return
    }

    const match = expression.value.match(/(\(-?.+?\)|\d+(\.\d+)?)$/)
    if (match) {
      const fullMatch = match[0]
      const start = match.index
      const before = expression.value.slice(0, start)
      const after = expression.value.slice(start + fullMatch.length)
      expression.value = before + '(' + fullMatch + '/100)' + after
      return
    }

    expression.value = '(' + expression.value + '/100)'
    return
  }

  // Prevent operator beside another operator
  if (operators.includes(value)) {
    if (operators.includes(lastChar)) {
      expression.value = expression.value.slice(0, -1) + value
      return
    }

    if (expression.value === '') {
      expression.value = '0' + value
      return
    }
  }

  // Default: Append value
  expression.value += value
}

function formatPercentages(expr) {
  let formatted = expr

  while (/\(([^()]+?)\/100\)/.test(formatted)) {
    formatted = formatted.replace(/\(([^()]+?)\/100\)/g, (_, inner) => `${formatPercentages(inner)}%`)
  }

  return formatted
    .replace(/\//g, '÷')
    .replace(/\*/g, '×')
}

const displayExpression = computed(() =>
  justEvaluated.value ? formatPercentages(expression.value) : ''
)

const displayResult = computed(() =>
  justEvaluated.value
    ? result.value || '0'
    : formatPercentages(expression.value || '0')
)

function getButtonSymbol(btn) {
  const symbolMap = {
    '/': '÷',
    '*': '×',
    '+/-': '±'
  }
  return symbolMap[btn] || btn
}

function getButtonClass(btn) {
  return [
    btn === '0' ? 'double' : '',
    ['AC', '+/-', '%'].includes(btn)
      ? 'function'
      : ['/', '*', '-', '+', '='].includes(btn)
        ? 'operator'
        : ''
  ]
}
</script>

<template>
  <div>
    <div class="display">
      <div class="expression">{{ displayExpression }}</div>
      <div class="result">{{ displayResult }}</div>
    </div>

    <div class="calculator">
      <template v-for="(row, rowIndex) in buttons" :key="rowIndex">
        <div
          v-for="(btn, btnIndex) in row"
          :key="btnIndex"
          class="btn"
          :class="getButtonClass(btn)"
          @click="handleClick(btn)"
        >
          {{ getButtonSymbol(btn) }}
        </div>
      </template>
    </div>
  </div>
</template>

<style scoped>
.calculator {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
  width: 100%;
  max-width: 360px;
  margin: auto;
  padding: 20px;
}

.btn {
  aspect-ratio: 1 / 1;
  width: 100%;
  border-radius: 50%;
  font-size: 28px;
  font-weight: 400;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  cursor: pointer;
  user-select: none;
  background-color: #505050;
  transition: filter 0.15s;
  text-align: center;
}

.btn.double {
  grid-column: span 2;
  aspect-ratio: auto;
  border-radius: 40px;
  justify-content: center;
}

.btn:active {
  filter: brightness(1.25);
}

.btn.function {
  background-color: #d4d4d2;
  color: black;
}

.btn.operator {
  background-color: #ff9500;
  color: white;
}

.display {
  background-color: #1c1c1c;
  border-radius: 16px;
  padding: 20px;
  margin: 20px auto 0;
  max-width: 360px;
  min-height: 100px;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  align-items: flex-end;
  box-sizing: border-box;
  color: white;
  font-weight: 200;
}

.result {
  font-size: 48px;
  line-height: 1.2;
}

.expression {
  font-size: 24px;
  opacity: 0.5;
  margin-bottom: 4px;
}

@media (max-width: 400px) {
  .btn {
    font-size: 22px;
  }

  .display {
    font-size: 20px;
    padding: 16px;
  }

  .result {
    font-size: 36px;
  }

  .expression {
    font-size: 18px;
  }
}
</style>
