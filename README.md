# 3D TIFF Porosity Analyzer

A basic Python workflow for calculating porosity, directional porosity profiles, and pore size distribution from 3D TIFF image stacks of porous materials.

このリポジトリは、3D SEM / FIB-SEMなどで取得した3D TIFFスタック画像から、セパレータや多孔質材料の基本的な空隙率と孔径分布を計算するためのPython Notebookです。

最初の目的は、複雑なtortuosity解析や孔ネットワーク解析の前段階として、以下を行うことです。

- 3D TIFFの読み込み
- 画像の確認
- ノイズ除去
- 二値化
- 全体空隙率の計算
- X/Y/Z方向の空隙率プロファイル
- c-PSDを参考にした孔径分布解析
- 最大球とcentroid pathに基づくstrict c-PSD近似
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
| 孔径分布 | 距離変換、local thickness、最大球法により孔径分布を評価する |
| CSV保存 | 解析結果を後処理・比較しやすい形式で保存する |

## Installation

Python 3.9以上を推奨します。

```bash
pip install numpy matplotlib tifffile scipy scikit-image pandas
```

または、同梱の `requirements.txt` を使ってインストールします。

```bash
pip install -r requirements.txt
```

`porespy` はoptionalです。インストールされていない場合でも、scipyの距離変換による孔径分布解析は実行できます。

## Recommended directory structure

```text
repository/
  README.md
  requirements.txt
  LICENSE
  notebooks/
    01_basic_porosity_analysis.ipynb
    02_cpsd_pore_size_distribution.ipynb
  data/
    .gitkeep
  outputs/
    .gitkeep
```

## Input assumptions

このNotebookでは、入力画像が以下の形式であることを前提にしています。

- 入力は `.tif` または `.tiff` の3Dスタック画像
- 配列のshapeは基本的に `(Z, Y, X)`
- 孔相と固体相に輝度差がある
- 最初は単純なOtsu二値化で孔相と固体相を分離する
- 孔が暗い画像をデフォルトとする

SEM画像では、一般に孔が暗く、固体が明るく見えることが多いため、デフォルトでは `PORE_IS_DARK = True` としています。  
ただし、画像条件によっては逆になる場合があるため、必ずpore maskを目視確認してください。

## Basic porosity analysis

解析Notebook: `notebooks/01_basic_porosity_analysis.ipynb`

### Usage

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

### Workflow

1. 3D TIFFを読み込む
2. 中央スライスとヒストグラムを確認する
3. 必要に応じてノイズ除去を行う
4. しきい値により孔相と固体相へ二値化する
5. 全体空隙率を計算する
6. X/Y/Z方向の空隙率プロファイルを計算する
7. 図とCSVを保存する

### Outputs

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

#### `porosity_summary.csv`

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

### Theory

空隙率（porosity）は、材料全体の体積に対して、孔相が占める体積の割合です。  
3D TIFF画像では、1つ1つのvoxelを小さな体積要素とみなし、孔相に分類されたvoxel数を全voxel数で割ることで空隙率を計算します。

```text
porosity = pore volume / total volume
         = number of pore voxels / number of total voxels
```

Notebook内では、二値化後の `pore_mask` を使って以下のように計算しています。

```python
porosity = np.mean(pore_mask)
```

`pore_mask` は孔相を `True`、固体相を `False` とする3D配列です。  
Pythonでは `True = 1`、`False = 0` として平均できるため、`np.mean(pore_mask)` は全voxelに対する孔voxelの割合になります。

#### 二値化と孔相の定義

SEM / FIB-SEM画像では、元画像は連続的な輝度値を持っています。  
空隙率を計算するには、まず画像を孔相と固体相に分ける必要があります。

孔が暗い画像では、しきい値 `T` より小さい輝度を孔相とします。

```text
pore = image < T
solid = image >= T
```

孔が明るい画像では、しきい値 `T` より大きい輝度を孔相とします。

```text
pore = image > T
solid = image <= T
```

このNotebookでは、デフォルトではOtsu法によりしきい値を自動決定します。  
Otsu法は、画像ヒストグラムを2つのクラスに分けたときに、クラス間の分離が大きくなるしきい値を選ぶ方法です。孔相と固体相の輝度分布が十分に分かれている画像では有効ですが、輝度ムラ、ノイズ、カーテニング、再付着がある場合は、必ず中央スライス、ヒストグラム、pore maskを見て妥当性を確認してください。

#### voxel sizeと空隙率

