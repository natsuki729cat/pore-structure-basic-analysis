# 3D TIFF Porosity Analyzer

A basic Python workflow for calculating porosity and directional porosity profiles from 3D TIFF image stacks of porous materials.

このリポジトリは、3D SEM / FIB-SEMなどで取得した3D TIFFスタック画像から、セパレータや多孔質材料の基本的な空隙率を計算するためのPython Notebookです。

最初の目的は、複雑な孔径分布やtortuosity解析の前段階として、以下を行うことです。

- 3D tifの読み込み
- 画像の確認
- ノイズ除去
- 二値化
- 全体空隙率の計算
- X/Y/Z方向の空隙率プロファイル
- 結果画像とCSV保存

## この解析でできること

| 解析項目 | 内容 |
|---|---|
| 3D TIFF読み込み | SEM / FIB-SEM由来の3Dスタック画像を読み込む |
| 中央スライス確認 | 画像品質、孔と固体のコントラストを確認する |
| ヒストグラム確認 | 二値化しきい値の妥当性を確認する |
| ノイズ除去 | メディアンフィルタなどで孤立ノイズを低減する |
| 二値化 | 孔相と固体相を分離する |
| 全体空隙率 | 観察視野全体の空隙体積率を計算する |
| Z方向空隙率 | 厚み方向の層差、詰まり、ムラを確認する |
| X/Y方向空隙率 | 面内方向のムラや異方性を確認する |
| CSV保存 | 解析結果を後処理・比較しやすい形式で保存する |

## Workflow

1. Sample preparation
2. 3D SEM / FIB-SEM imaging
3. Export as 3D TIFF stack
4. Load TIFF in Python
5. Check image quality
6. Denoising
7. Thresholding / segmentation
8. Porosity calculation
9. Directional porosity profile
10. Export figures and CSV files

3D SEM / FIB-SEM画像解析では、Pythonで計算する前の画像取得条件が非常に重要です。  
特に、voxel size、Z方向のslice pitch、視野サイズ、コントラスト、ドリフト、カーテニングの有無が、空隙率計算に大きく影響します。

## 画像取得時の注意

解析精度は、元画像の品質に大きく依存します。

### 確認すべきメタデータ

- pixel size in X
- pixel size in Y
- slice thickness / Z pitch
- voxel size
- image bit depth
- number of slices
- imaging direction
- SEM voltage / current
- detector condition
- FIB milling step
- sample orientation

### 画像品質で確認すべきこと

- 孔相と固体相のコントラストが十分か
- Z方向にドリフトしていないか
- FIBカーテニングが強く出ていないか
- チャージアップによる輝度ムラがないか
- 再付着物が孔を埋めていないか
- 視野が材料を代表しているか
- 表面近傍だけでなく内部構造が見えているか

## Assumptions

このNotebookでは、入力画像が以下の形式であることを前提にしています。

- 入力は `.tif` または `.tiff` の3Dスタック画像
- 配列のshapeは基本的に `(Z, Y, X)`
- 孔相と固体相に輝度差がある
- 最初は単純なOtsu二値化で孔相と固体相を分離する
- 孔が暗い画像をデフォルトとする

SEM画像では、一般に孔が暗く、固体が明るく見えることが多いため、デフォルトでは `PORE_IS_DARK = True` としています。  
ただし、画像条件によっては逆になる場合があるため、必ずpore maskを目視確認してください。

## Installation

Python 3.9以上を推奨します。

```bash
pip install numpy matplotlib tifffile scipy scikit-image pandas
```

または、同梱の `requirements.txt` を使ってインストールします。

```bash
pip install -r requirements.txt
```

## Usage

1. 解析したい3D TIFFファイルをリポジトリ内、または任意のフォルダに置きます。
2. Notebookを開きます。
3. 最初のパラメータセルで `INPUT_TIF` を変更します。
4. 孔が暗い画像の場合は `PORE_IS_DARK = True`、孔が明るい画像の場合は `False` にします。
5. Notebookを上から順番に実行します。
6. 出力フォルダに画像とCSVが保存されます。

パラメータ例:

```python
INPUT_TIF = "data/sample_3d_sem.tif"
OUTPUT_DIR = "outputs/porosity_output"
PORE_IS_DARK = True
USE_MEDIAN_FILTER = True
MEDIAN_SIZE = 2
USE_MANUAL_THRESHOLD = False
MANUAL_THRESHOLD = 128
```

## Outputs

Notebookを実行すると、以下のようなファイルが出力されます。

