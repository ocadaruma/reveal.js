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
  - Philippe Flajolet et al. (2007). HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm
  - 集合のCardinality(要素数)をO(1) spaceで高精度に近似できる
  - このスライドでは以下HLLと省略

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

- 64bit intを一様ランダムに選ぶとき、左から0がk bit連続している確率は`1/2^k`

![i003](img/i003.png)

---

## Intuitive explanation

- 言いかえると、`2^k`回試行しないと左から0がk bit連続する数が出ない
- 「64bit intを一様ランダムに選ぶ」ことを繰り返すとして、「最大で左から0が何bit連続したか」だけ記録しておけば「試行した回数」がわかる

---

## Use hash function for randomization

- よい64bit hash関数を使うと、hash値は一様にランダムな64bit intとなる
- つまり、cardinalityが`N`であるデータセットの要素をhash関数にかける ⇔ 「64bit intを一様ランダムに選ぶ」試行をN回繰り返す

---

![i004](img/i004.png)

---

## What does "LogLog" means

- いま、cardinality `N` を「左から連続する0のbit数」で近似した
- つまり`log2(N)`までの数を表現できればよい
- `log2(log2(N))` bit

---

## Stochastic averaging

- このままだと精度が悪いし、2^k単位でしか近似できない
-

---

## Entire HLL algorithm 

---

## HLL is a random variable

- HLLは「データセット

---

### Accuracy

- bucket数を`m`としたとき、標準誤差 `1.04/√m`
  - Flajolet et al. (2007)
  - Redisはdefaultだと16384 bucketなので`1.04/√16384 = 0.008125`
  - => 誤差0.81%

---

## Accuracy

- 「どんな入力に対しても誤差が0.81%におさまる」という意味ではない
- 例: Redis 4.0.9で、以下の入力は誤差が20%となる

```
$ redis-cli PFADD foo 351170 351171 351172 351173 351174\
                      351175 351176 351177 351178 351179
$ redis-cli PFCOUNT foo
(integer) 8
```

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
