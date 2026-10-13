# ロード画面 作成手順

Title → InGame への遷移時、モデル等の読み込みが重いため、間に `Lvl_Loading` を挟んで
重いアセットを **非同期で先読み** してから `Lvl_InGame` を開く。

- `OpenLevel` 直呼びはゲームスレッドが止まり画面が固まる → Async Load ならロード画面が動き続ける
- 読み込んだアセットは `BP_GameInstance` に保持（レベル遷移で解放されないように）

## 全体の流れ

```
WBP_Title ──OpenLevel──▶ Lvl_Loading ──(先読み完了)──OpenLevel──▶ Lvl_InGame
                          ├ WBP_Loading 表示
                          └ アセットを1個ずつ Async Load → GameInstance に保持
```

---

## 1. BP_GameInstance（`Content/Project/System`）

変数を2つ追加。

| 変数名 | 型 |
|---|---|
| `PreloadedAssets` | Object（Object Reference）の **Array** |
| `PreloadedClasses` | Object（**Class Reference**）の **Array** |

## 2. WBP_Loading（新規 Widget Blueprint）

1. `Canvas Panel` の下に `Image`（背景・アンカー全画面）と `Text`（"Now Loading..."）を配置
2. `Progress Bar` を置き、名前を `PB_Loading`、**Is Variable にチェック**

## 3. Lvl_Loading（新規 Empty Level → `Content/Project/Maps`）

1. **World Settings → GameMode Override** を `GameModeBase` にする
   （デフォルトの `BP_FirstPersonGameMode` だとキャラがスポーンしてしまう）
2. レベルBPに変数を作成

| 変数名 | 型 | デフォルト値 |
|---|---|---|
| `AssetsToLoad` | Object（**Soft Object Reference**）Array | モデル: MG, MG2, enemyBuggy, enemyBuggyExp, Drone, DroneExp |
| `ClassesToLoad` | Object（**Soft Class Reference**）Array | BP_FirstPersonCharacter, BP_WaveManager, BP_BattleCardManager, BP_CardFactory, 敵BP |
| `LoadingWidget` | WBP_Loading（Object Reference） | — |
| `LoadIndex` / `LoadedCount` / `TotalCount` | Integer | 0 |

> 先読み候補は `Lvl_InGame` が直接参照しているもの。スポーン時に読まれる Niagara / SE / マテリアルも入れると効果大。

## 4. Lvl_Loading レベルBPのノード

> ⚠ `ForEach` の中で `Async Load Asset` を呼ぶのは **NG**。
> このノードは前の読み込みが終わるまで次の呼び出しを無視するので、1個しか読み込まれない。
> → カスタムイベントで1個ずつ順番に読む。

### BeginPlay

```
Event BeginPlay
 → Create Widget (Class: WBP_Loading, Owning Player: Get Player Controller 0)
 → SET LoadingWidget
 → Add to Viewport
 → SET TotalCount = AssetsToLoad.Length + ClassesToLoad.Length
 → LoadNextAsset（カスタムイベント呼び出し）
```

### カスタムイベント `LoadNextAsset`

```
LoadNextAsset
 → Branch (LoadIndex < AssetsToLoad.Length)
   True  → Async Load Asset (Asset: AssetsToLoad GET LoadIndex)
            Completed → Cast to BP_GameInstance (Get Game Instance)
                      → PreloadedAssets ADD (Object)
                      → LoadIndex++ → UpdateProgress → LoadNextAsset
   False → SET LoadIndex = 0 → LoadNextClass
```

### カスタムイベント `LoadNextClass`

```
LoadNextClass
 → Branch (LoadIndex < ClassesToLoad.Length)
   True  → Async Load Class Asset (Asset Class: ClassesToLoad GET LoadIndex)
            Completed → Cast to BP_GameInstance
                      → PreloadedClasses ADD (Class)
                      → LoadIndex++ → UpdateProgress → LoadNextClass
   False → Delay 0.2 → Open Level (by Name) "Lvl_InGame"
```

### カスタムイベント `UpdateProgress`

```
UpdateProgress
 → LoadedCount++
 → LoadingWidget → PB_Loading → Set Percent
     (In Percent: ToFloat(LoadedCount) / ToFloat(TotalCount))
```

> 割り算の前に Float 変換すること。Integer 同士だと途中は 0 のままでバーが進まない。

## 5. WBP_Title

1. `Open Level` の Level Name を `Lvl_InGame` → **`Lvl_Loading`** に変更

---

## 動作確認

1. `Lvl_Title` から PIE 起動
2. ロード画面でバーが 0 → 100% 進み、InGame に切り替われば成功

## 補足

- パッケージ版で初回だけカクつく場合はシェーダーコンパイル（PSO）が原因。この先読みでは解消しないので別対応。
- BPの中身を確認してほしいときは、グラフで `Ctrl+A` → `Ctrl+C` してテキストを貼る（ノード接続まで読める）。
