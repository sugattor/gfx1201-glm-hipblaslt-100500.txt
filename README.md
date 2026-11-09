# gfx1201 GLM hipBLASLt tuning file (hipBLASLt 1.5.0 / 100500)

Project-Maya で使用する、prefill GEMM 用の hipBLASLt tuning file です。

- 対象 GPU：AMD Radeon AI PRO R9700（`gfx1201`）
- 対象 hipBLASLt：1.5.0（バージョン値 `100500`）
- 対象モデル：GLM 系（`--glm-pack` を使用する構成）
- ファイル：`gfx1201-glm-hipblaslt-100500.txt`（55 行）

## 背景

Project-Maya は、実行時の hipBLASLt のバージョンとデバイスアーキテクチャから、tuning file の名前を決めます。

```text
tools/hip/<arch>-glm-hipblaslt-<version>.txt
```

Project-Maya に同梱されている gfx1201 用の GLM tuning file は、`100202`（hipBLASLt 1.2.2）用だけです。hipBLASLt 1.5.0 で実行すると、次のメッセージが出ます。この場合、prefill GEMM は hipBLASEx で実行されます。

```text
prefill gemm: .../gfx1201-glm-hipblaslt-100500.txt: cannot open tuning file; using hipBLASEx
```

このファイルを使用すると、hipBLASLt のチューニング結果が適用されます。

## 動作環境

このファイルを作成し、動作を確認した環境は次のとおりです。

| 項目 | 内容 |
|---|---|
| OS | Ubuntu 26.04 |
| GPU | AMD Radeon AI PRO R9700（gfx1201） |
| ROCm | TheRock nightly Python wheel `rocm==10.3.0a20261108`（`rocm[libraries,devel,device-gfx1201]`） |
| hipBLASLt | 1.5.0（`libhipblaslt.so.1.5`） |

ROCm 10.1.0（hipBLASLt 1.4.1 / `100401`）では、このファイルは使用されません。

## 使用方法

1. このファイルを Project-Maya の `tools/hip/` に置きます。
2. hipBLASLt 1.5.0 を含む ROCm のライブラリディレクトリを、Project-Maya の `lib_dirs` に設定します。
3. Project-Maya を起動します。

次のログが出れば、ファイルは使用されています。

```text
prefill gemm: hipBLASLt tuning enabled (55 rows, gfx1201, version 100500)
```

読み込まれたライブラリは、次のコマンドで確認できます。

```bash
grep -E 'libhipblaslt|libamdhip64' /proc/$(pgrep -n -f 'build-hip/maya')/maps | awk '{print $6}' | sort -u
```

## ファイル形式

```text
# solution IDs are scoped to this hipBLASLt version and device architecture
STRATA_HIPBLASLT_TUNING_V1 gfx1201 100500
<dtype> <N> <K> <ldy> <T> <solution_id>
```

solution ID は、hipBLASLt のバージョンとアーキテクチャに固有です。別のバージョンや別の GPU には使用できません。

## 作成方法

1. Project-Maya の `tools/hip/tune_hipblaslt.cpp` を、hipBLASLt 1.5.0 の環境でビルドしました。
2. 既存の `gfx1201-glm-hipblaslt-100202.txt` から、GEMM の形状（dtype, N, K, ldy）だけを取り出しました。12 形状です。solution ID は使用していません。
3. 12 形状 × T = 64, 128, 256, 512, 1024 の 60 ケースをチューニングしました。
4. hipblasGemmEx に対する速度比が 1.02 未満の 5 行を削除しました。既存の `100202` 用ファイルと同じ基準です。

```bash
"$R"/bin/hipcc -O2 -std=c++17 tools/hip/tune_hipblaslt.cpp -o tune_hipblaslt \
  -I"$R"/include -L"$R"/lib -lhipblas -lhipblaslt -Wl,-rpath,"$R"/lib

./tune_hipblaslt --shapes-file glm-shapes-gfx1201.json --tokens 64,128,256,512,1024 \
  --tuning-out tools/hip/gfx1201-glm-hipblaslt-100500.txt
```

`$R` は、ROCm SDK のルートディレクトリです（`rocm-sdk path --root` の出力）。

## 測定結果

`tune_hipblaslt` の測定値です（GEMM 単体）。

| 項目 | 値 |
|---|---|
| 採用した行 | 55 / 60 |
| 速度比（hipblasGemmEx に対する比） | 1.029x 〜 3.26x |
| 採用した 55 行の合計時間 | hipblasGemmEx 13.72 ms → hipBLASLt 9.57 ms |

## 制限事項

- 対象の形状は、既存の `100202` 用ファイルにある 12 形状だけです。`100202` 用ファイルから除外されていた形状は、チューニングしていません。
- 上記の数値は GEMM 単体の値です。prefill 全体の速度向上は、これより小さくなります。
- 基準値の測定回数は少なく、ばらつきがあります。速度比が 1.1 倍未満の行は、誤差の範囲内である可能性があります。
- hipBLASLt 1.5.0 は、ROCm nightly ビルド `10.3.0a20261108` で確認しました。他のビルドでは、hipBLASLt のバージョン値が同じ `100500` でも solution ID が異なる可能性があります。

## ライセンス

MIT License です。詳細は `LICENSE` を参照してください。

このファイルは、[Project-Maya](https://github.com/mw00/project-maya)（MIT License）の `tools/hip/tune_hipblaslt.cpp` で生成しました。チューニング対象の形状は、Project-Maya の `tools/hip/gfx1201-glm-hipblaslt-100202.txt` から取り出しました。
