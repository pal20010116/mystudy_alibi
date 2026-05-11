# Power BI 手順書：5分刻み停止マップ作成

## CSV読み込み・DirectQuery移行想定・開始日選択で7日分表示する版

---

# 1. この手順で作るもの

この手順では、Power BIで次のような停止状況マップを作成します。

```text
行：号機 × 工場日付
列：07:00〜翌06:55 の5分刻み
色：その5分枠に重なる停止時間の長さ
表示期間：開始日を選び、その日から7日分
```

今回はSQL Serverを使わず、CSVをPower BIに読み込みます。
ただし、将来SQL ServerのDirectQueryに置き換えやすいように、Power Queryでは複雑な展開処理をせず、Power BI側の小さい時刻表とメジャーで停止マップを作ります。

---

# 2. 前提

## データの前提

CSVには、少なくとも次の列がある前提です。

| 列名 | 内容 |
|---|---|
| 号機 | 設備番号、機械番号 |
| 日付 | イベント発生日 |
| 時刻 | イベント発生時刻 |
| 故障名 | 停止理由、故障名 |
| 停止時間 | 停止時間。単位は秒 |
| 枚数 | 生産枚数。今回の停止マップでは必須ではないが残してよい |

重要点は、`停止時間` が **秒** であることです。

例えば、

```text
停止時間 = 600
```

なら、意味は

```text
600秒 = 10分
```

です。

---

# 3. 全体の作成流れ

```text
1. CSVを読み込む
2. Power Queryで必要列だけ残す
3. テーブル名を Event にする
4. Eventに計算列を作る
5. 5分刻みの時刻表 DimTimeBin5 を作る
6. 開始日選択用テーブル DateSelector を作る
7. 開始日から7日分を表示するメジャーを作る
8. 5分セルごとの停止色メジャーを作る
9. Matrixで停止マップを作る
10. スライサーで開始日を選べるようにする
11. 背景色を設定する
12. 表示を整える
```

---

# 4. CSVを読み込む

Power BI Desktopを開きます。

```text
ホーム
→ データを取得
→ テキスト/CSV
```

CSVファイルを選択します。

プレビュー画面が出たら、すぐに読み込みではなく、

```text
データの変換
```

を押します。

---

# 5. Power Queryで必要列だけ残す

Power Queryでは、まず以下の列だけ残します。

```text
号機
日付
時刻
故障名
停止時間
枚数
```

故障コードを使いたい場合は、残しても構いません。
ただし、最初の停止マップ作成では必須ではありません。

## 推奨する列の型

| 列 | 型 |
|---|---|
| 号機 | テキスト |
| 日付 | 日付 |
| 時刻 | 時刻 |
| 故障名 | テキスト |
| 停止時間 | 整数 |
| 枚数 | 整数 |

型を確認したら、

```text
ホーム
→ 閉じて適用
```

を押します。

---

# 6. テーブル名を Event にする

Power BI右側のフィールド一覧で、読み込んだCSVテーブルを右クリックします。

```text
名前の変更
→ Event
```

以降のDAXでは、テーブル名を `Event` として説明します。

---

# 7. Eventに計算列を作る

左側の「データビュー」を開きます。

```text
Event
→ 新しい列
```

以下の列を順番に作ります。

---

## 7.1 発生時刻 event_time

`日付` と `時刻` を結合して、イベントが発生した日時を作ります。

```DAX
event_time =
'Event'[日付] + 'Event'[時刻]
```

---

## 7.2 停止秒数 stop_seconds

`停止時間` は秒なので、そのまま使います。

```DAX
stop_seconds =
'Event'[停止時間]
```

---

## 7.3 停止分数 stop_minutes

色分けは分単位で考えるため、秒を分に変換します。

```DAX
stop_minutes =
'Event'[停止時間] / 60
```

例です。

```text
停止時間 = 600秒
stop_minutes = 600 / 60 = 10分
```

---

## 7.4 停止終了時刻 end_time

イベントの発生時刻に停止秒数を足して、停止終了時刻を作ります。

```DAX
end_time =
'Event'[event_time] + 'Event'[stop_seconds] / 86400
```

