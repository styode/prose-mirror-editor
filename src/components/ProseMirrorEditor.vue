<template>
  <!-- 保持模板部分不变 -->
  <div class="editor-container">
    <div class="rules-panel">
      <h3>输入规则说明：</h3>
      <div class="rules-content">
        <div class="rule-item">
          <span class="rule-number">1.</span>
          <span class="rule-text">光标位于行首时键入【，补全】,【】内文字为红色；</span>
        </div>
        <div class="rule-item">
          <span class="rule-number">2.</span>
          <span class="rule-text">光标位于行首时，输入4个汉字以内，键入:时，包含：之前的中文会认为是人名，人名颜色为红色，光标停留在:后，继续输入，（）内文字为绿色，其余文字为黑色。</span>
        </div>
      </div>
    </div>
    <div class="menu-bar">
      <button @click="saveContent" class="save-button">保存内容</button>
    </div>
    <div ref="editorRef" class="editor-content"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import { EditorState } from 'prosemirror-state'
import { EditorView } from 'prosemirror-view'
import { Schema } from 'prosemirror-model'
import { inputRules, InputRule } from 'prosemirror-inputrules'
import { Plugin } from 'prosemirror-state'
import 'prosemirror-view/style/prosemirror.css'

// ==================== 编辑器基础配置 ====================
// 输入法状态跟踪
let isComposing = false
let lastCompositionEnd = 0

// 文档Schema定义
const schema = new Schema({
  marks: {
    red: {
      parseDOM: [{ tag: 'span.red' }],
      toDOM: () => ['span', { class: 'red' }, 0]
    },
    green: {
      parseDOM: [{ tag: 'span.green' }],
      toDOM: () => ['span', { class: 'green' }, 0]
    }
  },
  nodes: {
    doc: { content: 'block+' },
    paragraph: {
      group: 'block',
      content: 'inline*',
      toDOM: () => ['p', 0]
    },
    text: { group: 'inline' }
  }
})

// ==================== 工具函数 ====================
// 检查是否在行首
const isAtLineStart = ($pos) => $pos.parentOffset === 0

// ==================== 编辑器插件 ====================
// 组合输入处理插件
const compositionPlugin = new Plugin({
  view(view) {
    const handleStart = () => {
      isComposing = true
      lastCompositionEnd = 0
    }

    const handleEnd = (e) => {
      isComposing = false
      lastCompositionEnd = e.timeStamp
    }

    view.dom.addEventListener('compositionstart', handleStart)
    view.dom.addEventListener('compositionend', handleEnd)

    return {
      destroy() {
        view.dom.removeEventListener('compositionstart', handleStart)
        view.dom.removeEventListener('compositionend', handleEnd)
      }
    }
  }
})

// 光标跟踪插件
const cursorPlugin = new Plugin({
  view(view) {
    let currentMarks = []

    const updateMarks = (pos) => {
      const { doc, selection } = view.state
      const node = doc.nodeAt(pos)
      const marks = new Set()

      // 检测当前位置的标记
      if (node) {
        node.marks.forEach(mark => {
          if (mark.type === schema.marks.red) marks.add('red')
          if (mark.type === schema.marks.green) marks.add('green')
        })
      }

      // 检查是否在括号内
      if (selection.empty) {
        const $pos = selection.$head
        const textBefore = $pos.parent.textContent.slice(0, $pos.parentOffset)
        const textAfter = $pos.parent.textContent.slice($pos.parentOffset)

        // 只检查中文括号
        const chineseOpenPos = textBefore.lastIndexOf('（')
        const chineseClosePos = textAfter.indexOf('）')

        // 如果在中文括号内，添加绿色标记
        if (chineseOpenPos !== -1 && chineseClosePos !== -1) {
          marks.add('green')
        }
      }

      // 更新标记状态
      if (Array.from(marks).join() !== currentMarks.join()) {
        currentMarks = Array.from(marks)
        const storedMarks = currentMarks.length ?
          currentMarks.map(m => schema.marks[m].create()) :
          null

        view.dispatch(view.state.tr.setStoredMarks(storedMarks))
      }
    }

    return {
      update(view) {
        if (view.state.selection.empty) {
          updateMarks(view.state.selection.$head.pos)
        }
      }
    }
  }
})

