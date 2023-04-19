# map-okn-folium

https://m-erts.github.io/map-okn-folium/ 

## Cостав команды
1. Гепалов Даниил
2. Постникова Арина
3. Эрцеговац Мария

## Краткая информация о городе
Ярославль - город в России, административный центр Ярославской области. Основан в 1010 году князем Ярославом Мудрым. Расположен на реке Волге, на перекрестке двух федеральных трасс М8 и М10. Население города составляет около 608 тысяч человек. Ярославль является промышленным и культурным центром региона, здесь расположены множество заводов, предприятий и крупных учебных заведений. Город славится своим красивым историческим центром, включенным в список объектов Всемирного наследия ЮНЕСКО, и богатым архитектурным наследием.

## Источники данных 
1. okn_yaroslavl.geojson - [https://opendata.mkrf.ru/opendata/7705851331-egrkn/](url)
2. yaroslavl_boundaries.geojson - OpenStreetMap Overpass API
3. yaroslavl_buildings.geojson - BBBike OpenStreetMap

## Описание основных методов
1. Создание grid
2. Присоединение через sjoin атрибуты одного слоя к другому.
3. Создание веб-карты с помощью библиотеки folium.

Самое сложное в работе - следить за правильными проекциями.
В работе получилось всё.

## 1. Загрузка библиотек
```python
import pandas as pd
import geopandas as gpd
import folium
from shapely import geometry
```
## 2. Загрузка и подготовка файлов
```python
bounds = gpd.read_file('yaroslavl_boundaries.geojson')
data_okn_points = gpd.read_file('okn_yaroslavl.geojson')
data_buildings = gpd.read_file('yaroslavl_buildings.geojson')
```
Все слои приводим к проекции WGS 84 / UTM zone 37N - EPSG:32637
```python
bounds = bounds.to_crs("EPSG:32637")
data_okn_points = data_okn_points.to_crs("EPSG:32637")
data_buildings = data_buildings.to_crs("EPSG:32637")
```
### Очистка атрибутивных таблиц
```python
data_okn_points = data_okn_points[['FIELD1', 'Объект', 'Полный адрес', 'Категория историко-культурного значения', 'Вид объекта', 'Принадлежность к Юнеско', 'Особо ценный объект', 'дата создания', 'geometry']]
data_okn_points.head()

data_buildings = data_buildings.drop(columns=['name', 'type'])
data_buildings.head()
```
Фильтрация точек и домов по границам Ярославля
```python
data_okn_points = gpd.clip(data_okn_points, bounds)
data_buildings = gpd.clip(data_buildings, bounds)
```
### Присоединение информации из точек к зданиям
Проверяем проекция, она должна быть идентична у слоев.
```python
data_buildings.crs = data_okn_points.crs
data_okn_points.crs.name
```
```python
data_buildings = data_buildings.sjoin(data_okn_points, how="left")
data_buildings.head()
data_okn_buildings = data_buildings[data_buildings['FIELD1'].notna()]
data_okn_buildings.head()
```
## 3. Создание сетки
Получение экстенда на основе точек ОКН. Таким образом мы обозначаем границы сетки.
```python
total_bounds = data_okn_points.total_bounds
minX, minY, maxX, maxY = total_bounds
```
Задаем размер единицы (ячейки) сетки.
```python
square_size = 300
```
Создаем сетку.
```python
grid_cells = []
x, y = (minX, minY)
geom_array = []

while y <= maxY:
        while x <= maxX:
            geom = geometry.Polygon([(x,y), (x, y+square_size), (x+square_size, y+square_size), (x+square_size, y), (x, y)])
            geom_array.append(geom)
            x += square_size
        x = minX
        y += square_size


fishnet = gpd.GeoDataFrame(geom_array, columns=['geometry']).set_crs('EPSG:32637')
fishnet['id'] = fishnet.index
```
## 4. Подсчет точек в единицах сетки
Соединение точек с единицами сетки и подсчет их кол-ва внутри каждой.
```python
merged = gpd.sjoin(data_okn_points, fishnet, how='left', predicate='within')
merged['n'] = 1
dissolve = merged.dissolve(by="index_right", aggfunc="count")
fishnet.loc[dissolve.index, 'n'] = dissolve.n.values
```
## 5. Создание веб-карты 
Необходимо изменить CRS на WGS 84 / EPSG:4326
```python
data_okn_points_4326 = data_okn_points.to_crs('EPSG:4326')
m = folium.Map(location=[data_okn_points_4326.centroid.y.mean(), data_okn_points_4326.centroid.x.mean()], zoom_start=13,  tiles="cartodb positron", control_scale=True)
m
```
### Создаем картограмму на базе сетки
```python
folium.Choropleth(
    geo_data=fishnet,
    data=fishnet,
    columns=['id', 'n'],
    fill_color='BuPu',
    fill_opacity = 0.7,
    key_on='id',
    nan_fill_opacity=0,
    line_color = "#0000",
    legend_name="amount of heritage sites",
    name='Heritage Sites Concentration'
).add_to(m)
```
### Добавляем здания (полигоны) ОКН с подписью категории историко-культурного значения
```python
folium.GeoJson(
    data_okn_buildings,
    name="Heritage buildings",
    tooltip=folium.GeoJsonTooltip(fields=["Категория историко-культурного значения"]),
    popup=folium.GeoJsonPopup(fields=['Вид объекта']),
    style_function=lambda x: {
        "fillColor": 'green'
    },
    highlight_function=lambda x: {"fillOpacity": 0.8},
    zoom_on_click=True,
    show=False,
).add_to(m)
```
### Добавляем административные границы города
```python
folium.GeoJson(
    bounds,
    name = "Yaroslavl boundaries",
    style_function=lambda x:{
        "fillColor":"grey"
    },
    highlight_function=lambda x: {"fillOpacity": 0.1},
    zoom_on_click=True,
    show=False,
).add_to(m)
```
### Добавляем кластеризацию точек
```python
from folium.plugins import MarkerCluster
```
```python
marker_cluster = MarkerCluster(name='Heritage Sites')
mc1= folium.plugins.FeatureGroupSubGroup(marker_cluster, 'Heritage Sites')
m.add_child(marker_cluster)
m.add_child(mc1)
mc1.add_child(folium.GeoJson(data_okn_points_4326.to_json(), embed=False, show=False))
```
### Добавляем дополнительные виджеты на карту
```python
from folium.plugins import MousePosition
from folium.plugins import Fullscreen
from folium.plugins import MiniMap
```
```python
folium.LayerControl().add_to(m)
```
```python
MousePosition().add_to(m)
Fullscreen(
    position="bottomright",
    title="Expand me",
    title_cancel="Exit me",
    force_separate_button=True,
).add_to(m)
MiniMap().add_to(m)
```
## 6. Сохраняем карту в формате index.html и мы готовы ее опубликовать!
```python
m.save("index.html")
```
