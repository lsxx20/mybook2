---
title: 資料處理／地圖繪製
---
## 資料處理／地圖繪製

###### 3D 模型掃描建置（Luma AI）

* 於手機中下載 Luma 3D Capture

* 依照軟體介面提示，環繞拍攝欲建置的物件

* 上傳至 Luma 雲端 （上傳時間大約 1 小時，視拍攝物件大小）

* 於電腦 Luma AI - Interactive Scenes 網頁登入拍攝時所使用的帳號，檢視個人先前上傳的影像  
https://lumalabs.ai/interactive-scenes

* 點選分享功能，內有可複製的 iframe code ，可嵌入於 jupyter book 中完成 3D 模型呈現


###### 日治時期 Kinaji／Marikowan代表聚落位置圖 

* 準備點位資料 (QGIS)

  建立新的 gpkg ，根據整理文獻所得，點出當時聚落的大致位置。屬性表欄位包括聚落名稱、所屬社群

<figure style="margin-left:2em; text-align:left;">
  <img src="_static/tribe2.png" alt="logo" width="300" style="display:inline-block; margin:0;">
  <figcaption style="font-size:0.9em; color:gray; text-align:left;">
    點位示意圖　圖片來源：筆者照片
  </figcaption>
</figure>

<figure style="margin-left:2em; text-align:left;">
  <img src="_static/tribe3.png" alt="logo" width="300" style="display:inline-block; margin:0;">
  <figcaption style="font-size:0.9em; color:gray; text-align:left;">
    屬性表示意圖　圖片來源：筆者照片
  </figcaption>
</figure>


* 匯入套件
```python
from pathlib import Path
import re
import geopandas as gpd
import folium
import pandas as pd
```
* 輸入／輸出路徑
```python
DATA = Path("日治原社.gpkg")  
OUT  = Path("docs/_static/maps/kinaji_marikowan_points.html")
```
* 讀取全部圖層，設定座標為 EPSG:4326
```python
layers = gpd.io.file.fiona.listlayers(str(DATA))
gdfs = []
for lyr in layers:
    g = gpd.read_file(DATA, layer=lyr)
    if g.crs and g.crs.to_epsg() != 4326:
        g = g.to_crs(4326)
    g["__layer__"] = lyr
    gdfs.append(g)
gdf = gpd.GeoDataFrame(pd.concat(gdfs, ignore_index=True), crs="EPSG:4326")
```
* 給定欄位、中文族語對照資料
```python
col_comm = "社群"
col_zh = "中文名"
col_at = "族語"
name_override = {
    "司馬庫斯":"Smangus", "斯馬庫斯":"Smangus",
    "宇老":"Uraw", "抬耀":"Tayax", "泰崗":"Thyakan",
    "田埔":"Tbahu", "石磊":"Quri", "錦路":"Kin lwan",
    "養老":"Yuluw", "馬美":"Mami", "馬里光":"Llyung",
    "拉號":"Rahaw", "鳥嘴":"Cyocuy", "鎮西堡":"Cinsbu",
}
```
* 組合標籤顯示名稱
```python
def to_label(row):
    zh = str(row[col_zh]).strip() if col_zh else ""
    at = str(row[col_at]).strip() if (col_at and pd.notna(row[col_at])) else ""
    if not at and zh in name_override:
        at = name_override[zh]
    if not at:
        at = zh
    return f"{at}（{zh}）" if zh else at

gdf["label_fmt"] = gdf.apply(to_label, axis=1)
```
* 根據 gpkg 的點位屬性資料做社群分類
```python
def grp(c):
    s = str(c).upper()
    if s.startswith("K"): return "Kinaji（基那吉）"
    if s.startswith("M"): return "Marikowan（馬里光）"
    return "未分類"

gdf["group_name"] = gdf[col_comm].apply(grp) if col_comm else "未分類"
```
* 繪製地圖、設定底圖
```python
center = [gdf.geometry.y.mean(), gdf.geometry.x.mean()]
m = folium.Map(location=center, zoom_start=10, control_scale=True, tiles=None)

folium.TileLayer("OpenStreetMap", name="OSM 現代地圖", overlay=False, control=True, show=True).add_to(m)
folium.TileLayer(
    tiles="https://gis.sinica.edu.tw/tileserver/file-exists.php?img=JM300K_1924-png-{z}-{x}-{y}",
    name="1924 日治臺灣全圖（XYZ）",
    attr="GIS Center, RCHSS, Academia Sinica — non-commercial use",
    overlay=False, control=True, show=False
).add_to(m)
```
* 標點樣式設定
```python
COLOR_K = "#1f77b4"  # 藍：Kinaji
COLOR_M = "#d62728"  # 紅：Marikowan
COLOR_U = "#7f7f7f"

bounds = []
for _, r in gdf.iterrows():
    if r.geometry is None:
        continue
    lat, lon = float(r.geometry.y), float(r.geometry.x)
    grp_name = r["group_name"]
    if grp_name.startswith("Kinaji"):
        fill = COLOR_K
    elif grp_name.startswith("Marikowan"):
        fill = COLOR_M
    else:
        fill = COLOR_U
    folium.CircleMarker(
        location=[lat, lon],
        radius=7,
        color="white",
        weight=1.5,
        fill=True, fill_color=fill, fill_opacity=0.95,
        tooltip=f"{r['label_fmt']} · {grp_name}",
        popup=f"<b>{r['label_fmt']}</b><br/>{grp_name}"
    ).add_to(m)
    bounds.append([lat, lon])
```
* 將圖例插到地圖，並且加入LayerControl功能，讓使用者可以切換底圖
```html
<div style="...">
  <div style="font-weight:600;margin-bottom:6px;">代表社群</div>
  <div style="display:flex;align-items:center;margin:3px 0;">
    <span style="...background:#1f77b4..."></span>
    <span style="font-size:13px;">Kinaji（基那吉）</span>
  </div>
  <div style="display:flex;align-items:center;margin:3px 0;">
    <span style="...background:#d62728..."></span>
    <span style="font-size:13px;">Marikowan（馬里光）</span>
  </div>
</div>
"""
m.get_root().html.add_child(folium.Element(legend_html))
folium.LayerControl(position="topright", collapsed=False).add_to(m)
```
* 調整地圖縮放比例，剛好框住所有點。建立輸出資料夾，存成 HTML，並在終端機顯示路徑。
```python
if bounds:
    m.fit_bounds(bounds)
    OUT.parent.mkdir(parents=True, exist_ok=True)
m.save(OUT)
print("Saved:", OUT)
```