全体空隙率そのものは、voxel sizeが一定であれば、voxel数の比から計算できます。  
各voxelの体積を `V_voxel` とすると、孔体積と全体体積は以下のように表せます。

```text
pore volume  = N_pore  * V_voxel
total volume = N_total * V_voxel
porosity     = (N_pore * V_voxel) / (N_total * V_voxel)
             = N_pore / N_total
```

したがって、空隙率は無次元量であり、通常は%で表示します。  
一方で、孔径、実体積、比表面積などを物理単位で評価する場合には、正しいvoxel sizeが必要です。

#### 方向別空隙率プロファイル

全体空隙率は、観察視野全体を1つの値にまとめたものです。  
しかし、多孔質材料では厚み方向や面内方向に構造ムラがあることがあります。そこで、このNotebookではX/Y/Z方向ごとに断面単位の空隙率を計算します。

Z方向プロファイルは、各Zスライスにおける孔voxel割合です。

```text
porosity_z[z] = number of pore voxels in slice z / number of voxels in slice z
```

Notebook内では以下に対応します。

```python
porosity_z = pore_mask.mean(axis=(1, 2))
```

同様に、Y方向、X方向のプロファイルは以下のように計算します。

```python
porosity_y = pore_mask.mean(axis=(0, 2))
porosity_x = pore_mask.mean(axis=(0, 1))
```

### How to interpret the results

全体空隙率は、観察した3D視野内で孔相が占める体積割合です。  
ただし、FIB-SEMの視野は小さいことが多いため、材料全体の代表値とは限りません。

Z方向の空隙率が大きく変動する場合、以下の可能性があります。

- 多層構造
- 塗工層と基材の違い
- 厚み方向の詰まり
- FIB加工影響
- 画像輝度ドリフト
- 局所的な構造ムラ

X/Y方向で傾向が異なる場合、面内の異方性や加工方向の影響が示唆されます。

## c-PSD / pore size distribution analysis

解析Notebook: `notebooks/02_cpsd_pore_size_distribution.ipynb`

このリポジトリでは、基本的な空隙率解析に加えて、3D孔径分布解析を追加しています。  
参考論文では、c-PSD法として、3D空隙内に入る球の半径を評価し、半径ごとに孔体積分率をプロットしています。本リポジトリでは、距離変換ベースの孔径分布に加えて、最大球とcentroid pathに基づくstrict c-PSD近似も実装しています。

### Workflow

1. 3D TIFFを読み込む
2. ノイズ除去
3. 二値化して孔相を抽出する
4. 孔相に対して3D距離変換を行う
5. 各孔ボクセルの局所半径を取得する
6. 半径分布、直径分布、累積孔径分布を計算する
7. 最大球候補を抽出し、大きい球から孔相を被覆する
8. 最大球中心間のcentroid pathを計算する
9. CSVと図を保存する

### Outputs

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
  strict_cpsd_maximal_spheres.csv
  strict_cpsd_sphere_coverage.csv
  strict_cpsd_distribution.csv
  strict_cpsd_centroid_paths.csv
  strict_cpsd_summary.csv
  strict_cpsd_radius_distribution.png
  strict_cpsd_cumulative_distribution.png
  strict_cpsd_assigned_radius_map.png