`86400` は1日の秒数です。

```text
1日 = 24時間 × 60分 × 60秒 = 86400秒
```

---

## 7.5 工場日付 FactoryDate

このダッシュボードでは、通常の日付ではなく、工場日付を使います。

```text
工場日付：07:00〜翌06:59 を1日として扱う
```

そのため、発生時刻から7時間引いて日付化します。

```DAX
FactoryDate =
VAR T =
    'Event'[event_time] - TIME(7, 0, 0)
RETURN
    DATE(
        YEAR(T),
        MONTH(T),
        DAY(T)
    )
```

例です。

| 実際の発生時刻 | FactoryDate |
|---|---|
| 2026/05/01 06:30 | 2026/04/30 |
| 2026/05/01 07:00 | 2026/05/01 |
| 2026/05/01 23:00 | 2026/05/01 |
| 2026/05/02 06:59 | 2026/05/01 |

---

## 7.6 工場日付ラベル FactoryDateLabel

Power BIは日付列を自動で「年・四半期・月・日」の階層にすることがあります。
Matrixではこれが邪魔になるので、表示用の文字列列を作ります。

```DAX
FactoryDateLabel =
FORMAT(
    'Event'[FactoryDate],
    "yyyy/MM/dd"
)
```

Matrixの行には、`FactoryDate` ではなく `FactoryDateLabel` を使います。

---

# 8. 5分刻みの時刻表を作る

次に、横軸用の時刻表を作ります。

これは元データとは別の小さい表です。
1日24時間を5分刻みにするため、288行になります。

```text
24時間 × 60分 ÷ 5分 = 288行
```

Power BI上部で、

```text
モデリング
→ 新しいテーブル
```

を選び、次を貼り付けます。

```DAX
DimTimeBin5 =
ADDCOLUMNS(
    GENERATESERIES(0, 287, 1),
    "factory_bin", [Value],
    "factory_time_label",
        VAR TotalMin =
            MOD(420 + [Value] * 5, 1440)
        VAR HH =
            INT(TotalMin / 60)
        VAR MM =
            MOD(TotalMin, 60)
        RETURN
            FORMAT(
                TIME(HH, MM, 0),
                "HH:mm"
            )
)
```

この表の意味は次の通りです。

| factory_bin | factory_time_label |
|---:|---|
| 0 | 07:00 |
| 1 | 07:05 |
| 2 | 07:10 |
| ... | ... |
| 287 | 06:55 |

---

# 9. 5分時刻ラベルの並び替え

この設定は必須です。

`factory_time_label` は文字列なので、そのままだと時刻順に並ばないことがあります。

```text
DimTimeBin5[factory_time_label] を選択
→ 列ツール
→ 列で並べ替え
→ DimTimeBin5[factory_bin]
```

これで横軸が、

```text
07:00, 07:05, 07:10, ..., 06:55
```

の順になります。

---

# 10. 開始日選択用テーブル DateSelector を作る

今回はB案を採用します。

```text
開始日を1つ選ぶ
→ その開始日から7日分を表示する
```

このため、開始日選択用の小さいテーブルを作ります。

```text
モデリング
→ 新しいテーブル
```

で、次を作ります。

```DAX
DateSelector =
DISTINCT(
    SELECTCOLUMNS(
        'Event',
        "StartDate", 'Event'[FactoryDate]
    )
)
```

この `DateSelector` は、スライサーで開始日を選ぶためだけの表です。

重要点です。

```text
DateSelector と Event のリレーションは作らなくてよい
```

むしろ、今回はリレーションを作らず、メジャーで制御します。

---

# 11. 開始日から7日分を表示するメジャーを作る

```text
モデリング
→ 新しいメジャー
```

で、次を作ります。

```DAX
ShowSelected7FactoryDays =
VAR SelectedStartDate =
    SELECTEDVALUE('DateSelector'[StartDate])
VAR CurrentDate =
    MAX('Event'[FactoryDate])
RETURN
    IF(
        ISBLANK(SelectedStartDate),
        1,
        IF(
            CurrentDate >= SelectedStartDate
                && CurrentDate <= SelectedStartDate + 6,
            1,
            0
        )
    )
```