###### 司馬庫斯遷徙地圖 ######

* 準備點位資料 (QGIS)

  建立新的 gpkg ，根據整理文獻所得，加入當時聚落的大致位置與名稱


* 匯入套件
```python
from pathlib import Path
import re
import geopandas as gpd
import folium
```
* 輸入／輸出路徑
```python
DATA = Path("tribe.gpkg")  # 點位　gpkg
OUT  = Path("docs/_static/maps/smangus_immigration.html")
```
* 讀取圖層，設定座標為 EPSG:4326
```python
all_layers = gpd.io.file.fiona.listlayers(str(DATA))
gdfs = []
for lyr in all_layers:
    g = gpd.read_file(DATA, layer=lyr)
    if g.crs and g.crs.to_epsg() != 4326:
        g = g.to_crs(4326)
    g["__layer__"] = lyr
    gdfs.append(g)
gdf = gpd.GeoDataFrame(pd.concat(gdfs, ignore_index=True), crs="EPSG:4326")
```
* 幫每一筆資料的所有文字欄位串在一起，方便用關鍵字篩選
```python
def row_text(r):
    txts = []
    for c in gdf.columns:
        if c == gdf.geometry.name: 
            continue
        v = r.get(c, None)
        if isinstance(v, str):
            txts.append(v)
    return " ".join(txts)

gdf["__text__"] = gdf.apply(row_text, axis=1)
```
* 建立一個空列表，用來存每個遷徙階段的資訊。並且標籤固定用「泰雅族語（中文）」的呈現格式
```python
stages = []
def add(idx, num, sub=None):
    if idx is None: 
        return
    pt = gdf.iloc[idx].geometry
    stages.append({
        "stage": num, 
        "part": sub, 
        "lat": float(pt.y), 
        "lon": float(pt.x)
    })

add(idx_1, 1); add(idx_2, 2); add(idx_3, 3)
add(idx_4, 4); add(idx_5a, 5, 1); add(idx_5b, 5, 2); add(idx_6, 6)

def label_of(stage, part=None):
    if stage == 1: return "Pinsbkan（瑞岩）"
    if stage == 2: return "Quri Sqabu（思源埡口）"
```
* 建立地圖、設定底圖
```python
center = [gdf.geometry.y.mean(), gdf.geometry.x.mean()]
m = folium.Map(location=center, zoom_start=10, control_scale=True, tiles=None)

folium.TileLayer("OpenStreetMap", name="OSM 現代地圖", overlay=False, control=True, show=True).add_to(m)
folium.TileLayer(
    tiles="https://gis.sinica.edu.tw/tileserver/file-exists.php?img=JM300K_1924-png-{z}-{x}-{y}",
    name="1924 日治臺灣全圖（XYZ）",
    attr="GIS Center, RCHSS, Academia Sinica — non-commercial use",
    overlay=False, control=True, show=False
).add_to(m)
```
* 設定點位標記樣式
```python
palette = ["#1f77b4","#ff7f0e","#2ca02c","#d62728","#9467bd","#8c564b","#e377c2","#7f7f7f","#bcbd22","#17becf"]

bounds = []
for r in stages:
    g = r["stage"]; p = r["part"]
    label_num = f"{g}-{p}" if p else f"{g}"
    color = palette[(g-1) % len(palette)]
    text = label_of(g, p)
    bounds.append([r["lat"], r["lon"]])
    html = f"""
    <div style="display:inline-block; padding:2px 8px; min-width:32px;
        background:{color}; border:2px solid #ffffff; color:#ffffff;
        border-radius:16px; font-weight:700; font-size:12px; line-height:20px;
        text-align:center;">{label_num}</div>"""
    folium.Marker(
        location=[r["lat"], r["lon"]],
        icon=folium.DivIcon(html=html),
        tooltip=f"{label_num}. {text}",
        popup=f"<b>{label_num}. {text}</b>"
    ).add_to(m)
```
* 整理圖例中每個階段的名稱
```python
items_by_stage = {}
for r in stages:
    key = r["stage"]                       
    items_by_stage.setdefault(key, [])
    disp = label_of(r["stage"], r["part"]) 
    num  = f"{r['stage']}-{r['part']}" if r["part"] else f"{r['stage']}"  
    s = f"{disp}（{num}）" if r["part"] else f"{disp}"
    if s not in items_by_stage[key]:
        items_by_stage[key].append(s)
```
* 轉換成 HTML 的圖例
```python
legend_rows = []
for s in sorted(items_by_stage):
    color = palette[(s-1) % len(palette)]          
    names = "、".join(items_by_stage[s])           
    legend_rows.append(
        f'<div style="display:flex;align-items:center;margin:3px 0;">'
        f'<span style="display:inline-block;width:18px;height:18px;border-radius:9px;background:{color};'
        f'border:1px solid #fff;margin-right:6px;"></span>'
        f'<span style="font-size:13px;">{s}. {names}</span></div>'
    )
```
* 將圖例插到地圖，並且加入LayerControl功能，讓使用者可以切換底圖
```html
legend_html = f"""
<div style="position: fixed; bottom: 16px; right: 16px; z-index: 9999;
 background: white; padding: 10px 12px; border: 1px solid #ccc;
 border-radius: 6px; box-shadow: 0 1px 4px rgba(0,0,0,.3);
 font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Noto Sans TC', sans-serif;">
  <div style="font-weight:600;margin-bottom:6px;">遷徙階段</div>
  {''.join(legend_rows)}
</div>
"""
m.get_root().html.add_child(folium.Element(legend_html))
folium.LayerControl(position="topright", collapsed=False).add_to(m)
```
* 調整地圖縮放比例，剛好框住所有點。建立輸出資料夾，存成 HTML，並在終端機顯示路徑。
```python
if bounds: 
    m.fit_bounds(bounds)             
OUT.parent.mkdir(parents=True, exist_ok=True)
m.save(OUT)                        
print("Saved:", OUT)
```
