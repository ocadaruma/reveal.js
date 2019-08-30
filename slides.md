## HyperLogLog sketchは面白い

### builderscon tokyo 2019
#### Haruki Okada (@ocadaruma)

---

## Introduction

- リアルタイムアクセス解析システム "HogeAnalytics" を作るとする
- Webサイトのアクセス統計をリアルタイムに提供
  - ページごとのPV数
  - ページごとのユニークユーザー(cookie)数

---

![i001](img/i001.png)

---

![i002](img/i002.png)

---

## Difficulty

- PV数: `Map<PageURL, Long>`を的なのを持ち、アクセスのたびにインクリメントすればよい
- ユニークユーザー数: どうやって出すか？
  - Count-distinct problemとよばれる
  - ナイーブな方法だと非効率になる

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
- abuse耐性が無い
  - でたらめなcookieIdを送り続けるといずれMemoryに保持できる数を超えてサーバーが死ぬ

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
  - Philippe Flajolet et al., 2007. HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm
  - 集合のCardinality(要素数)をO(1) spaceで高精度に近似できる
  - このスライドでは以下HLLと省略

---

## Agenda

1. How HLL works
2. HLL on Redis
3. HyperMinHash

---

# How HLL works

---

## Intuitive explanation

- 64bit intを一様ランダムに選ぶとき、先頭から0がk bit連続している確率は`1/2^k`

![i003](img/i003.png)

---

## Intuitive explanation

- 言いかえると、`2^k`回試行しないと先頭から0がk bit連続する数が出ない
- 「64bit intを一様ランダムに選ぶ」ことを繰り返すとき「最大で先頭から0が何bit連続したか」だけ記録しておけば「試行した回数」を推定できる

---

## Use hash function for randomization

- よい64bit hash関数を使うと、hash値は64bit int空間に一様に分布する
- つまり、cardinalityが`N`であるデータセットの要素をhash関数にかける ⇔ 「64bit intを一様ランダムに選ぶ」試行をN回繰り返す

---

![i004](img/i004.png)

---

## What does "LogLog" means

- いま、cardinality `N` を「先頭から連続する0のbit数」で近似した
- つまり`log2(N)`までの数だけで`N`を近似したことになる
- `log2(log2(N))` bit

---

## Stochastic averaging

- これだけだと精度が悪いし、2^k単位でしか近似できない
- 複数のhash関数を使ってその平均を取ることで正確になる

---

## Stochastic averaging

- データセットの各要素に対し複数のhash関数をかけるのは時間がかかる
- ハッシュ値の先頭`p` bitを使って、`m=2^p`個のbucketに振り分ける
- bucketごとに、残りの`64-p` bitを使って、先頭から連続する0-bitを数える

---

![i006](img/i006.png)

---

## HLL Sketch

- この、先頭から連続する0-bit数を保持した`m`個のbucket列をsketchとよぶ

```java
byte[] sketch = new byte[m];
```

---

## Stochastic averaging

- sketchをなめて、bucketごとの値を**いい感じに**足し合わせる
  - Flajolet & Martin., 1985. Probabilistic Counting Algorithms for Data Base Applications

---

## Estimation

- sketchをMとしてM[i]でi番目のbucketを表すと、HLLでは以下のように最終的な値を計算する
- (`αm`は、bucket数`m`に依存した補正係数)

![i007](img/i007.png)

---

## Entire HLL process

- したがって、HLLは2 stepにわけられる
- (1) Sketch construction
  - データセットをなめて、各要素をbucketに振り分け、先頭の連続する0-bitを数えて保存
- (2) Estimation
  - sketchをなめて、前述の式で値を計算する

---

## Pseudo code

- Sketch construction

```java
byte[] sketch = new byte[m];

for (String element : dataset) {
    long hash = calcHash(element);
    int bucket = calcBucket(hash);
    byte leadingZeros = calcLeadingZeros(hash);
    
    sketch[bucket] = Math.max(sketch[bucket], leadingZeros);
}
```

---

## Pseudo code

- Estimation

```java
double z = 0;
for (byte leadingZeros : sketch) {
    z += 1.0 / Math.pow(2, leadingZeros);
}

return alpha * m * m / z;
```

---

## HLL is a random variable

- HLLは任意のデータセットに対して前述のアルゴリズムで値を定める確率変数である
- この確率変数の期待値がcardinality `n`に等しいということ

---

### Accuracy

- bucket数を`m`としたとき、標準誤差 = `1.04/√m`
  - Flajolet et al., 2007.
  - ここでいう標準誤差 := 標準偏差を真のcardinalityで割って得た相対誤差
  - Redisはdefaultだと16384 bucketなので`1.04/√16384 = 0.008125`
  - => 誤差0.81%

---

## Accuracy

