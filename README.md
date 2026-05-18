# 単価比較ツール

商品の単価をリアルタイムに比較するシングルページアプリ。  
Svelte 5 + Vite 製。GitHub Pages で動作する。

https://h-kono-it.github.io/tanka-hikaku/

- 価格・内容量を入力するだけで単価を自動計算
- 最安値を即座にハイライト
- 値引き（%）の入力に対応
- 商品は追加・削除可能（デフォルト3件）

## ローカル起動

```bash
npm install
npm run dev
```

http://localhost:5173 で起動する。

## デプロイ（GitHub Pages）

1. GitHub にリポジトリを作成して push する
2. リポジトリの **Settings → Pages → Source** を `GitHub Actions` に設定する
3. `main` ブランチへ push するたびに自動デプロイされる
