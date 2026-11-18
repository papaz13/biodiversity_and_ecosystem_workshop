---
layout: page
title: 00_Intro_GEE
parent: "01-GoogleEarthEngine"
nav_order: 1
---

## 00_Intro_GEE

Esta sesión introduce el Code Editor de Google Earth Engine usando como caso la **Reserva Nacional de Lachay** (Huaral, Lima, Perú). Se parte de JavaScript básico, se explora el relieve de la reserva con un modelo de elevación y se construyen **compuestos Sentinel-2 de la temporada húmeda y de la seca** para ver "aparecer" la vegetación de lomas con la neblina. Los compuestos de esta sesión son la base de las sesiones 2, 3 y 4.

## Objetivos

1. Entender la diferencia entre JavaScript que corre en el navegador y objetos `ee.` que se calculan en los servidores de Google.
2. Cargar un área protegida desde la Base Mundial de Áreas Protegidas (WDPA) y explorar su relieve con SRTM.
3. Construir compuestos Sentinel-2 de la temporada húmeda y de la seca, enmascarando nubes con Cloud Score+.
4. Calcular NDVI, NDMI y MNDWI, y mapear el reverdecimiento estacional (NDVI húmedo − NDVI seco).
5. Resumir numéricamente el NDVI medio dentro de la reserva.
6. (Opcional) Exportar los compuestos a Google Drive o a un Asset propio.

**Aporta a:** Meta 21 (datos e información para la toma de decisiones) del Marco Mundial de Biodiversidad Kunming-Montreal.

## Datos

- **`WCMC/WDPA/current/polygons`** — Base Mundial de Áreas Protegidas (límite de la reserva).<br>
Collection: `ee.FeatureCollection('WCMC/WDPA/current/polygons')`

- **`USGS/SRTMGL1_003`** — Modelo digital de elevación SRTM (30 m).<br>
Image: `ee.Image('USGS/SRTMGL1_003')`

- **`COPERNICUS/S2_SR_HARMONIZED`** — Sentinel-2 Nivel 2A, reflectancia de superficie armonizada.<br>
Collection: `ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')`

- **`GOOGLE/CLOUD_SCORE_PLUS/V1/S2_HARMONIZED`** — Cloud Score+, puntaje de claridad por píxel para enmascarar nubes y neblina.<br>
Collection: `ee.ImageCollection('GOOGLE/CLOUD_SCORE_PLUS/V1/S2_HARMONIZED')`

## Métodos

1. Fundamentos de JavaScript (variables, listas, diccionarios, funciones) y su puente hacia objetos de Earth Engine.
2. Filtrado de una `FeatureCollection` por atributo (`ee.Filter.eq`) para extraer un área protegida.
3. Estadísticos de una imagen en una región con `reduceRegion` y `ee.Reducer.minMax()`.
4. Reclasificación por zonas de elevación cada 100 m con `divide().floor().multiply()`.
5. Filtrado de colecciones por área, año y rango de meses (`filterBounds`, `filterDate`, `calendarRange`).
6. Enmascaramiento de nubes píxel a píxel con Cloud Score+ (`linkCollection`) y escalado a reflectancia.
7. Índices espectrales con `normalizedDifference` y compuestos por mediana.
8. Compuestos alternativos: `qualityMosaic` de máximo NDVI.

## Parte 0 — JavaScript básico

```javascript
// Variables
var ciudad = 'Lima';
print(ciudad);

// Listas: los elementos se cuentan desde 0
var ciudades = ['Lima', 'Huacho', 'Trujillo', 'Arequipa', 'Cusco'];
print(ciudades);
print('Segunda ciudad de la lista:', ciudades[1]);

// Diccionarios (objetos): pares clave–valor
var datosArea = {
  nombre: 'Lachay',
  coordenadas: [-77.39, -11.37],
  superficieHa: 5070
};
print(datosArea);
print('Superficie (ha):', datosArea.superficieHa);

// Funciones
var saludar = function(nombre) {
  return 'Hola ' + nombre + ', bienvenida/o al taller';
};
print(saludar('Paula'));

// Puente hacia GEE
var puntoLachay = ee.Geometry.Point(datosArea.coordenadas);
Map.centerObject(puntoLachay, 13);
Map.addLayer(puntoLachay, {color: 'blue'}, 'Punto de referencia (Lachay)');
```