// ==================== 输入规则 ====================
const buildInputRules = (schema) => {
  const createRules = () => {
    // 方括号自动补全（行首限定）
    const bracketRule = new InputRule(/^(【|\[)$/, (state, match, start, end) => {
      if (isComposing || Date.now() - lastCompositionEnd < 100) return null
      if (!isAtLineStart(state.doc.resolve(start))) return null

      const tr = state.tr
      const isChinese = match[0] === '【'
      const closeChar = isChinese ? '】' : ']'

      tr.delete(start, end)
      tr.insertText(match[0] + closeChar)
      tr.setSelection(state.selection.constructor.near(tr.doc.resolve(start + 1)))
      tr.setStoredMarks([schema.marks.red.create()])

      return tr
    })

    // 方括号内文本着色
    const bracketContentRule = new InputRule(/^(.+?)(】|])$/, (state, match, start, end) => {
      if (isComposing) return null

      const tr = state.tr
      const text = match[1]
      const closeChar = match[2]
      const $pos = state.doc.resolve(start)

      if (!$pos.parent.textContent.slice(0, $pos.parentOffset).endsWith('【')) return null

      tr.delete(start, end)
      tr.insertText(text + closeChar)
      tr.addMark(start, start + text.length, schema.marks.red.create())
      tr.setStoredMarks([])

      return tr
    })

    // 人名规则处理函数
    const createNameRule = (isChineseColon) => new InputRule(
      isChineseColon
        ? /^([\u4e00-\u9fa5]{1,4})：$/
        : /^([\u4e00-\u9fa5]{1,4}):$/,
      (state, match, start, end) => {
        if (isComposing) return null
        if (!isAtLineStart(state.doc.resolve(start))) return null

        const tr = state.tr
        const name = match[1]
        const colon = isChineseColon ? '：' : ':'

        tr.delete(start, end)
        tr.insertText(name + colon)
        tr.addMark(start, start + name.length, schema.marks.red.create())
        tr.setSelection(state.selection.constructor.near(tr.doc.resolve(start + name.length + 1)))
        tr.setStoredMarks([])

        return tr
      }
    )

    // 自动补全括号
    const parenthesisRule = new InputRule(/（/, (state, match, start, end) => {
      if (isComposing) return null

      const tr = state.tr
      const $pos = state.doc.resolve(start)
      const textBefore = $pos.parent.textContent.slice(0, $pos.parentOffset)
      const textAfter = $pos.parent.textContent.slice($pos.parentOffset)

      // 检查是否已经在括号内
      const chineseOpenPos = textBefore.lastIndexOf('（')
      const chineseClosePos = textAfter.indexOf('）')

      // 如果已经在括号内，不重复添加
      if (chineseOpenPos !== -1 && chineseClosePos !== -1) {
        return null
      }

      // 删除输入的左括号
      tr.delete(end - 1, end)
      // 插入完整的一对括号
      tr.insertText('（）', end - 1)
      // 将光标定位到括号中间
      tr.setSelection(state.selection.constructor.near(tr.doc.resolve(end)))
      // 添加绿色标记，用于括号内输入
      tr.setStoredMarks([schema.marks.green.create()])

      return tr
    })

    // 括号内文本着色
    const parenthesisContentRule = new InputRule(/\）/, (state, match, start, end) => {
      if (isComposing) return null

      const tr = state.tr
      const $pos = state.doc.resolve(end - 1)
      const parentStart = $pos.start()
      const textBefore = $pos.parent.textContent.slice(0, $pos.parentOffset)

      // 查找对应的左括号
      const openPos = textBefore.lastIndexOf('（')
      if (openPos === -1) return null

      // 计算括号内文本范围
      const from = parentStart + openPos + 1
      const to = end - 1

      // 检查括号内是否有内容
      if (from >= to) return null

      // 应用绿色标记到括号内文本
      tr.addMark(from, to, schema.marks.green.create())
      // 清除存储标记
      tr.setStoredMarks([])

      return tr
    })

    // 括号内实时着色
    const parenthesisLiveRule = new InputRule(/(.*)/, (state, match, start, end) => {
      if (isComposing || match[0].length === 0) return null

      const tr = state.tr
      const $pos = state.doc.resolve(start)
      const parentStart = $pos.start()
      const textBefore = $pos.parent.textContent.slice(0, $pos.parentOffset)
      const textAfter = $pos.parent.textContent.slice($pos.parentOffset)

      // 检查是否在中文括号内
      const chineseOpenPos = textBefore.lastIndexOf('（')
      const chineseClosePos = textAfter.indexOf('）')

      // 只有在中文括号内才应用样式
      if (chineseOpenPos !== -1 && chineseClosePos !== -1) {
        const bracketStart = parentStart + chineseOpenPos + 1
        if (end > bracketStart) {
          // 只添加样式，不修改文本
          tr.addMark(bracketStart, end, schema.marks.green.create())
          // 设置存储标记，后续输入仍为绿色
          tr.setStoredMarks([schema.marks.green.create()])
          return tr
        }
      }

      return null
    })

    return [
      bracketRule,
      bracketContentRule,
      createNameRule(false), // 英文冒号
      createNameRule(true),  // 中文冒号
      parenthesisRule,
      parenthesisContentRule,
      parenthesisLiveRule
    ]
  }

  return inputRules({ rules: createRules() })
}

