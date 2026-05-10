# Power BI 教科書：設備停止監視ダッシュボード作成手順
## 5分ビン版・初心者向け完全手順

---

# 1. ゴール

この手順書では、次のダッシュボードをPower BIで初心者でも再現できるようにします。

- KPIカード
- イベントマップ（5分ビン）
- 主要停止イベント
- 工場日付別サマリー
- 停止時間ごとの停止件数割合

特徴：

- 工場日付：7:00〜翌6:59
- 表示期間：7日固定
- イベントマップ：
  「影響範囲＋イベント重大度色型」

---

# 2. 完成イメージ

## 上段

- 総停止時間
- 停止件数
- 最長停止
- 簡易稼働率

## 中央左

イベントマップ

## 中央右

主要停止イベント

## 下段左

工場日付別サマリー

## 下段右

停止時間ごとの停止件数割合

---

# 3. 使用ファイル

## 元イベントデータ

```text
sample_event_log_7days_sequential_stop_with_delay_original_columns.csv
```

## 5分ビン展開データ

```text
event_map_5min_pattern_c_powerbi.csv
```

---

# 4. Power BI Desktopをインストール

Microsoft公式：

https://powerbi.microsoft.com/

インストール後、Power BI Desktopを起動。

---

# 5. 元CSVを読み込む

## 操作

```text
ホーム
→ データを取得
→ テキスト/CSV
```

元イベントCSVを選択。

```text
データの変換
```

を押す。

---

# 6. Power Queryで型を確認

| 列 | 型 |
|---|---|
| 発生時刻 | 日付/時刻 |
| 故障名 | テキスト |
| 停止時間 | テキスト |

---

# 7. stop_secondsを作る

## 操作

```text
列の追加
→ カスタム列
```

列名：

```text
stop_seconds
```

式：

```powerquery
let
    txt = [停止時間],

    hour =
        if Text.Contains(txt, "時間")
        then try Number.FromText(Text.BeforeDelimiter(txt, "時間")) otherwise 0
        else 0,

    afterHour =
        if Text.Contains(txt, "時間")
        then Text.AfterDelimiter(txt, "時間")
        else txt,

    minute =
        if Text.Contains(afterHour, "分")
        then try Number.FromText(Text.BeforeDelimiter(afterHour, "分")) otherwise 0
        else 0,

    afterMinute =
        if Text.Contains(afterHour, "分")
        then Text.AfterDelimiter(afterHour, "分")
        else afterHour,

    second =
        if Text.Contains(afterMinute, "秒")
        then try Number.FromText(Text.BeforeDelimiter(afterMinute, "秒")) otherwise 0
        else 0
in
    hour * 3600 + minute * 60 + second
```

型：

```text
整数
```

---

# 8. Power Queryを閉じる

```text
ホーム
→ 閉じて適用
```

---

# 9. テーブル名変更

右側フィールド欄：

```text
右クリック
→ 名前変更
```

```text
Event
```

にする。

---

# 10. DAX列を作る

## 操作

```text
データビュー
→ Event
→ 新しい列
```

---

## stop_minutes

```DAX
stop_minutes =
'Event'[stop_seconds] / 60
```

---

## event_time

```DAX
event_time =
'Event'[発生時刻]
```

---

## end_time

```DAX
end_time =
'Event'[event_time]
+ 'Event'[stop_seconds] / 86400
```

---

# 11. 工場日付を作る

## FactoryDateTime

```DAX
FactoryDateTime =
'Event'[event_time] - TIME(7,0,0)
```

---

## FactoryDate

```DAX
FactoryDate =
FORMAT(
    'Event'[FactoryDateTime],
    "MM/dd"
)
```

---

# 12. 5分ビンを作る

## FactoryElapsedMin

```DAX
FactoryElapsedMin =
MOD(
    HOUR('Event'[event_time]) * 60
    + MINUTE('Event'[event_time])
    - 420,
    1440
)
```

---

## FactoryBin5

```DAX
FactoryBin5 =
INT(
    'Event'[FactoryElapsedMin] / 5
)
```

---

## FactoryTimeLabel5

```DAX
FactoryTimeLabel5 =
VAR TotalMin =
    MOD(
        420 + 'Event'[FactoryBin5] * 5,
        1440
    )
VAR HH =
    INT(TotalMin / 60)
VAR MM =
    MOD(TotalMin, 60)
RETURN
    FORMAT(
        TIME(HH, MM, 0),
        "HH:mm"
    )
```

---

# 13. 並び替え設定

ここ非常に重要。

## 操作

```text
FactoryTimeLabel5 を選択
→ 列ツール
→ 列で並べ替え
→ FactoryBin5
```

これをしないと時刻順にならない。

---

# 14. 停止時間ランクを作る

```DAX
StopClass =
SWITCH(
    TRUE(),
    'Event'[stop_minutes] < 1, "0-1分",
    'Event'[stop_minutes] < 5, "1-5分",
    'Event'[stop_minutes] < 10, "5-10分",
    'Event'[stop_minutes] < 30, "10-30分",
    'Event'[stop_minutes] < 60, "30-60分",
    "60分以上"
)
```

---

# 15. StopClass並び順

