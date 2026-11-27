---
title: "横スクロールが数px出るときの4つの原因。body に overflow-x: hidden を当てる前に測る"
emoji: "📏"
type: "tech"
topics: ["css", "html", "frontend", "responsive"]
published: true
---

## 結論

横スクロールが数px出るとき、`body { overflow-x: hidden }` は効きますが**原因を隠すだけ**です。1ページを調べたら、原因は4種類ありました。どれも個別に直せて、見た目も変わりませんでした。

症状が「どの画面幅でも同じ数pxだけ右にスクロールできる」なら、その値が手がかりになります。画面幅によらず一定なら、ほぼスクロールバーの幅です。

## 確認方法を先に決める

目視では数pxのはみ出しを見つけられないので、数値で判定します。

```js
// 横スクロールが出ているか
console.log({
  scrollWidth: document.documentElement.scrollWidth,
  clientWidth: document.documentElement.clientWidth,
});

// はみ出している要素を特定する
const limit = document.documentElement.clientWidth;
[...document.querySelectorAll('*')].forEach((el) => {
  const r = el.getBoundingClientRect();
  if (r.right > limit + 1 || r.left < -1) {
    console.log(Math.round(r.left), Math.round(r.right), el);
  }
});
```

`scrollWidth === clientWidth` になるまで直します。幅は1つでは足りません。私は 390 / 768 / 1024 / 1280 / 1440px の5つで毎回確認しています。

## 原因1：`100vw` はスクロールバーの幅を含む

一番多いのがこれです。`100vw` は**縦スクロールバーが無いものとして計算された幅**を返します。一方 `100%` は親要素の幅（スクロールバーを除いた表示領域）です。縦に長いページでは常にスクロールバーが出るので、`width: 100vw` の要素はその幅ぶんはみ出します。

Chromium（ウィンドウ幅1200px、クラシックなスクロールバー）で測ると、はっきり差が出ます。

| 指定 | 実際の幅 |
|---|---|
| `window.innerWidth` | 1200px |
| `documentElement.clientWidth` | 1185px |
| `width: 100vw` | 1200px |
| `width: 100%` | 1185px |

差の15pxがそのまま横スクロールになります。CSS仕様でも、ビューポート単位は「**スクロールバーが存在しないものとして**」算出すると定められています（[CSS Values and Units Level 4](https://www.w3.org/TR/css-values-4/)）。

```css
/* 修正前 */
.hero-bg { position: absolute; width: 100vw; }

/* 修正後 */
.hero-bg { position: absolute; width: 100%; }
```

全幅の親に対する絶対配置なら、`100%` に変えても**見た目は一切変わりません**。私が調べたページでは、このパターンが複数箇所ありました。

### `100dvw` や `100svw` に変えても直らない（手元で確認済み）

同じページで測った結果です。

| 指定 | 実際の幅 |
|---|---|
| `100vw` / `100dvw` / `100svw` / `100lvw` | いずれも 1200px |
| `100%` | 1185px |

`dvw` / `svw` / `lvw` はモバイルのアドレスバーなどで変わる「ビューポートの高さ・幅の取り方」の違いで、**スクロールバーの扱いは `vw` と同じ**でした。

一方、**ルート要素に `scrollbar-gutter: stable`（または `overflow-y: scroll`）を当てると、`100vw` は `clientWidth` と一致します**。仕様に「ルート要素の `overflow` や `scrollbar-gutter` が無条件にスクロールバーを出す／場所を確保する場合は、その軸のビューポート単位が縮む」とある通りの挙動です。

```css
/* これを当てると 100vw = clientWidth（= 1185px）になる */
:root { scrollbar-gutter: stable; }
```

ただし、ページ全体のレイアウト幅が常にスクロールバーぶん狭くなるので、既存のデザインに後から当てると影響範囲が広くなります。私は個別に `100%` へ直す方を選びました。

:::message
要確認：上の表は Chromium で測った結果です。Firefox / Safari での挙動は確かめていません。
:::

## 原因2：下限付きの `width: max(100vw, ...)`

セクション背景の装飾を「狭い画面でも途切れさせない」ために `max()` で下限を付けると、意図どおりはみ出します。

セクションの外にはみ出す要素（上に飛び出すタブなど）があると、セクション自体に `overflow: hidden` を当てられません。そこで**装飾だけをクリップ用の `div` で囲みます**。

```html
<section class="sec">
  <div class="sec__deco-clip"><div class="sec__deco"></div></div>
  <!-- はみ出させたい要素はクリップの外に置く -->
</section>
```

```css
.sec__deco-clip { position: absolute; inset: 0; overflow: hidden; }
.sec__deco { width: max(100%, 1440px); }
```

**装飾を消すのではなく、枠で囲んで隠す**のがポイントです。デザインを変えずに済みます。

## 原因3：`calc(50% - 固定px)` で置いた装飾

画面中央からコンテンツ幅の半分だけ離した位置に飾りを置く書き方です。そのレイアウト幅を前提にしているので、それ未満の画面では右にはみ出します。

これも親側でクリップします。

```css
.sec__decos { position: absolute; inset: 0; overflow: hidden; }
```

`inset: 0` で親の padding box に一致させているので、**装飾の位置計算の基準は変わりません**。広い画面での見た目はそのままで、狭い画面でだけ内側に収まります。

## 原因4：固定幅カードの合計

固定px幅のカードを複数列＋余白で並べていて、合計が画面幅を超えていました。可変にします。

```css
.card {
  width: 100%;
  max-width: 270px;  /* デザイン上の最大値 */
  aspect-ratio: 1;
}
```

広い画面では従来どおりの大きさのまま、狭い画面では縮みます。`aspect-ratio` を付けておくと、幅が縮んでも正方形のまま保てます。

## なぜ `overflow-x: hidden` を選ばなかったか

理由は3つです。

1. **原因が隠れる。** 次に誰かが同じ書き方を足しても気づけません
2. **`position: sticky` が効かなくなることがある。** 祖先に `overflow` が付くとスクロール用の祖先が変わるため、sticky の挙動が変わります。そのページでは sticky も fixed も未使用でしたが、後から足せない制約を残したくありませんでした
3. **4件とも個別に直せる見込みが立った。** 調査に1時間かからず、見た目も変わりませんでした

## 直したあとの確認

- 390 / 768 / 1024 / 1280 / 1440px すべてで `scrollWidth === clientWidth`
- 広い画面で装飾がクリップされて消えていないこと（原因2・3の直し方が正しいかはここで分かります）
- カードの実寸が意図どおり（広い画面で最大値、狭い画面では縮んでいる）

## ついでに見つかること

横スクロールを0にする過程で、ブレークポイントがスマホ用の1つしか無いページだと、**タブレット幅の装飾の重なり**が見つかりやすいです。PC用の `%` と `calc()` で配置した装飾が、中間の幅で本文に重なります。横スクロールの有無とは別の問題なので、直すならタブレット用の指定を足すことになります。

## まとめ

- `100vw` はスクロールバーの幅を含む。全幅の絶対配置なら `100%` にする
- `dvw` / `svw` に変えても直らない。ルート要素の `scrollbar-gutter: stable` なら直るが、影響範囲は広い
- 意図的にはみ出す装飾は、消さずに `inset: 0` + `overflow: hidden` の枠で囲む
- 固定px幅は `max-width` + `width: 100%` + `aspect-ratio` に置き換える
- `body { overflow-x: hidden }` を当てる前に、はみ出している要素を数値で特定する