```text
porosity_output/
  center_slice_original.png
  center_slice_filtered.png
  center_slice_pore_mask.png
  porosity_z_profile.png
  porosity_xy_profile.png
  porosity_summary.csv
  porosity_z_profile.csv
  porosity_x_profile.csv
  porosity_y_profile.csv
```

### `porosity_summary.csv`

| column | description |
|---|---|
| input_file | 入力ファイル名 |
| shape_z | Z方向のslice数 |
| shape_y | Y方向pixel数 |
| shape_x | X方向pixel数 |
| dtype | 画像データ型 |
| threshold | 二値化しきい値 |
| pore_is_dark | 孔を暗い相として扱ったか |
| porosity | 空隙率 |
| porosity_percent | 空隙率[%] |
| solid_fraction | 固体率 |
| solid_fraction_percent | 固体率[%] |

## How to interpret the results

### 全体空隙率

全体空隙率は、観察した3D視野内で孔相が占める体積割合です。  
ただし、FIB-SEMの視野は小さいことが多いため、材料全体の代表値とは限りません。

### Z方向空隙率プロファイル

Z方向の空隙率が大きく変動する場合、以下の可能性があります。

- 多層構造
- 塗工層と基材の違い
- 厚み方向の詰まり
- FIB加工影響
- 画像輝度ドリフト
- 局所的な構造ムラ

### X/Y方向空隙率プロファイル

X/Y方向で傾向が異なる場合、面内の異方性や加工方向の影響が示唆されます。

## Limitations

このNotebookは初歩的な空隙率解析を目的としています。  
以下の点に注意してください。

- Otsu二値化が常に正しいとは限らない
- しきい値によって空隙率は大きく変わる
- FIB-SEM画像ではカーテニングや再付着が解析結果に影響する
- voxel sizeが未設定でも空隙率自体は計算できるが、孔径や体積の物理単位には影響する
- 観察視野が小さい場合、材料全体の代表値とは限らない
- 孔径分布は近似実装であり、連結性、tortuosityはまだ計算しない


## c-PSD / pore size distribution analysis

このリポジトリでは、基本的な空隙率解析に加えて、3D孔径分布解析を追加しています。
参考論文では、c-PSD法として、3D空隙内に入る球の半径を評価し、半径ごとに孔体積分率をプロットしています。  
本リポジトリでは、その考え方を参考に、まずPythonで扱いやすい距離変換ベースの孔径分布を実装しています。

解析Notebook: `notebooks/02_cpsd_pore_size_distribution.ipynb`

### 解析の流れ

1. 3D TIFFを読み込む
2. ノイズ除去
3. 二値化して孔相を抽出する
4. 孔相に対して3D距離変換を行う
5. 各孔ボクセルの局所半径を取得する
6. 半径分布、直径分布、累積孔径分布を計算する
7. CSVと図を保存する

### 出力ファイル

```text
cpsd_output/
  center_slice_original.png
  center_slice_pore_mask.png
  center_slice_local_radius_map.png
  pore_radius_distribution.png
  pore_diameter_distribution.png
  cumulative_pore_size_distribution.png
  distance_transform_psd.csv
  cpsd_summary.csv
  porespy_local_thickness_distribution.csv      # porespy使用時のみ
  porespy_local_thickness_distribution.png      # porespy使用時のみ
  porespy_local_thickness_map.png               # porespy使用時のみ
```

### 注意点

- 本実装は論文のc-PSD法を参考にした近似実装です
- distance transformで得られる値は、各孔ボクセルから界面までの距離です
- 厳密な最大球法、centroid path法、constrictivity評価とは異なります
- voxel size、二値化しきい値、ノイズ除去条件に強く依存します
- 画像解像度以下の孔は検出できません
- FIB-SEM画像ではカーテニング、再付着、チャージアップの影響を受けます
- 異方性voxelの場合、孔径解析には等方voxelへのリサンプリングを検討してください
- `porespy` が利用できる環境では、local thickness / maximal sphereに近い考え方の追加解析も実行できます
- `porespy` が利用できない環境でも、Notebook全体は停止せず、scipyの距離変換ベース解析だけで動作します

## Future work

今後、以下の解析へ拡張する予定です。

- pore size distribution
- strict c-PSD implementation based on maximal spheres and centroid paths
- constrictivity analysis
- pore throat / bottleneck analysis
- pore network model extraction
- tortuosity and effective diffusivity calculation
- connected porosity
- isolated pore removal
- flux distribution map
- representative volume element analysis
- comparison between multiple samples

## Recommended directory structure

```text
repository/
  README.md
  requirements.txt
  LICENSE
  notebooks/
    01_basic_porosity_analysis.ipynb
  data/
    .gitkeep
  outputs/
    .gitkeep
```

## License

MIT License
