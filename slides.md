## HyperLogLog sketchは面白い

### builderscon tokyo 2019
#### Haruki Okada (@ocadaruma)

---

## Introduction

- リアルタイムなアクセス解析システムを作るとする

---

## Hoge Analytics

- TODO: 図

---

## Problem

- ユニークユーザー数をどうやって出すか？

---

## Probabilistic Way

---

## Agenda

1. How HLL works
2. HLL on Redis and BigQuery
3. Deep dive into Redis HLL
4. HyperMinHash

---

## How HLL works

---

## Intuitive explanation

- 64bit intを一様ランダムに選ぶとする

---

## Intuitive explanation

- 2^n個選んだ

---

## What does "LogLog" means

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