```

### Theory

c-PSD（continuous pore size distribution）は、3D画像中の孔相に対して「その位置を含む、またはその近傍に入る球の大きさ」を評価し、孔径分布として表す考え方です。  
2D断面だけから孔径を推定するのではなく、3D空間内で孔がどの程度の大きさの球を収容できるかを見るため、複雑な多孔質構造の評価に向いています。

本Notebookでは、まず3D画像を孔相と固体相に二値化します。

```text
pore voxel  = 1
solid voxel = 0
```

#### 距離変換ベースの孔径評価

二値化した孔相に対して、3Dユークリッド距離変換（Euclidean distance transform, EDT）を行います。  
距離変換では、各孔ボクセルについて「最も近い固体相または孔/固体界面までの距離」を計算します。

```text
r(x, y, z) = distance from pore voxel (x, y, z) to nearest solid/interface
```

この `r` は、その孔ボクセルを中心としたときに固体相にぶつからず入る局所的な球半径と解釈できます。  
そのため、孔径は以下のように表します。

```text
pore radius   = r
pore diameter = 2r
```

実装では `scipy.ndimage.distance_transform_edt` を用い、voxel sizeを反映するために以下のsamplingを指定します。

```python
sampling = (VOXEL_SIZE_Z, VOXEL_SIZE_Y, VOXEL_SIZE_X)
```

これにより、X/Y/Z方向のvoxel sizeが異なる場合でも、距離はµm単位で計算されます。  
ただし、voxel異方性が大きい場合は、孔径分布の形に影響するため、等方voxelへのリサンプリングを検討してください。

#### 孔径分布の作り方

距離変換で得られたすべての孔ボクセルの局所半径 `r` を集め、指定したbin幅でヒストグラム化します。  
各binに入った孔ボクセル数を、全孔ボクセル数で割ることで体積分率として表します。

```text
volume_fraction_i = voxel_count_i / total_pore_voxel_count
```

ここで得られる分布は、「孔相体積のうち、局所半径がその範囲にある部分がどれだけあるか」を示します。  
大きな半径側の割合が高いほど、広い孔空間が多いことを意味します。一方、小さな半径側の割合が高い場合は、細い孔、狭い隙間、またはしきい値・ノイズの影響を強く受けている可能性があります。

累積孔径分布は、半径binごとの体積分率を小さい方から積算したものです。

```text
cumulative_fraction_i = sum(volume_fraction_0 ... volume_fraction_i)
```

この値は、全孔体積のうち、ある孔半径以下の領域がどれだけを占めるかを表します。

#### 最大球とcentroid pathに基づくstrict c-PSD近似

論文で用いられるc-PSD法では、孔空間に入る最大球、球中心の経路、孔のくびれ、constrictivityなどを考慮して、孔ネットワークの幾何をより厳密に評価します。

本Notebookでは、距離変換マップの局所最大を最大内接球の中心候補として抽出します。  
次に、半径の大きい最大球から順に孔相を被覆し、各最大球が新たに代表した孔voxel数を半径binへ集計します。

```text
maximal sphere center = local maximum of EDT
covered pore volume   = pore voxels newly covered by each maximal sphere
strict c-PSD bin      = covered pore volume grouped by maximal sphere radius
```

さらに、各最大球中心を、より大きい近傍の最大球中心へ接続し、centroid pathを作成します。  
その経路上の距離変換値の最小値を、孔のくびれやボトルネックに近い幾何指標として保存します。

```text
centroid_path_min_radius = minimum EDT value along the center-to-center path
```

ただし、このstrict c-PSDは論文実装の完全再現ではなく、再現可能な近似実装です。  
接続規則、最大球の選別、画像の間引き率、二値化条件によって結果は変わります。constrictivityそのものやtortuosityは、まだ直接計算していません。

`porespy` が利用できる場合は、`local_thickness` による解析も追加で実行できます。  
local thicknessは、各孔ボクセルを含む最大内接球に近い考え方で厚みを評価するため、単純な距離変換よりもc-PSDの最大球的な解釈に近くなります。

## 画像取得時の注意

解析精度は、元画像の品質に大きく依存します。  
3D SEM / FIB-SEM画像解析では、Pythonで計算する前の画像取得条件が非常に重要です。特に、voxel size、Z方向のslice pitch、視野サイズ、コントラスト、ドリフト、カーテニングの有無が、空隙率や孔径分布に大きく影響します。

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

## Limitations

このNotebookは初歩的な空隙率解析と孔径分布解析を目的としています。  
以下の点に注意してください。

- Otsu二値化が常に正しいとは限らない
- しきい値によって空隙率と孔径分布は大きく変わる
- FIB-SEM画像ではカーテニングや再付着が解析結果に影響する
- voxel sizeが未設定でも空隙率自体は計算できるが、孔径や体積の物理単位には影響する
- 観察視野が小さい場合、材料全体の代表値とは限らない
- 画像解像度以下の孔は検出できない
- 異方性voxelの場合、孔径解析には等方voxelへのリサンプリングを検討する
- strict c-PSDは最大球とcentroid pathに基づく近似実装であり、論文コードの完全再現ではない
- 連結性、tortuosity、constrictivityはまだ直接計算しない

したがって、数値だけを採用せず、元画像、ヒストグラム、pore mask、方向別プロファイル、孔径分布図をセットで確認することが重要です。

## Future work

今後、以下の解析へ拡張する予定です。

- constrictivity analysis
- pore throat / bottleneck analysis
- pore network model extraction
- tortuosity and effective diffusivity calculation
- connected porosity
- isolated pore removal
- flux distribution map
- representative volume element analysis
- comparison between multiple samples

## License

MIT License
