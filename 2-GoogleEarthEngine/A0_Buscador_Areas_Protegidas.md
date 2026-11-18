---
layout: page
title: A0_Buscador_Areas_Protegidas
parent: "01-GoogleEarthEngine"
nav_order: 2
---

## A0_Buscador_Areas_Protegidas

Este script encuentra el **`SITE_ID`** de un área protegida en la Base Mundial de Áreas Protegidas (WDPA). Ese número es el que se pega en los scripts A1, A2 y A3 para analizar el área elegida. Ofrece dos formas de búsqueda (por nombre y con un clic en el mapa) y explica qué hacer si el área no está en la WDPA.

## Objetivos

1. Buscar áreas protegidas por país (código ISO3) y por una palabra del nombre.
2. Identificar el `SITE_ID`, la designación y la superficie de cada resultado.
3. Detectar áreas registradas solo como punto (sin límite) y áreas no terrestres, que no sirven igual en los scripts A2 y A3.
4. Obtener el `SITE_ID` haciendo clic directamente sobre el mapa.
5. Conocer las alternativas cuando el área no está en la WDPA (subir un asset o dibujarla).

## Datos

- **`WCMC/WDPA/current/polygons`** — Áreas protegidas con límite (polígonos). Actualización mensual; desde 2025 el identificador se llama `SITE_ID`.<br>
Collection: `ee.FeatureCollection('WCMC/WDPA/current/polygons')`

- **`WCMC/WDPA/current/points`** — Áreas protegidas registradas solo como punto, sin límite.<br>
Collection: `ee.FeatureCollection('WCMC/WDPA/current/points')`

## Métodos

1. Filtrado por país con `ee.Filter.stringContains` sobre el campo `ISO3`.
2. Búsqueda de texto insensible a mayúsculas: se crean propiedades auxiliares en minúsculas y se filtra con `ee.Filter.or`.
3. Impresión legible de resultados con `reduceColumns(ee.Reducer.toList(...))` y `evaluate()`.
4. Verificación de resultados por tipo: solo punto, no terrestres (`REALM`).
5. Consulta interactiva por clic con `Map.onClick` y `filterBounds`.

## Paso 1 — Parámetros

```javascript
var PAIS_ISO3 = 'PER';
var TEXTO = 'tambo';
```

**Parámetros:**
- **`PAIS_ISO3`** — código de país de tres letras: `PER`, `BOL`, `COL`, `ECU`, `BRA`, `CHL`, `MEX`, `GTM`, `SLV`, `CRI`, `BLZ`, `GUY`, `USA`, etc.
- **`TEXTO`** — una palabra del nombre, corta (por ejemplo, `'tambo'` encuentra `Tambopata`). Estas son las únicas dos líneas que se cambian.

## Paso 2 — Búsqueda por nombre

```javascript
var wdpa = ee.FeatureCollection('WCMC/WDPA/current/polygons');
var wdpaPuntos = ee.FeatureCollection('WCMC/WDPA/current/points');

var CAMPOS = ['SITE_ID', 'NAME', 'NAME_ENG', 'DESIG', 'STATUS', 'REALM', 'GIS_AREA'];
var textoMin = TEXTO.toLowerCase();

function buscar(coleccion) {
  return coleccion
    .filter(ee.Filter.stringContains('ISO3', PAIS_ISO3))
    .map(function(f) {
      var nombre = ee.String(ee.Algorithms.If(f.get('NAME'), f.get('NAME'), '')).toLowerCase();
      var nombreEng = ee.String(ee.Algorithms.If(f.get('NAME_ENG'), f.get('NAME_ENG'), '')).toLowerCase();
      return f.set('nombre_min', nombre).set('nombre_eng_min', nombreEng);
    })
    .filter(ee.Filter.or(
      ee.Filter.stringContains('nombre_min', textoMin),
      ee.Filter.stringContains('nombre_eng_min', textoMin)
    ));
}
```