このメジャーの意味は次の通りです。

```text
開始日として 2026/05/01 を選ぶ
↓
2026/05/01〜2026/05/07 の7工場日だけ表示する
```

---

# 12. 5分セルごとの停止色メジャーを作る

このメジャーが停止マップの本体です。

Matrixの各セルについて、次を判定します。

```text
この号機
この工場日付
この5分枠
に停止が重なっているか
```

```text
モデリング
→ 新しいメジャー
```

で、次を作ります。

```DAX
StopSeverityInBin5 =
VAR CurrentMachine =
    SELECTEDVALUE('Event'[号機])

VAR CurrentFactoryDate =
    MAX('Event'[FactoryDate])

VAR CurrentBin =
    SELECTEDVALUE('DimTimeBin5'[factory_bin])

VAR BinStart =
    CurrentFactoryDate
        + TIME(7, 0, 0)
        + DIVIDE(CurrentBin * 5, 1440)

VAR BinEnd =
    BinStart + DIVIDE(5, 1440)

VAR MaxStopMinInBin =
    CALCULATE(
        MAX('Event'[stop_minutes]),
        FILTER(
            ALLSELECTED('Event'),
            'Event'[号機] = CurrentMachine
                && 'Event'[stop_seconds] > 0
                && 'Event'[event_time] < BinEnd
                && 'Event'[end_time] > BinStart
        )
    )

RETURN
    SWITCH(
        TRUE(),
        ISBLANK(CurrentMachine), BLANK(),
        ISBLANK(CurrentFactoryDate), BLANK(),
        ISBLANK(CurrentBin), BLANK(),
        ISBLANK(MaxStopMinInBin), 0,
        MaxStopMinInBin >= 60, 60,
        MaxStopMinInBin >= 30, 30,
        MaxStopMinInBin >= 10, 10,
        MaxStopMinInBin > 0, 5,
        0
    )
```

---

# 13. StopSeverityInBin5 の色分け意味

`StopSeverityInBin5` は、停止時間を次の値に変換します。

| 値 | 意味 |
|---:|---|
| 0 | 停止なし |
| 5 | 0分超〜10分未満 |
| 10 | 10分以上〜30分未満 |
| 30 | 30分以上〜60分未満 |
| 60 | 60分以上 |

例です。

```text
停止時間 = 2400秒
2400秒 = 40分
```

この場合、40分停止なので、該当する5分セルが8個分、横方向に塗られます。

```text
40分 ÷ 5分 = 8セル
```

---

# 14. Matrixを作る

レポート画面に戻ります。

```text
視覚化
→ Matrix
```

を選びます。

フィールドは次のように入れます。

| 場所 | 入れるもの |
|---|---|
| 行 | Event[号機] |
| 行 | Event[FactoryDateLabel] |
| 列 | DimTimeBin5[factory_time_label] |
| 値 | StopSeverityInBin5 |

注意点です。

```text
FactoryDate の日付階層は使わない
```

行には必ず、

```text
Event[FactoryDateLabel]
```

を入れます。

---

# 15. 開始日スライサーを作る

レポート画面で、

```text
視覚化
→ スライサー
```

を選びます。

スライサーに入れる列は、

```text
DateSelector[StartDate]
```

です。

このスライサーで開始日を1つ選ぶと、その日から7日分が表示されます。

## スライサーの見た目

Power BIのバージョンによって、スライサーの表示設定名が違います。

ドロップダウンが使える場合は、ドロップダウンにします。

```text
スライサーを選択
→ 書式
→ スライサー設定
→ スタイル
→ ドロップダウン
```

もしドロップダウン設定が見つからない場合は、リスト型のままで問題ありません。

重要なのは、次です。

```text
DateSelector[StartDate] から開始日を1つ選ぶ
```

できれば、単一選択をオンにします。

```text
スライサーを選択
→ 書式
→ 選択
→ 単一選択：オン
```

単一選択が見つからない場合は、手動で開始日を1つだけ選んでください。

---

# 16. Matrixに7日表示フィルターをかける