**Parámetros:**
- **`ciudades[1]`** — las listas se indexan desde 0: el índice `0` es `'Lima'` y el `1` es `'Huacho'`, la segunda ciudad.
- **`coordenadas: [-77.39, -11.37]`** — orden `[longitud, latitud]`, que es el orden que espera `ee.Geometry.Point`. Es el centro aproximado de la reserva (11°20'–11°24' S, 77°19'–77°28' O).
- **`superficieHa: 5070`** — superficie de referencia según el MINAM (*Gestión del agua y biodiversidad en Lomas de Lachay*).
- **Regla práctica:** todo lo que empieza con `ee.` se calcula en los servidores de Google; el resto es JavaScript normal que corre en el navegador. `ee.Geometry.Point(...)` es el puente entre ambos mundos.

> **Nota de transcripción:** el rótulo del `print` dice "Segunda ciudad" y por eso el índice correcto es `1`. Con `ciudades[3]` se imprimiría `Arequipa`, que es la cuarta.

## Parte 1 — Una imagen: relieve y franja de neblina

### 1.1 Área protegida desde la WDPA

```javascript
var wdpa = ee.FeatureCollection('WCMC/WDPA/current/polygons');
var lachay = wdpa.filter(ee.Filter.eq('SITE_ID', 254));
var area = lachay.geometry();
print('Área protegida (verificar nombre):', lachay.aggregate_array('NAME'));

Map.centerObject(area, 13);
Map.addLayer(ee.Image().paint(lachay, 0, 5), {palette: ['yellow']}, 'Lachay (límite WDPA)');
```

**Parámetros:**
- **`SITE_ID`** — identificador único de cada área en la WDPA. El script `A0_Buscador_Areas_Protegidas` sirve para encontrar el de cualquier área; aquí conviene verificar el nombre impreso antes de continuar.
- **`ee.Image().paint(lachay, 0, 5)`** — dibuja solo el contorno (grosor 5 px) para no tapar las imágenes que se agregan debajo.

### 1.2 Elevación SRTM

```javascript
var elevacion = ee.Image('USGS/SRTMGL1_003').select('elevation');

var estadElev = elevacion.reduceRegion({
  reducer: ee.Reducer.minMax(),
  geometry: area,
  scale: 30
});
print('Elevación mínima y máxima en Lachay (m):', estadElev);

var areaVista = area.buffer(50);
Map.addLayer(elevacion.clip(areaVista), {min: 0, max: 800, palette: ['006837', 'a6d96a', 'ffffbf', 'fdae61', 'a50026']}, 'Elevación (m)', true);
```

**Parámetros:**
- **`ee.Reducer.minMax()`** — devuelve el mínimo y el máximo de la elevación dentro de `area`; sirve para ajustar los valores `min`/`max` de la visualización al rango real.
- **`areaVista = area.buffer(50)`** — 50 m extra alrededor de la reserva para ver algo de contexto; es la región de recorte (`clip`) y de exportación en el resto de la sesión.

### 1.3 Zonas de elevación cada 100 m

```javascript
var zonas = elevacion.divide(100).floor().multiply(100).rename('zona_100m');
Map.addLayer(zonas.clip(areaVista), {min: 0, max: 1000, palette: ['313695', '74add1', 'e0f3f8', 'fee090', 'f46d43', 'a50026']}, 'Zonas de elevación cada 100 m', false);
```

**Parámetros:**
- **`divide(100).floor().multiply(100)`** — redondea cada píxel hacia abajo al múltiplo de 100 m más cercano (por ejemplo 347 m pasa a 300 m).
- **Contexto ecológico:** la vegetación de lomas crece en laderas orientadas al mar, desde cerca del nivel del mar hasta unos 1 000 m; por encima, la inversión térmica corta la influencia de la neblina (MINAM).
- **Actividad:** con la herramienta **Inspector**, hacer clic en distintas laderas y anotar la zona de elevación.

