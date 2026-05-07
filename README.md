# HTML-in-Canvas 3D Cube & Scrolling

HTML-in-Canvas (`texElementImage2D`) を使用して、3D空間上の面にHTMLコンテンツを描画するプロジェクトです。
公式ドキュメントを正しく理解せずに、試行錯誤中に発見した、**「ネイティブスクロールが動作しない（透明になる）」** という重要な制限事項とその回避策についてまとめています。

## ⚠️ 重要な制限事項：スクロールの罠

現在の HTML-in-Canvas 試験実装において、以下のCSS設定は**正常に描画されません。**

```css
/* ❌ 動作しない（中身が透明になる、または更新されない） */
.element {
    overflow-y: scroll;
    overflow-x: auto;
}
```

### なぜスクロールができないのか？

#### 1. 公式仕様書での言及
GitHubのプロポーザル（[vmpstr/web-proposals](https://github.com/vmpstr/web-proposals/blob/main/canvas-draw-element/README.md#scrolling-and-animations)）および [公式ドキュメント](https://html-in-canvas.dev/docs/browser-support/#unsupported-features) には以下の制約が明記されています。

> *"Scrolling is not supported in this version of the API. Elements will be captured at the scroll position they had when they were last painted on the main thread."*
> （このバージョンのAPIではスクロールはサポートされていません。要素は、メインスレッドで最後にペイントされた時のスクロール位置でキャプチャされます。）

#### 2. 技術的な理由
*   **スレッドの分離:** Chromeなどのモダンブラウザは、パフォーマンス向上のためスクロール要素を **「コンポジタスレッド（GPUレイヤー）」** という別枠に分離して処理します。
*   **キャプチャの仕組み:** 現在の HTML-in-Canvas は「メインスレッド」での描画（Paint）の瞬間をスナップショットとして切り取る仕組みです。
*   **結果:** 別レイヤーに分離された「スクロール中身」はメインスレッドのキャプチャ対象から外れてしまい、結果として背景や文字が消えて透明（WebGLの背景が透けた状態）になってしまいます。

---

## 💡 解決策：擬似スクロールの実装

ネイティブの `overflow: scroll` が使えないため、JavaScript で要素の座標を直接動かす **「擬似スクロール」** で代用します。

### 1. HTML/CSS の構成
`overflow: hidden` にした親要素の中で、中身（`#scroll-content`）を `position: absolute` で配置します。

```html
<div class="parent" style="overflow: hidden; position: relative;">
    <div id="scroll-content" style="position: absolute; top: 0px;">
        <!-- 長文コンテンツ -->
    </div>
</div>
```

### 2. JavaScript での制御
マウスホイールイベントを検知し、`scrollTop` の代わりに `style.top` を書き換えます。この方法であればメインスレッド上で描画が行われるため、100%確実に 3D テクスチャとしてキャプチャされます。

```javascript
// スクロール処理の例
const deltaY = event.deltaY;
let currentTop = parseFloat(scrollContent.style.top || '0');
let newTop = currentTop - deltaY;

// 限界値の制御をして適用
scrollContent.style.top = newTop + "px";

// 強制的に再描画を要求
canvas.requestPaint();
```

---

## 🔗 公式リファレンス
*   [vmpstr/web-proposals - Scrolling and Animations](https://github.com/vmpstr/web-proposals/blob/main/canvas-draw-element/README.md#scrolling-and-animations)
*   [html-in-canvas.dev - Unsupported features](https://html-in-canvas.dev/docs/browser-support/#unsupported-features)

## 免責事項
このプロジェクトは **Chrome Canary** または **Brave Stable (Chromium 147+)** で、`chrome://flags/#canvas-draw-element` を有効にした環境で動作します。