```DAX
StopClassOrder =
SWITCH(
    'Event'[StopClass],
    "0-1分", 1,
    "1-5分", 2,
    "5-10分", 3,
    "10-30分", 4,
    "30-60分", 5,
    "60分以上", 6
)
```

## 操作

```text
StopClass
→ 列で並べ替え
→ StopClassOrder
```

---

# 16. EventExpandedを読み込む

## 操作

```text
ホーム
→ データを取得
→ テキスト/CSV
```

```text
event_map_5min_pattern_c_powerbi.csv
```

を読み込む。

テーブル名：

```text
EventExpanded
```

---

# 17. EventExpandedの並び替え

## 操作

```text
factory_time_label
→ 列で並べ替え
→ factory_bin
```

---

# 18. KPIメジャー

## TotalStopHour

```DAX
TotalStopHour =
SUM('Event'[stop_seconds]) / 3600
```

---

## StopCount

```DAX
StopCount =
COUNTROWS('Event')
```

---

## MaxStopMin

```DAX
MaxStopMin =
MAX('Event'[stop_minutes])
```

---

## ImpactStopHour

```DAX
ImpactStopHour =
SUM('EventExpanded'[overlap_minutes]) / 60
```

---

## SimpleAvailability

```DAX
SimpleAvailability =
VAR StopHour = [ImpactStopHour]
VAR PlannedHour = 7 * 24
RETURN
    1 - DIVIDE(StopHour, PlannedHour)
```

表示形式：

```text
パーセント
```

---

# 19. KPIカードを作る

## 操作

```text
視覚化
→ カード
```

を4つ置く。

| カード | メジャー |
|---|---|
| 総停止時間 | TotalStopHour |
| 停止件数 | StopCount |
| 最長停止 | MaxStopMin |
| 簡易稼働率 | SimpleAvailability |

---

# 20. イベントマップを作る

## Matrixを配置

```text
視覚化
→ Matrix
```

---

## フィールド

| 場所 | フィールド |
|---|---|
| 行 | factory_date |
| 列 | factory_time_label |
| 値 | SeverityForColor |

---

## SeverityForColor

```DAX
SeverityForColor =
MAX('EventExpanded'[severity_for_color])
```

---

# 21. イベントマップ色設定

## 操作

```text
SeverityForColor の▼
→ 条件付き書式
→ 背景色
```

---

## 推奨色

| 値 | 色 |
|---:|---|
| 0 | 白 |
| 5 | 薄黄 |
| 10 | 黄 |
| 30 | オレンジ |
| 60以上 | 赤 |

---

# 22. Matrix軽量化設定

5分ビンは288列ある。

## 設定

```text
書式
→ Grid
→ Row padding 小
```

```text
書式
→ Values
→ Text size 6〜7
```

```text
書式
→ Column headers
→ Text size 6〜7
```

---

# 23. 主要停止イベント

## ビジュアル

```text
Table
```

---

## 表示列

| 列 |
|---|
| event_time |
| stop_minutes |
| 故障名 |

---

## 並び替え

```text
stop_minutes DESC
```

---

# 24. 工場日付別サマリー

## DailyStopCount

```DAX
DailyStopCount =
DISTINCTCOUNT('EventExpanded'[event_id])
```

---

## DailyMaxStopMin

```DAX
DailyMaxStopMin =
MAX('EventExpanded'[event_stop_minutes])
```

---

## DailyImpactStopMin

```DAX
DailyImpactStopMin =
SUM('EventExpanded'[overlap_minutes])
```

---

## DailyRunRatio

```DAX
DailyRunRatio =
VAR StopMin = [DailyImpactStopMin]
VAR PlannedMin = 24 * 60
RETURN
    1 - DIVIDE(StopMin, PlannedMin)
```

---

# 25. 稼働率バー

## 操作

```text
DailyRunRatio ▼
→ 条件付き書式
→ データバー
```

---

# 26. 停止時間ごとの件数割合

## グラフ

```text
集合縦棒グラフ
```

---

## X軸

```text
StopClass
```

---

## Y軸

```DAX
StopClassCount =
COUNTROWS('Event')
```

---

# 27. レイアウト

## 推奨

```text
上段：
KPIカード

中央左：
イベントマップ

中央右：
主要停止イベント

下段左：
工場日付別サマリー

下段右：
停止時間割合
```

---

# 28. 重要な実務ポイント

## EventExpandedが必須

イベントマップは、

```text
EventExpanded
```

を使う。

元イベントテーブルではダメ。

---

## overlap_minutesで色付けしない

間違い：

```text
5分セルを5分色
```

正解：

```text
severity_for_color
```

で塗る。

---

## 40分停止の正しい見え方

```text
08:20〜09:00
```

なら、

```text
08:20〜08:25
08:25〜08:30
...
08:55〜09:00
```

全部が40分色。

---

# 29. 最終チェック

- 横軸が07:00開始
- 00:00が途中
- 工場日付になっている
- 停止が帯になっている
- 長時間停止が濃い
- 5分ビンで細かく見える
- 稼働率バーが出ている
- 停止時間割合がランク順

---

# 30. 実務おすすめ

| 用途 | 推奨ビン |
|---|---|
| 日常監視 | 30分 |
| 詳細分析 | 5分 |
| 微停止分析 | 5分 |
| 管理者画面 | 30分 |
| 大型モニター | 5分可 |

