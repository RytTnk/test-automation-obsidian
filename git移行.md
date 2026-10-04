 - 戦略作成
 - 移行手順作成(コマンド)

## 戦略1
- v2で生成したテストコードを，オリジンに追加，gt add, commint, push

## 戦略2
- 改善版claude案を採用
- 念の為v2のコピーバックアップは取っておく


## 移行手順
### 戦略1
v2で生成したテストコードを，オリジンに手動追加
オリジンのブランチで，
```
git add -A   # ステージング
git commit -m
```

### 戦略2

```sh
# ステップ1：安全に開始 カレントディレクトリは_v2
git init
git remote add origin v1
git fetch origin
# ここまでで確認

# ステップ2：v2のファイルをバージョン記録
git add -A   # v2のファイルをすべてステージング
git commit -m "v2 初期状態のスナップショット"

# ステップ3：v1をマージ
git merge origin/feature_1 --allow-unrelated-histories
# 競合（conflict）が出たら、後者のバージョンを選択，全て後者v2を選択

# ステップ4：ブランチ作成
git checkout -b feature_1_v2

# ステップ5：整理
git commit -m "v2 と feature_1 の統合完了"
```

この戦略2をgeminiで実験
- 事前準備
	- repository
	- v1, 
	- v2