**Parámetros:**
- **`stringContains('ISO3', ...)`** — se usa "contiene" y no igualdad porque las áreas transfronterizas pueden tener varios países en el campo `ISO3`.
- **`toLowerCase()`** — pasa nombres y texto a minúsculas para que la búsqueda no dependa de mayúsculas.
- **`NAME` y `NAME_ENG`** — se busca tanto en el nombre original como en el nombre en inglés.
- **`ee.Algorithms.If(..., '')`** — sustituye valores nulos por texto vacío para evitar errores al convertir a minúsculas.

> **Nota técnica — tildes:** la búsqueda convierte a minúsculas pero **no elimina tildes**. Si el nombre guardado en la WDPA tiene una vocal acentuada, un texto sin tilde que la incluya no coincidirá en ese campo (por ejemplo, `'huascaran'` frente a `Huascarán`). La búsqueda en `NAME_ENG` compensa parcialmente, pero lo más seguro es elegir un fragmento que no contenga vocales acentuadas (`'huasc'`).

## Paso 3 — Impresión de resultados

```javascript
function imprimirLista(fc, titulo) {
  var filas = fc.limit(50).map(function(f) {
    return f.set({
      _n: ee.Algorithms.If(f.get('NAME'), f.get('NAME'), '-'),
      _d: ee.Algorithms.If(f.get('DESIG'), f.get('DESIG'), '-'),
      _a: ee.Algorithms.If(f.get('GIS_AREA'), f.get('GIS_AREA'), 0)
    });
  }).reduceColumns(ee.Reducer.toList(4), ['SITE_ID', '_n', '_d', '_a']).get('list');
  ee.List(filas).evaluate(function(lista) {
    print(titulo);
    if (!lista || lista.length === 0) { print('   (ninguna)'); return; }
    for (var i = 0; i < lista.length; i++) {
      var r = lista[i];
      print('   ' + (i + 1) + '.  SITE_ID ' + r[0] + '  –  ' + r[1] + '  (' + r[2] + ', ' + Math.round(r[3] * 100) + ' ha)');
    }
  });
}

var encontradas = buscar(wdpa);
var encontradasPuntos = buscar(wdpaPuntos);

print('Número de polígonos encontrados:', encontradas.size());
imprimirLista(encontradas, 'RESULTADOS – copiar el SITE_ID de su área:');
print('Detalle de los resultados (tabla completa):', encontradas.select(CAMPOS).limit(50));
```

**Parámetros:**
- **`limit(50)`** — la lista impresa muestra como máximo 50 resultados.
- **`ee.Reducer.toList(4)`** — junta en una lista los cuatro campos seleccionados (`SITE_ID`, nombre, designación, superficie) de todas las filas.
- **`Math.round(r[3] * 100)`** — el campo `GIS_AREA` de la WDPA viene en km²; multiplicar por 100 lo convierte a hectáreas.
- **`evaluate()`** — trae el resultado del servidor al navegador para poder armar una línea de texto por área en la consola.

## Paso 4 — Avisos: solo punto, sin resultados y no terrestres

```javascript
encontradasPuntos.size().evaluate(function(n) {
  if (n > 0) {
    imprimirLista(encontradasPuntos, 'ATENCIÓN: estas áreas están en la WDPA solo como PUNTO (sin límite). Para analizarlas, ' +
          'use el modo "asset" o "dibujo" en los scripts A1–A3:');
  }
});

encontradas.size().evaluate(function(n) {
  if (n === 0) {
    print('No se encontraron polígonos con "' + TEXTO + '" en ' + PAIS_ISO3 + '. Pruebe otra palabra (sin tildes), ' +
          'revise el código de país, o haga clic sobre su área en el mapa (sección 3). Si su área no está en la WDPA, ' +
          'use el modo "asset" o "dibujo" en los scripts A1–A3.');
  }
});

var noTerrestres = encontradas.filter(ee.Filter.neq('REALM', 'Terrestrial'));
noTerrestres.size().evaluate(function(n) {
  if (n > 0) {
    imprimirLista(noTerrestres, 'Nota: estas áreas no son terrestres (campo REALM). Los scripts A2 (pérdida de cobertura) y ' +
          'A3 (conectividad) están pensados para áreas terrestres:');
  }
});

Map.addLayer(encontradas.style({color: 'ffff00', fillColor: 'ffff0033', width: 2}), {}, 'Resultados de la búsqueda');
encontradas.size().evaluate(function(n) {
  if (n > 0) { Map.centerObject(encontradas, 7); }
});
```

