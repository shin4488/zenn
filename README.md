# zenn

[Zenn](https://zenn.dev/shin4488) の記事をGitHub連携で管理するリポジトリ。mainブランチへのpushで自動デプロイされる。

## 構成

```
articles/   記事のMarkdown(ファイル名 = スラッグ)
images/     記事に貼る画像(記事スラッグごとにサブディレクトリ)
assets/svg/ 図の元データSVG(デプロイ対象外)
books/      Zennの本(未使用)
```

## セットアップ

```sh
npm install
```

## よく使うコマンド

```sh
npx zenn new:article   # 記事の雛形を作成
npx zenn preview       # http://localhost:8000 でプレビュー
```

## 記事を書くときのルール

- フロントマターは `title` / `emoji` / `type: tech` / `topics` / `published` を指定する
- スラッグ(ファイル名)は半角英数字・ハイフン・アンダースコアで12〜50字
- 画像は `images/<記事スラッグ>/` に置き、本文からは `/images/<記事スラッグ>/xxx.png` の**絶対パス**で参照する(相対パス不可)
- `images/` に置けるのは png / jpg / jpeg / gif / webp のみ(SVG不可)、1ファイル3MB以内
- 下書きは `published: false` のままpushしてよい(Zenn上でも下書き扱いになる)

## 図の作り方

図はSVGで `assets/svg/<記事スラッグ>/` に作り、ChromeヘッドレスでPNGに変換して `images/` に置く。

```sh
CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
"$CHROME" --headless=new --disable-gpu \
  --screenshot=images/<記事スラッグ>/figN.png --window-size=<幅>,<高さ> \
  "file://$PWD/assets/svg/<記事スラッグ>/figN.svg"
```

window-size は各SVGの width / height 属性に合わせる。

## 公開フロー

1. 記事を書いて `npx zenn preview` で確認
2. `published: true` にしてmainへpush
3. 自動デプロイで公開される

## 注意

- 記事の削除は、リポジトリからのファイル削除とZennダッシュボードからの削除の両方が必要(ファイルが残っていると次のデプロイで復活する)
- Zennに連携できるリポジトリはアカウントごとに最大2つ
- このリポジトリに入れていない既存記事(Webエディタで書いたもの)は、連携しても影響を受けない