## Parte 2 — Una colección: compuestos Sentinel-2 húmedo vs. seco

> **Nota técnica — bloque desactivado en el archivo fuente:** en el script original, la Parte 2, el Nivel intermedio y la Parte 3 están dentro de un único comentario de bloque que abre con `/*` justo antes del encabezado "PARTE 2" y cierra con `*/` en la última línea. Mientras esos delimitadores estén puestos, esas secciones no se ejecutan. Para correrlas hay que eliminar ambos delimitadores.

### 2.1 Temporadas

```javascript
var ANIO = 2024;
var MESES_HUMEDO = [7, 10];   // julio–octubre
var MESES_SECO = [1, 3];      // enero–marzo
```

**Parámetros:**
- **`ANIO`** — año del análisis; se puede cambiar entre 2019 y 2025 para comparar el reverdecimiento entre años.
- **`MESES_HUMEDO` / `MESES_SECO`** — en Lachay el período húmedo (neblina) va de mayo a octubre, con máxima neblina de junio a septiembre, y el seco de diciembre a abril (MINAM). Se toma el final de la temporada húmeda, cuando la vegetación ya respondió a la neblina, y el corazón de la seca.

### 2.2 Colecciones y máscara de nubes

```javascript
var s2 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED');
var csPlus = ee.ImageCollection('GOOGLE/CLOUD_SCORE_PLUS/V1/S2_HARMONIZED');

var BANDA_QA = 'cs_cdf';
var UMBRAL_CLARO = 0.60;
```

**Parámetros:**
- **`BANDA_QA = 'cs_cdf'`** — banda de Cloud Score+ usada para decidir qué píxeles están despejados.
- **`UMBRAL_CLARO = 0.60`** — píxeles con `cs_cdf` mayor o igual a este valor se conservan; el catálogo indica que valores entre 0.50 y 0.65 suelen funcionar bien.
- **Por qué no se usa `QA60`:** la banda `QA60` de Sentinel-2 está vacía entre el 2022-01-25 y el 2024-02-28 (según el catálogo de GEE), lo que invalida el enmascaramiento clásico para buena parte del período de análisis.

### 2.3 Preparación de imágenes

```javascript
function prepararS2(imagen) {
  var mascara = imagen.select(BANDA_QA).gte(UMBRAL_CLARO);
  var reflectancia = imagen.select('B.*').divide(10000);
  return ee.Image(reflectancia.updateMask(mascara).copyProperties(imagen, ['system:time_start']));
}
```

**Parámetros:**
- **`select('B.*')`** — expresión regular que selecciona todas las bandas espectrales (B1…B12 y B8A) y deja fuera las bandas de control de calidad.
- **`divide(10000)`** — convierte los números digitales a reflectancia de 0 a 1.
- **`ee.Image(...)`** — `copyProperties` devuelve un objeto genérico; `ee.Image()` lo vuelve a convertir en imagen para que `map` funcione correctamente.

### 2.4 Índices espectrales

```javascript
function agregarIndices(imagen) {
  var ndvi = imagen.normalizedDifference(['B8', 'B4']).rename('NDVI');
  var ndmi = imagen.normalizedDifference(['B8', 'B11']).rename('NDMI');
  var mndwi = imagen.normalizedDifference(['B3', 'B11']).rename('MNDWI');
  return imagen.addBands([ndvi, ndmi, mndwi]);
}
```

| Índice | Fórmula | Bandas | Qué mide |
|---|---|---|---|
| NDVI | (NIR − Rojo) / (NIR + Rojo) | B8, B4 | Verdor de la vegetación |
| NDMI | (NIR − SWIR1) / (NIR + SWIR1) | B8, B11 | Humedad de la vegetación |
| MNDWI | (Verde − SWIR1) / (Verde + SWIR1) | B3, B11 | Agua superficial |

