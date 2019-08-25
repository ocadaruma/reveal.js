## HyperLogLog sketchは面白い

### builderscon tokyo 2019
#### Haruki Okada (@ocadaruma)

---

## Introduction

- リアルタイムアクセス解析システムを作るとする
  - "HogeAnalytics"
  - Webサイトのアクセス統計をリアルタイムに提供
      - ページごとのPV数
      - ページごとのユニークユーザー(cookie)数

---

![i001](img/i001.png)

---

![i002](img/i002.png)

---

## Problem

- PV数: `Map<PageURL, Long>`を的なのを持ち、アクセスのたびにインクリメントすればよい
- ユニークユーザー数: どうやって出すか？
  - Count-distinct problemとよばれる
  - ナイーブな方法だとどうしても非効率になる

---

## Simple set approach

```java
class UniqueUserStats {
  private Map<PageURL, Set<CookieId>> map;

  public void record(PageURL url, CookieId user) {
    Set<CookieId> set = map.get(url);
    set.add(user);
  }

  public int countUU(PageURL url) {
    Set<CookieId> set = map.get(url);
    return set.size();
  }
}
```

---

## Simple set approach

- O(N) space必要 (N = ユニークユーザー数)
- UUID StringのcookieIdを考える(36 byte)
  - 1億UUの保存に3.6GB
  - 100URLで360GB
  - たくさんのWebサイトを計測することを考えるとリーズナブルじゃない
- abuse耐性が無い
  - でたらめなcookieIdを送り続けると、いずれMemoryに保持できる数を超えてサーバーが死ぬ

---

## Batch approach

- 以下のようなSQLで事前にUU数を集計しておく

```sql
SELECT
  url, COUNT(DISTINCT cookie_id) uu
FROM
  access_log
GROUP BY
  url
```

- "リアルタイム"要件をみたせない

---

## Solution: Probabilistic approach

- 確率的アルゴリズムによる近似値を使う
- **HyperLogLog**
  - 集合のCardinality(要素数)を近似するアルゴリズム
  - O(1) spaceで高精度に近似できる
  - 本資料では、以下HLLと略する

---

## Agenda

1. How HLL works
2. HLL on Redis and BigQuery
3. Deep dive into Redis HLL
4. HyperMinHash

---

# How HLL works

---

## Intuitive explanation

- 64bit intを一様ランダムに選んだとき、最初に0がk bit連続する確率は`1/2^k`

---

## Intuitive explanation

- i.e. `2^k`回試行しないと最初に0がk bit連続する数が出ない

---

## What does "LogLog" means

- N (簡単のためN = `2^k`とする)

---

## Stochastic averaging

---

## Accuracy

- Relative error

---

## Accuracy

- HLLによる推定値は確率変数
- 期待値がcardinalityになる

## Entire HLL algorithm

---

## HLL Sketch

---

## Sketch and estimator

---

## Improve sketch

---

## Improve estimator

---

## HLL on Redis

---

## Mergeability

---

## Performance

---

## HLL on BigQuery

---

## Deep dive into Redis HLL

- 最初に導入されたバージョン

---

## Redis v4

---

## Redis v5

---

## Redis HLL vulnerability

---

## HyperMinHash

---

## HyperMinHash sketch

---

## Conclusion