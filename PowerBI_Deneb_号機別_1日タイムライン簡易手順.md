# Power BI Deneb 号機別 1日停止タイムライン 簡易手順書

## 1. 目的

Power BIで、選択した1日分の停止イベントを、号機別の横棒タイムラインとして表示する。

縦軸は号機、横軸は工場日付の1日、つまり7:00から翌7:00までとする。

この画面は、日付スライサーで1日を選択し、その日の各号機の停止発生状況を横並びで比較するために使う。

## 2. 前提

Power BI側で、以下の列をDenebビジュアルに渡す。

| 列名 | 内容 |
|---|---|
| 号機名 | 号機名。例：1号機、2号機 |
| 工場日付表示 | 表示用の日付 |
| 時刻 | 停止開始時刻 |
| 停止時間 | 停止時間。単位は秒 |
| 故障名 | 停止理由、故障名 |

注意：データ側の列名が `号機名` ではなく `号機` の場合は、Denebコード内の `号機名` をすべて `号機` に置き換える。

## 3. Power BI側の設定

日付スライサーを配置し、工場日付を1日だけ選択できるようにする。

このDenebビジュアルは、Power BI側で1日分に絞られたデータを受け取る前提で作成する。

Denebに渡すデータは、停止時間が0より大きい行だけに絞る。

フィルター条件は以下。

```text
停止時間 > 0
```

## 4. Denebビジュアルの作成

Power BIでDenebビジュアルを追加し、必要な列を値に追加する。