### 2.5 Compuesto de una temporada

```javascript
function compuestoTemporada(meses) {
  var coleccion = s2
    .filterBounds(area)
    .filterDate(ANIO + '-01-01', (ANIO + 1) + '-01-01')
    .filter(ee.Filter.calendarRange(meses[0], meses[1], 'month'))
    .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 80))
    .linkCollection(csPlus, [BANDA_QA])
    .map(prepararS2)
    .map(agregarIndices);
  print('Imágenes en la temporada (meses ' + meses[0] + '–' + meses[1] + '):', coleccion.size());
  return coleccion.median().clip(areaVista);
}

var compHumedo = compuestoTemporada(MESES_HUMEDO);
var compSeco = compuestoTemporada(MESES_SECO);
```

**Parámetros:**
- **`calendarRange(meses[0], meses[1], 'month')`** — conserva solo las imágenes cuyo mes cae dentro del rango de la temporada.
- **`CLOUDY_PIXEL_PERCENTAGE < 80`** — filtro por metadato deliberadamente amplio: Cloud Score+ trabaja píxel a píxel, y en la temporada húmeda un filtro estricto por escena eliminaría casi todas las imágenes.
- **`linkCollection(csPlus, [BANDA_QA])`** — une a cada imagen de Sentinel-2 la banda `cs_cdf` de su imagen Cloud Score+ correspondiente.
- **`median()`** — la mediana por píxel reduce el efecto de nubes residuales y sombras.

### 2.6 Visualización

```javascript
var visColor = {bands: ['B4', 'B3', 'B2'], min: 0, max: 0.25};
var visFalso = {bands: ['B8', 'B4', 'B3'], min: 0, max: 0.35};
var visNDVI = {min: 0, max: 0.6, palette: ['8c510a', 'd8b365', 'f6e8c3', 'c7eae5', '5ab4ac', '01665e']};

Map.addLayer(compSeco, visColor, 'Seco ' + ANIO + ' – color verdadero', false);
Map.addLayer(compHumedo, visColor, 'Húmedo ' + ANIO + ' – color verdadero', false);
Map.addLayer(compSeco, visFalso, 'Seco ' + ANIO + ' – falso color', false);
Map.addLayer(compHumedo, visFalso, 'Húmedo ' + ANIO + ' – falso color');
Map.addLayer(compSeco.select('NDVI'), visNDVI, 'NDVI seco', false);
Map.addLayer(compHumedo.select('NDVI'), visNDVI, 'NDVI húmedo', false);
```

**Parámetros:**
- **`visFalso`** — composición falso color (B8, B4, B3): la vegetación sana aparece en rojo.
- **Actividad:** activar y desactivar las capas en el panel para comparar color verdadero y falso color entre ambas temporadas.

### 2.7 Reverdecimiento estacional

```javascript
var difNDVI = compHumedo.select('NDVI').subtract(compSeco.select('NDVI')).rename('dif_NDVI');
Map.addLayer(difNDVI, {min: -0.1, max: 0.4, palette: ['a50026', 'ffffbf', '006837']}, 'Reverdecimiento estacional (NDVI húmedo − seco)');
```

**Parámetros:**
- **`dif_NDVI`** — valores positivos (verde) indican dónde la vegetación reverdece con la neblina; valores cercanos a cero indican que no hay cambio estacional.
- **Actividad:** comparar el reverdecimiento con las zonas de elevación de la Parte 1 y responder en qué franja se concentra.

### 2.8 Resumen numérico

```javascript
var ndviMedio = ee.Image.cat([compSeco.select('NDVI').rename('NDVI_seco'), compHumedo.select('NDVI').rename('NDVI_humedo')])
  .reduceRegion({reducer: ee.Reducer.mean(), geometry: area, scale: 10, maxPixels: 1e9});
print('NDVI medio en Lachay (seco vs. húmedo):', ndviMedio);
```

