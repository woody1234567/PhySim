# Help Modal 實作說明

這份文件詳細說明了在 `app/pages/Chapter/Incline.vue` 中是如何實作物理模擬的說明彈窗（Help Modal）。

## 核心技術
- **Vue 3 `ref`**: 用於控制彈窗的顯示狀態。
- **HTML `<dialog>` 標籤**: 原生彈窗結構，搭配 CSS 類別（如 DaisyUI 的 `modal`）進行美化。
- **Icon 元件**: 使用 `@nuxt/icon` (或 `<Icon>` 元件) 顯示導覽圖示。
- **MathLatex 元件**: 用於渲染數學公式（LaTeX）。

## 實作步驟

### 1. 狀態定義
在 `<script setup>` 中定義了一個響應式變數 `showHelpModal`，預設為 `false`。
```typescript
const showHelpModal = ref(false);
```

### 2. 觸發按鈕
在模擬介面的右上角放置了一個固定位置的按鈕。當點擊時，將 `showHelpModal` 設為 `true`。
```html
<button
  @click="showHelpModal = true"
  class="absolute top-4 right-4 btn btn-circle btn-ghost ..."
>
  <Icon name="heroicons:question-mark-circle" class="text-3xl" />
</button>
```

### 3. Modal 結構
使用 DaisyUI 的 `modal` 樣式。關鍵在於透過 `:class="{ 'modal-open': showHelpModal }"` 來控制彈窗的開關。

#### 內部組成：
- **標題區**: 包含一個學術圖示 (`heroicons:academic-cap`)。
- **內容區**: 
  - 使用 `<section>` 區分「物理動力學」、「摩擦力與加速度」及「操作指南」。
  - 整合了 `<MathLatex />` 元件來展示物理公式，例如 `F_x = m \cdot g \cdot \sin(\theta)`。
- **操作區 (Action)**: 提供一個「Got it!」按鈕來關閉彈窗。
- **背景遮罩 (Backdrop)**: 允許使用者點擊背景區域來關閉彈窗。

```html
<dialog class="modal" :class="{ 'modal-open': showHelpModal }">
  <div class="modal-box max-w-2xl ...">
    <!-- 標題與內容 -->
    <div class="modal-action">
      <button class="btn btn-primary" @click="showHelpModal = false">Got it!</button>
    </div>
  </div>
  <form method="dialog" class="modal-backdrop">
    <button @click="showHelpModal = false">close</button>
  </form>
</dialog>
```

## 功能特點
1. **Latex 支援**: 能夠直接在說明中顯示專業的物理公式。
2. **毛玻璃效果**: 觸發按鈕採用了 `backdrop-blur`，使其在 3D 背景上清晰可見。
3. **響應式佈局**: 彈窗設定了 `max-w-2xl`，確保在寬螢幕下內容易於閱讀。