// ==================== 编辑器初始化 ====================
const editorRef = ref(null)
const editorView = ref(null)

const saveContent = () => {
  if (!editorView.value) return
  const content = editorView.value.state.doc.toJSON()
  console.log('保存内容:', JSON.stringify(content, null, 2))
  alert('内容已保存到控制台')
}

onMounted(() => {
  const state = EditorState.create({
    schema,
    plugins: [
      buildInputRules(schema),
      cursorPlugin,
      compositionPlugin
    ]
  })

  editorView.value = new EditorView(editorRef.value, {
    state,
    dispatchTransaction(transaction) {
      this.updateState(this.state.apply(transaction))
    }
  })
})

onBeforeUnmount(() => {
  if (editorView.value) editorView.value.destroy()
})
</script>

<style scoped>
.editor-container {
  border: 1px solid #e8e8e8;
  border-radius: 8px;
  max-width: 800px;
  margin: 20px auto;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  background: #ffffff;
  overflow: hidden;
}

.rules-panel {
  padding: 20px;
  background: #fafafa;
  border-bottom: 1px solid #f0f0f0;
}

.rules-panel h3 {
  margin: 0 0 16px 0;
  color: #262626;
  font-size: 18px;
  font-weight: 600;
}

.rules-content {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.rule-item {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  line-height: 1.6;
  padding: 8px 12px;
  background: #ffffff;
  border-radius: 6px;
  border: 1px solid #f0f0f0;
}

.rule-number {
  color: #1890ff;
  font-weight: 600;
  min-width: 24px;
  font-size: 16px;
}

.rule-text {
  color: #595959;
  flex: 1;
  font-size: 14px;
}

.menu-bar {
  padding: 12px 20px;
  background: #ffffff;
  border-bottom: 1px solid #f0f0f0;
  display: flex;
  justify-content: flex-end;
}

.save-button {
  background: #1890ff;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 8px;
}

.save-button:hover {
  background: #40a9ff;
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(24, 144, 255, 0.35);
}

.save-button:active {
  transform: translateY(0);
  box-shadow: none;
}

.editor-content {
  padding: 20px;
  min-height: 300px;
}

/* 标记样式 */
:deep(.red) {
  color: #ff4d4f !important;
  font-weight: 500;
  background: rgba(255, 77, 79, 0.1);
  padding: 0 2px;
  border-radius: 2px;
}

:deep(.green) {
  color: #52c41a !important;
  background: rgba(82, 196, 26, 0.1);
  padding: 0 2px;
  border-radius: 2px;
}

:deep(.ProseMirror) {
  min-height: 200px;
  padding: 12px;
  outline: none;
  line-height: 1.8;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell;
  font-size: 15px;
  color: #262626;
}

:deep(.ProseMirror p) {
  margin: 0 0 1em;
}

:deep(.ProseMirror p:last-child) {
  margin-bottom: 0;
}

:deep(.ProseMirror:focus) {
  outline: none;
}

/* 滚动条样式 */
:deep(.ProseMirror::-webkit-scrollbar) {
  width: 8px;
  height: 8px;
}

:deep(.ProseMirror::-webkit-scrollbar-track) {
  background: #f5f5f5;
  border-radius: 4px;
}

:deep(.ProseMirror::-webkit-scrollbar-thumb) {
  background: #d9d9d9;
  border-radius: 4px;
}

:deep(.ProseMirror::-webkit-scrollbar-thumb:hover) {
  background: #bfbfbf;
}
</style>