**Parámetros:**
- **`scale: 10`** — resolución nativa de las bandas B2, B3, B4 y B8 de Sentinel-2.
- **`ee.Image.cat`** — apila ambas temporadas en una sola imagen de dos bandas para resolver las dos medias con un solo `reduceRegion`.

## Nivel intermedio (opcional)

```javascript
var mosaicoVerde = s2
  .filterBounds(area)
  .filterDate(ANIO + '-01-01', (ANIO + 1) + '-01-01')
  .filter(ee.Filter.calendarRange(MESES_HUMEDO[0], MESES_HUMEDO[1], 'month'))
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 80))
  .linkCollection(csPlus, [BANDA_QA])
  .map(prepararS2)
  .map(agregarIndices)
  .qualityMosaic('NDVI')
  .clip(areaVista);
Map.addLayer(mosaicoVerde, visFalso, 'Húmedo – mosaico de máximo NDVI', false);
```

**Parámetros:**
- **`qualityMosaic('NDVI')`** — en lugar de la mediana, conserva en cada píxel los valores de la imagen con mayor NDVI de la temporada: un mosaico "más verde".
- **Ejercicios sugeridos:** (a) reemplazar `median()` por `mean()`, `min()` o `max()` en `compuestoTemporada` y observar el efecto de las nubes residuales; (b) probar `UMBRAL_CLARO` en 0.50 y 0.65 y contar cuántos píxeles quedan sin datos.

## Parte 3 — Exportación (opcional)

```javascript
var MI_CARPETA_ASSETS = 'projects/SU-PROYECTO/assets/';

Export.image.toDrive({
  image: compHumedo.select(['B2', 'B3', 'B4', 'B8', 'B11', 'B12', 'NDVI', 'NDMI', 'MNDWI']).toFloat(),
  description: 'S2_humedo_' + ANIO + '_Lachay',
  folder: 'GEE_taller_lima',
  region: areaVista,
  scale: 10,
  maxPixels: 1e10
});

Export.image.toAsset({
  image: compHumedo.select(['B2', 'B3', 'B4', 'B8', 'B11', 'B12', 'NDVI', 'NDMI', 'MNDWI']).toFloat(),
  description: 'S2_humedo_' + ANIO + '_Lachay_asset',
  assetId: MI_CARPETA_ASSETS + 'S2_humedo_' + ANIO + '_Lachay',
  region: areaVista,
  scale: 10,
  maxPixels: 1e10
});
```

**Parámetros:**
- **`MI_CARPETA_ASSETS`** — reemplazar `SU-PROYECTO` por el ID del propio proyecto de Google Cloud antes de exportar a Asset.
- **`toFloat()`** — fija el tipo de dato a flotante de 32 bits para que bandas de reflectancia y de índices convivan en un mismo archivo sin pérdida de valores decimales.
- Ambos bloques quedan comentados por defecto; hay que descomentarlos para generar las tareas en la pestaña **Tasks**.

## Notas metodológicas generales

- **Compuestos como base común:** los compuestos húmedo y seco de esta sesión se reutilizan en las sesiones siguientes; conviene exportarlos a Asset para no recalcularlos.
- **Mediana vs. mosaico de máximo NDVI:** la mediana es más estable y representa mejor la condición típica de la temporada; el mosaico de máximo NDVI resalta el pico de verdor pero es más sensible a artefactos (por ejemplo, sombras o nubes residuales con valores altos).
- **Neblina y Cloud Score+:** la neblina de las lomas puede ser tratada por Cloud Score+ como píxeles poco claros; si quedan muy pocos píxeles válidos en la temporada húmeda, bajar `UMBRAL_CLARO` es la primera palanca.
- **Autoría:** las Partes 0 y 1 están adaptadas de EEFA (cap. F1) y de ejemplos de Google; mantener la atribución en cualquier material derivado.


## Script "`00_Intro_GEE`":
https://code.earthengine.google.com/?scriptPath=users%2Fpaulapaz1101%2Fbiodiversity_and_ecosystem_workshop%3ALima_Nov_2026%2F00_Introduccion