その後、Denebのエディタに以下のJSONを貼り付ける。

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "name": "dataset"
  },
  "autosize": {
    "type": "fit-x",
    "contains": "padding"
  },
  "transform": [
    {
      "calculate": "toDate(datum['時刻'])",
      "as": "開始時刻_Date"
    },
    {
      "calculate": "hours(datum['開始時刻_Date'])",
      "as": "開始時"
    },
    {
      "calculate": "minutes(datum['開始時刻_Date'])",
      "as": "開始分"
    },
    {
      "calculate": "seconds(datum['開始時刻_Date'])",
      "as": "開始秒"
    },
    {
      "calculate": "(datum['開始時'] < 10 ? '0' : '') + datum['開始時'] + ':' + (datum['開始分'] < 10 ? '0' : '') + datum['開始分'] + ':' + (datum['開始秒'] < 10 ? '0' : '') + datum['開始秒']",
      "as": "停止開始時刻_表示"
    },
    {
      "calculate": "datum['開始時'] < 7 ? datum['開始時'] + 24 : datum['開始時']",
      "as": "開始時_24"
    },
    {
      "calculate": "datum['開始時_24'] * 3600 + datum['開始分'] * 60 + datum['開始秒']",
      "as": "停止開始_総秒_24"
    },
    {
      "calculate": "datum['停止開始_総秒_24'] / 3600",
      "as": "停止開始_時刻値"
    },
    {
      "calculate": "datum['停止時間'] / 3600",
      "as": "停止時間_時間"
    },
    {
      "calculate": "datum['停止開始_総秒_24'] + datum['停止時間']",
      "as": "停止終了_総秒_24"
    },
    {
      "calculate": "datum['停止終了_総秒_24'] / 3600",
      "as": "停止終了_時刻値"
    },
    {
      "calculate": "floor(datum['停止終了_総秒_24'] / 3600)",
      "as": "終了時_24"
    },
    {
      "calculate": "floor((datum['停止終了_総秒_24'] % 3600) / 60)",
      "as": "終了分"
    },
    {
      "calculate": "floor(datum['停止終了_総秒_24'] % 60)",
      "as": "終了秒"
    },
    {
      "calculate": "datum['終了時_24'] >= 24 ? datum['終了時_24'] - 24 : datum['終了時_24']",
      "as": "終了時"
    },
    {
      "calculate": "(datum['終了時'] < 10 ? '0' : '') + datum['終了時'] + ':' + (datum['終了分'] < 10 ? '0' : '') + datum['終了分'] + ':' + (datum['終了秒'] < 10 ? '0' : '') + datum['終了秒']",
      "as": "停止終了時刻"
    },
    {
      "calculate": "datum['停止時間'] / 60",
      "as": "停止時間_分"
    },
    {
      "calculate": "datum['停止時間'] < 300 ? '5分未満' : datum['停止時間'] < 900 ? '5〜15分' : datum['停止時間'] < 1800 ? '15〜30分' : datum['停止時間'] < 3600 ? '30〜60分' : '60分以上'",
      "as": "停止時間区分_表示"
    }
  ],
  "width": "container",
  "height": 420,
  "layer": [
    {
      "transform": [
        {
          "aggregate": [
            {
              "op": "count",
              "as": "dummy_count"
            }
          ],
          "groupby": [
            "号機名"
          ]
        }
      ],
      "mark": {
        "type": "bar",
        "color": "#43A047",
        "opacity": 0.16,
        "cornerRadius": 0
      },
      "encoding": {
        "x": {
          "datum": 7,
          "type": "quantitative",
          "scale": {
            "domain": [
              7,
              31
            ]
          },
          "axis": {
            "title": "時刻",
            "values": [
              7,
              9,
              12,
              15,
              18,
              21,
              24,
              27,
              31
            ],
            "labelExpr": "datum.value >= 24 ? datum.value - 24 + ':00' : datum.value + ':00'"
          }
        },
        "x2": {
          "datum": 31
        },
        "y": {
          "field": "号機名",
          "type": "nominal",
          "sort": "ascending",
          "scale": {
            "paddingInner": 0.28,
            "paddingOuter": 0.18
          },
          "axis": {
            "title": null,
            "labelFontSize": 10,
            "labelLimit": 90,
            "labelPadding": 6
          }
        }
      }
    },
    {
      "mark": {
        "type": "bar",
        "cornerRadius": 0
      },
      "encoding": {
        "x": {
          "field": "停止開始_時刻値",
          "type": "quantitative",
          "scale": {
            "domain": [
              7,
              31
            ]
          },
          "axis": {
            "title": "時刻",
            "values": [
              7,
              9,
              12,
              15,
              18,
              21,
              24,
              27,
              31
            ],
            "labelExpr": "datum.value >= 24 ? datum.value - 24 + ':00' : datum.value + ':00'"
          }
        },
        "x2": {
          "field": "停止終了_時刻値"
        },
        "y": {
          "field": "号機名",
          "type": "nominal",
          "sort": "ascending",
          "scale": {
            "paddingInner": 0.28,
            "paddingOuter": 0.18
          },
          "axis": {
            "title": null,
            "labelFontSize": 10,
            "labelLimit": 90,
            "labelPadding": 6
          }
        },
        "color": {
          "field": "停止時間区分_表示",
          "type": "nominal",
          "scale": {
            "domain": [
              "5分未満",
              "5〜15分",
              "15〜30分",
              "30〜60分",
              "60分以上"
            ],
            "range": [
              "#1E88E5",
              "#CDDC39",
              "#FFC107",
              "#FB8C00",
              "#D32F2F"
            ]
          },
          "legend": {
            "title": "停止時間区分"
          }
        },
        "tooltip": [
          {
            "field": "号機名",
            "type": "nominal",
            "title": "号機"
          },
          {
            "field": "工場日付表示",
            "type": "nominal",
            "title": "工場日付"
          },
          {
            "field": "停止開始時刻_表示",
            "type": "nominal",
            "title": "停止開始時刻"
          },
          {
            "field": "停止終了時刻",
            "type": "nominal",
            "title": "停止終了時刻"
          },
          {
            "field": "停止時間_分",
            "type": "quantitative",
            "title": "停止時間[分]",
            "format": ".1f"
          },
          {
            "field": "故障名",
            "type": "nominal",
            "title": "故障名"
          },
          {
            "field": "停止時間区分_表示",
            "type": "nominal",
            "title": "停止時間区分"
          }
        ]
      }
    }
  ],
  "config": {
    "view": {
      "stroke": null
    },
    "axis": {
      "grid": true,
      "labelFontSize": 11,
      "titleFontSize": 12
    },
    "legend": {
      "labelFontSize": 11,
      "titleFontSize": 12
    }
  }
}
```

## 5. 表示の調整

グラフの高さは以下で調整する。

```json
"height": 420
```

号機数が多くて窮屈な場合は、`520` などに上げる。

バー同士の間隔は以下で調整する。

```json
"paddingInner": 0.28
```

値を大きくするとバーが細くなり、間隔が広がる。

値を小さくするとバーが太くなり、間隔が狭くなる。

## 6. 注意点

このDenebは、停止時間があるイベントだけを表示する。

停止がない号機も必ず表示したい場合は、停止イベントだけでなく、各号機に対してダミー行を持つテーブルを別途用意する必要がある。

日付を複数選択すると、複数日の停止イベントが同じ号機の行に重なって表示される。そのため、日付スライサーは1日選択にする。