**Parámetros:**
- **Áreas solo como punto:** no tienen límite, por lo que no sirven con `SITE_ID` en los scripts de análisis; hay que usar `MODO = 'asset'` o `'dibujo'`.
- **`REALM`** — distingue áreas terrestres, marinas y costeras. La pérdida de cobertura arbórea (A2) y la conectividad terrestre (A3) no aplican igual a áreas marinas.
- **`style({color, fillColor, width})`** — contorno amarillo con relleno translúcido (el sufijo `33` en el código hexadecimal es la transparencia).

## Paso 5 — Búsqueda con un clic en el mapa

```javascript
Map.style().set('cursor', 'crosshair');
var capaClic = null;
Map.onClick(function(coords) {
  var punto = ee.Geometry.Point([coords.lon, coords.lat]);
  var enPunto = wdpa.filterBounds(punto).select(CAMPOS);
  imprimirLista(enPunto, 'Áreas protegidas en el punto (' + coords.lon.toFixed(4) + ', ' + coords.lat.toFixed(4) + '):');
  if (capaClic !== null) { Map.layers().remove(capaClic); }
  capaClic = ui.Map.Layer(enPunto.style({color: 'ff0000', fillColor: 'ff000033', width: 2}), {}, 'Área seleccionada con clic');
  Map.layers().add(capaClic);
});
```

**Parámetros:**
- **`Map.onClick`** — al hacer clic, la consola muestra todas las áreas de la WDPA que contienen ese punto, con su `SITE_ID`. Varias áreas pueden superponerse en un mismo punto (por ejemplo, un parque nacional dentro de una zona reservada).
- **`capaClic`** — guarda la capa roja de la selección anterior para quitarla antes de dibujar la nueva.
- **`Map.style().set('cursor', 'crosshair')`** — cambia el cursor a una cruz para indicar que el mapa responde a clics.

## Paso 6 — Si el área no está en la WDPA

**Opción A: subir el límite como asset** (recomendado si se tiene el límite oficial). Conviene hacerlo antes de la sesión porque la carga tarda minutos.
1. Panel izquierdo → pestaña **Assets** → botón **NEW** → **Shape files**.
2. Seleccionar los archivos del shapefile: `.shp`, `.shx`, `.dbf` y `.prj` (el `.prj` es necesario para la proyección).
3. Escribir un nombre (por ejemplo `mi_area`) → **UPLOAD**. Revisar el avance en la pestaña **Tasks**.
4. Al terminar, copiar la ruta del asset (por ejemplo `projects/SU-PROYECTO/assets/mi_area`) y pegarla en `RUTA_ASSET` de A1–A3, con `MODO = 'asset'`.

**Opción B: dibujar el área.** En los scripts A1–A3, usar la herramienta de polígono del mapa, nombrar la geometría `miArea` y poner `MODO = 'dibujo'`. Es lo más rápido, pero menos preciso que un límite oficial.

## Notas metodológicas generales

- **Salida del script:** el `SITE_ID` obtenido aquí es la única entrada obligatoria para correr A1, A2 y A3 en `MODO = 'wdpa'`.
- **Cobertura de la WDPA:** una búsqueda sin resultados no implica que el área no exista; puede estar registrada con otro nombre, en otro idioma, solo como punto o en otro país.
- **Actualización mensual:** el catálogo se actualiza cada mes, por lo que un `SITE_ID` o un nombre puede cambiar entre versiones. Conviene verificar el nombre impreso antes de usar el identificador en otros scripts.

## Script "`A0_Buscador_Areas_Protegidas`":
https://code.earthengine.google.com/?scriptPath=users%2Fpaulapaz1101%2Fbiodiversity_and_ecosystem_workshop%3ALima_Nov_2026%2FA0_Buscador_Areas_Protegidas -->