Matrixを選択した状態で、右側のフィルター欄を使います。

```text
このビジュアルでのフィルター
→ ShowSelected7FactoryDays
```

を入れます。

条件は、

```text
次の値以上
1
```

にします。

これで、スライサーで選んだ開始日から7日分だけがMatrixに表示されます。

---

# 17. 5分列を最後まで表示する

時刻が途中までしか出ない場合は、Matrixの列に入れている

```text
DimTimeBin5[factory_time_label]
```

の右側の▼を押します。

そして、

```text
データのない項目を表示
```

をオンにします。

これで、停止がない時刻列も表示されやすくなります。

また、データビューで `DimTimeBin5` を開き、`factory_bin` が次の範囲になっているか確認してください。

```text
0〜287
```

もし `0〜99` などで止まっている場合は、時刻表が途中までしか作れていません。

---

# 18. 背景色を付ける

Matrixの値に入れた `StopSeverityInBin5` のメニューを開きます。

```text
StopSeverityInBin5 の ▼
→ 条件付き書式
→ 背景色
```

「ルール」で次のように設定します。

| 条件 | 色の意味 | 例 |
|---|---|---|
| 0 | 停止なし | 白 |
| 5以上 10未満 | 短い停止 | 薄い黄色 |
| 10以上 30未満 | 中程度の停止 | 黄色 |
| 30以上 60未満 | 長めの停止 | オレンジ |
| 60以上 | 長時間停止 | 赤 |

数字が邪魔な場合は、フォント色も背景色に近づけます。

```text
StopSeverityInBin5 の ▼
→ 条件付き書式
→ フォントの色
```

背景色と同じ、または近い色にすると、ヒートマップのように見えます。

---

# 19. Matrixの見た目を整える

5分刻みは288列あるため、Power BI標準Matrixでは横にかなり長くなります。
そのため、できるだけ列幅と文字を小さくします。

Matrixを選択し、右側の「視覚化」ペインでブラシアイコンを押します。

推奨設定は次の通りです。

| 設定 | 推奨値 |
|---|---|
| 値のテキストサイズ | 5 |
| 列見出しのテキストサイズ | 5 |
| 行の余白 | 最小 |
| 行の小計 | オフ |
| 列の小計 | オフ |
| 列見出しの折り返し | オフ |
| 自動列幅 | オフ |

`自動列幅` がオンだと、時刻ラベルに合わせて列が広がり、1日全体が見えにくくなります。
オフにした後、列幅をできるだけ狭くしてください。

---

# 20. 5分刻み版の注意点

5分刻みは、停止のタイミングを細かく見られる反面、列数が多いです。

```text
1日 = 288列
7日 × 号機数分の行
```

そのため、Power BI標準Matrixでは、基本的に横スクロール前提になります。

用途としては、次のように考えるとよいです。

| 用途 | 推奨ビン |
|---|---|
| 1日全体を一目で見る | 15分または30分 |
| 停止の細かい発生タイミングを見る | 5分 |
| 管理者向けの大画面表示 | 15分または30分 |
| 詳細分析 | 5分 |

今回は、詳細分析寄りなので5分刻みを採用します。

---

# 21. 完成チェック

完成したら、次を確認してください。

```text
横軸が 07:00 から始まる
最後が 06:55 になる
00:00 が途中に出る
行が 号機 × 工場日付 になっている
スライサーで開始日を選べる
選んだ開始日から7日分だけ表示される
停止が点ではなく横方向の帯になる
停止時間が長いほど濃い色になる
停止時間600秒が10分として扱われている
```

---

# 22. よくあるトラブルと直し方

## 22.1 FactoryDateが年・四半期・月・日に分かれる

原因は、Power BIが日付階層を自動で使っているためです。

対策は、Matrixの行に `FactoryDate` ではなく、次を使うことです。

```text
Event[FactoryDateLabel]
```

---

## 22.2 時刻が15:15までしか出ない

原因候補は2つです。

1つ目は、`DimTimeBin5` が途中までしか作れていないことです。
データビューで `DimTimeBin5` を開き、`factory_bin` が `0〜287` まであるか確認します。