- 「どんな入力に対しても誤差が0.81%以内」という意味ではない
- 例: Redis 4.0.9で以下の入力は相対誤差-90%となる

```
$ redis-cli PFADD foo 98567648 19857710 293736832 \                                                                                                                                                  master
                      275337325 304058906 154945851 \
                      227134849 290132289 168593923 \
                      279957693
$ redis-cli PFCOUNT foo
(integer) 1
```

---

## What causes high error ?

- ハッシュ値の衝突
  - だが一般的にはハッシュのpre imageを求めるのは困難
- 同じbucketへの振り分けられて、かつ同じ数先頭の0 bitが連続している場合
  - 98567648, 19857710,...はすべて、Redisのhash関数で`bucket 0 && 先頭の連続する0の数 = 0`になる

---

## Streaming update

- HLL sketch全体を再構築せずに、要素を追加できる

```java
public void add(String element) {
    long hash = calcHash(element);
    int bucket = calcBucket(hash);
    byte leadingZeros = calcLeadingZeros(hash);
    
    sketch[bucket] = Math.max(sketch[bucket], leadingZeros);
}
```

---

## Merge two sketches

- ２つのHLL sketchはloss lessでmergeできる
  - (bucket数やhash関数は同じ前提)

![i008](img/i008.png)

---

## Merge two sketches

```java
public byte[] merge(byte[] sketch1, byte[] sketch2) {
    int m = sketch1.length;
    byte[] merged = new byte[m];
    
    for (int i = 0; i < m; i++) {
        merged[i] = Math.max(sketch1[i], sketch2[i]);
    }
    
    return merged;
}
```

---

## Easy to parallelize

- mergeがloss lessなので、大量のdatasetのHLL sketch構築は容易に並列化できる

![i009](img/i009.png)

---

## Sketch & Estimation

- HLL sketchはFlajolet et al., 2007が初出ではない
  - Durand & Flajolet., 2003. Loglog Counting of Large Cardinalities
- sketch構築はHLLと同じだが、足し合わせ方が違う

![i010](img/i010.png)

---

## LogLog-Beta

- Jason Qin et al., 2016. LogLog-Beta and More: A New Algorithm for Cardinality Estimation Based on LogLog Counting

![i011](img/i011.png)

---

## Otmar Ertl method

- Otmar Ertl, 2017. New cardinality estimation algorithms for HyperLogLog sketches
- Redis 5.0.5（現時点のlatest）で採用されているestimation
- これも、sketchはオリジナルのHLLと同じ

---

# HLL on Redis

---

## Using HLL on Redis

- RedisのHLL関連commandは`PFxxx`

```bash
$ for i in `seq 1000`; do
  redis-cli PFADD foo $i > /dev/null
done

$ redis-cli PFCOUNT foo
(integer) 1001
```

---

## Merge

```bash
$ for i in `seq 2000 2500`; do
  redis-cli PFADD bar $i > /dev/null
done

$ redis-cli PFMERGE merged foo bar
$ redis-cli PFCOUNT merged
(integer) 1506
```

---

## Get sketch

```bash
$ reids-cli PFADD baz 1

$ redis-cli GET baz
"HYLL\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x80]f\x80b\x97"
```

---

## Performance

---

## History

- Redis 2.8.9で導入
  - https://raw.githubusercontent.com/antirez/redis/2.8.9/00-RELEASENOTES
- Linear CountingとHLLを併用
  - cardinalityが小さいとき、HLLは精度が低い
  - Flajolet et al., 2007でも併用について言及

---

## Redis v4.0.0

- https://raw.githubusercontent.com/antirez/redis/4.0.0/00-RELEASENOTES
- estimationがLogLog-Betaに切り替わった
- Linear Countingを使わなくなった

---

## Redis v5.0.0

- https://raw.githubusercontent.com/antirez/redis/5.0.0/00-RELEASENOTES
- estimationがOtmar Ertl, 2017. による方式に切り替わった

---

## HLL Representation

---

# HyperMinHash

---

## Intersection cardinality

- HLLを使うことで省メモリにcardinalityを保持・計算でき、unionも取れることがわかった
- unionが取れるならintersectionも欲しくなる

---

### Hoge Analytics 2.0

![i005](img/i005.png)

---

## Difficulty

- 以下のディメンションを任意に組み合わせたい
  - 地域 (100種類)
  - URL (100ページ)
  - OS (10種類)
  - 流入キーワード (10000種類)
  - 流入元サイト (1000サイト)
- 1兆通り
  - Redisの場合、sketchは12KB => 計12PB

---

## Probabilistic approach again

- **MinHash**
  - Jaccard Indexを近似する確率的アルゴリズム

---

## MinHash and HyperLogLog

- 

---

## HyperMinHash

---

## Conclusion

- HyperLogLog sketchは面白い
- 確率的アルゴリズムは面白い
