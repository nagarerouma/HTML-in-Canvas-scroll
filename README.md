# HTML-in-Canvas 3D Cube & Scrolling

HTML-in-Canvas (`texElementImage2D`) を使用して、3D空間上の面にHTMLコンテンツを描画するプロジェクトです。
試行錯誤中に発見した、**「ネイティブスクロールが動作しない（透明になる）」** という重要な制限事項とその回避策についてまとめています。公式ドキュメントを読んでいれば迷わなかったことです。

<video src="https://github.com/nagarerouma/HTML-in-Canvas-scroll/video.mp4" controls="controls" muted="muted" style="max-width: 100%;"></video>

## 📁 ファイル構成

- `index.html` - メインサンプル。`<canvas layoutsubtree>` の直接子として 6 つの iframe を配置し、各 iframe の描画内容を `texElementImage2D` で WebGL テクスチャに転送して 3D キューブの各面に貼り付けます。ドラッグで回転、ホイールでズーム、クリック位置を 3D 空間上の UV 座標に変換して iframe 内 DOM へイベントを転送する実装です。
- `index1.html` - 立方体の 1 面目用。通常の HTML `input` を 3D 面に表示し、フォーカスと文字入力が可能なことを示します。
- `index2.html` - 立方体の 2 面目用。`transform` ではなく `font-size` / `color` / `text-shadow` のような描画プロパティをアニメーションさせ、HTML-in-Canvas 上でも更新が反映される例です。
- `index3.html` - 立方体の 3 面目用。マウスホバーに相当する UI 変化を実装し、色や背景の変化で paint が発火することを示します。
- `index4.html` - ❌ 立方体の 4 面目用。最もシンプルな静的 HTML テクスチャ例で、ネイティブスクロールが使えない制約と擬似スクロールの考え方を理解するための内容を含みます。
- `index5.html` - 立方体の 5 面目用。通常の HTML ボタンを置き、3D 面上でクリックを受け取って DOM に転送できることを確認するためのテスト用ページです。
- `index6.html` - 立方体の 6 面目用。クロスオリジン制約について学ぶためのページで、外部サイトを直接 `texElementImage2D` でテクスチャ化できない仕様を説明する UI を含みます。

## ⚠️ 重要な制限事項：スクロール

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