2つ目は、停止がない時刻列が非表示になっていることです。
Matrixの列に入れている `factory_time_label` の設定で、

```text
データのない項目を表示
```

をオンにしてください。

---

## 22.3 全部赤くなる

原因は、停止時間の単位を間違えている可能性があります。

今回の前提は、`停止時間` が秒です。

正しい列は次です。

```DAX
stop_seconds =
'Event'[停止時間]
```

```DAX
stop_minutes =
'Event'[停止時間] / 60
```

間違って、

```DAX
stop_minutes =
'Event'[停止時間]
```

としていると、600秒を600分として扱ってしまい、色がほぼ全部赤になります。

---

## 22.4 スライサーで選んでも7日分に変わらない

確認点は3つです。

1つ目は、スライサーに入れている列です。

```text
DateSelector[StartDate]
```

を使ってください。

2つ目は、Matrixのビジュアルフィルターです。

```text
ShowSelected7FactoryDays >= 1
```

になっているか確認してください。

3つ目は、DateSelectorとEventをリレーションでつないでいないかです。
今回の方法では、DateSelectorは独立テーブルとして使います。

---

## 22.5 停止が横帯にならない

`StopSeverityInBin5` が正しく入っているか確認してください。

Matrixの値には、単なる停止時間ではなく、次を入れます。

```text
StopSeverityInBin5
```

このメジャーが、5分枠と停止イベントの重なりを判定しています。

---

# 23. 将来DirectQueryに置き換えるときの考え方

今回はCSVをImportで読み込んでいます。
ただし、構成はDirectQueryに移しやすいようにしています。

将来SQL Serverに置き換えるときは、基本的に次の考え方です。

```text
CSV由来の Event
↓
SQL Server由来の Event に差し替える
```

そのとき、次の列がSQL Server側またはPower BI側で用意できれば、今回のMatrixは近い形で再利用できます。

```text
号機
日付
時刻
event_time
stop_seconds
stop_minutes
end_time
FactoryDate
FactoryDateLabel
```

DirectQueryでは、5分展開テーブルをPower BI側で大量に作るより、今回のように小さい時刻表とメジャーで試す方が、会社のSQL Serverを変更しにくい場合には始めやすいです。

ただし、データ量が増えて重くなった場合は、最終的にはSQL Server側に5分展開済みのViewやテーブルを作る方が安定します。

---

# 24. 最終的なフィールド構成

## Event

| 列 | 用途 |
|---|---|
| 号機 | 行、絞り込み |
| 日付 | event_time作成用 |
| 時刻 | event_time作成用 |
| 故障名 | 詳細表示用 |
| 停止時間 | 元の停止秒数 |
| stop_seconds | 停止秒数 |
| stop_minutes | 停止分数 |
| event_time | 停止開始時刻 |
| end_time | 停止終了時刻 |
| FactoryDate | 工場日付判定 |
| FactoryDateLabel | Matrix表示用 |

## DimTimeBin5

| 列 | 用途 |
|---|---|
| factory_bin | 5分ビン番号、並び替え用 |
| factory_time_label | Matrixの列表示 |

## DateSelector

| 列 | 用途 |
|---|---|
| StartDate | 7日表示の開始日選択 |

## メジャー

| メジャー | 用途 |
|---|---|
| ShowSelected7FactoryDays | 開始日から7日分だけ表示 |
| StopSeverityInBin5 | 5分セルの停止色判定 |

---

# 25. 完成形

最終的なレポートは、まず次の2つだけあれば十分です。

```text
1. DateSelector[StartDate] のスライサー
2. 5分刻み停止マップのMatrix
```

Matrixの設定は次です。

| 場所 | フィールド |
|---|---|
| 行 | Event[号機] |
| 行 | Event[FactoryDateLabel] |
| 列 | DimTimeBin5[factory_time_label] |
| 値 | StopSeverityInBin5 |
| ビジュアルフィルター | ShowSelected7FactoryDays >= 1 |

この形で、開始日を選ぶたびに、その日から7日分の停止状況を5分刻みで確認できます。
