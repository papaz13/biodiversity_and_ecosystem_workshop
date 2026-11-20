---
layout: page
title: 01_Detección_Cambios_y_Clasificación
parent: "00-GoogleEarth"
nav_order: 1
---

# Análisis de paisaje: Detección de cambios y Clasificación

## Objetivo
Usar las nuevas herramientas **Detect Change** y **Classify** de Google Earth en la web, basadas en los *embeddings* satelitales de **AlphaEarth Foundations (AEF)**, para identificar cambios en el paisaje y mapear coberturas del suelo en áreas de interés de la región.

📑 Presentación completa: [RedLAC - Google Earth Workshop #2](https://docs.google.com/presentation/d/1yWrpiWDF0723hjX611RNPhCl4P3pw2LDIcK-FbmkHhM/edit?usp=sharing)  
📂 Datos del taller (.shp / .kml): [Carpeta de datos](https://drive.google.com/drive/folders/1lnqEQvlJ2ETWOVK590MaZWmuWKMZDm5V?usp=sharing)  
*Material: Joel Conkling y Paula Paz, RedLAC Workshop 2026*

| Ejercicio | Área de estudio | Tema |
|---|---|---|
| 1 – Detección de cambios | Pucallpa, Perú | Deforestación, expansión urbana y cambios en cuerpos de agua |
| 2 – Clasificación | Reserva Nacional Tambopata, Madre de Dios, Perú | Minería y deforestación en un área protegida y de amortiguamiento |


---

## 🌎 ¿Qué es AlphaEarth Foundations?

AlphaEarth Foundations es un modelo que integra datos de muchas fuentes de observación de la Tierra en una sola representación por píxel: óptico (Sentinel-2, Landsat), térmico (Landsat), radar (Sentinel-1, ALOS PALSAR), LiDAR (GEDI), gravedad (GRACE), clima (ERA5-Land), elevación (Copernicus DEM), además de alineación con texto y datos semánticos.

El resultado es el **Satellite Embedding Dataset**: una capa anual, a 10 m de resolución, con 64 bandas por píxel. Estos *embeddings* permiten hacer búsqueda por similitud, clasificación y detección de cambios.

---

## Ejercicio 1: Detección de cambios

### Objetivos
1. Usar la herramienta **Detect Change** para identificar cambios desde 2017 en Pucallpa, Perú.
2. Explorar las nuevas funciones: barra de umbral y exportación a vectores.
3. Cargar un área de estudio en formato `.shp` / `.kml`.
4. Calcular el área de cambio (m²) relacionada con cambios como deforestación.

### Ejercicio previo (5 minutos)
Antes de usar la herramienta, intenta detectar cambios usando las **Imágenes históricas**. Elige una región (o usa Pucallpa) y responde: ¿qué cambió y cuándo?, ¿cómo lo identificaste? y ¿qué dificultades encontraste?

### Paso a paso
1. Abre **Detect Change** desde el menú principal de Google Earth.
2. Dibuja un área de interés o carga el polígono.
> ⚠️ El área de interés no debe superar ~20 200 ha (50 000 acres).
3. Selecciona los años a comparar (datos disponibles desde 2017).
4. Ajusta el **umbral de cambio** con el control deslizante (0 = sin cambio, 1 = cambio).

   💡 Modifica el umbral y observa los resultados en tiempo real; cuando estés conforme, guarda los resultados.

5. Crea la capa de salida:
   - **Ráster**, o
   - **Vector + tabla** con el área de cada polígono de cambio.

📂 [Ver proyecto de ejemplo en Google Earth](https://earth.google.com/earth/d/1dJgTcN98J-_5Wh4CYGa9VKcJIntDNR0s?usp=sharing)

### ¿Cómo funciona?
La herramienta calcula el **producto punto** entre las 64 bandas de los *embeddings* de cada año y normaliza el resultado. Así obtiene un puntaje de 0 (estable) a 1 (cambio).

📖 [Documentación oficial de Detect Change](https://developers.google.com/maps/documentation/earth/detect-change?hl=es-419)

### Aspectos a tener en cuenta
- Los *embeddings* combinan datos de distintos sensores, por lo que pueden detectar cambios que no se ven a simple vista: humedad del suelo, rotación de cultivos o degradación bajo el dosel.
- Un puntaje cercano a 1 no siempre corresponde a un cambio más *visible*.
- Cada *embedding* resume un año calendario completo; no equivale a una imagen de un día específico.
- Diferencias en fechas de cosecha entre años pueden marcarse como cambio aunque el uso siga siendo agrícola.

---

## Ejercicio 2: Clasificación de coberturas

Realizaremos una clasificación de coberturas del suelo a partir de los *embeddings* satelitales de AlphaEarth Foundations (AEF), utilizando el clasificador supervisado Random Forest.

### Objetivos
1. Cargar el área de estudio en formato `.shp` / `.kml`.
2. Usar la herramienta **Classify** para clasificar los tipos de cobertura de la Reserva Nacional Tambopata.
3. Validar rápidamente resultados con datos auziliares como registros mineros del **SERNANP**.
3. Usar **Detect Change** para revisar si hubo cambios desde 2017.

### ¿Cómo funciona?

```
Puntos de entrenamiento por clase (ejemplo 4 clases: bosque, minería, agua, deforestación)
            │
            ▼
Satellite Embedding Dataset (anual desde 2017 hasta 2025, 10 m, 64 bandas)
            │
            ▼
Random Forest (clasificador de Earth Engine)
            │
            ▼
        Mapa de coberturas
```
### Paso a paso
### Parte A: Clasificación
1. Abre la herramienta **Classify** desde el menú principal.
2. Define el área de interés: dibuja un polígono o carga el área de estudio (`.kml` / `.shp`).
3. Selecciona el año de clasificación entre 2017 a 2025.
4. Define las clases con el botón **+ Add custom class** (por ejemplo: bosque denso, minería, deforestación, agua).
5. Marca puntos de muestreo para cada clase.

   💡 **Tip:** 
   -> Puedes usar información auxiliar como raster de Land Cover (WorldCover v200), ESA para crear los puntos de muestreo. 
   -> Selecciona puntos balanceados por clase, distribuidos en toda el área y representativos de su variabilidad.

6. Genera el mapa (se necesitan **al menos 2 clases**).
7. Revisa el resultado e itera:
   - ¿No te convence? → agrega más puntos.
   - ¿Necesitas más clases? → crea una nueva y repite.
8. Compara tus resultados con los registros de minería del **SERNANP**.

### Parte B: Detección de cambios en coberturas de la tierra
1. Dentro del área protegida, dibuja un polígono como área de estudio.
2. Abre **Detect Change** y selecciona los años de interés (desde 2017 hasta 2025).
3. Modifica el umbral para detectar cambios asociados a minería y deforestación.
**Tip:** Teniendo en cuenta el ejercicio anterior un buen umbral de detección de cambios es >0.5 , verifica si se adapta a la zona de estudio.
4. Exporta los resultados en ráster y vector.
5. Estima el área aproximada afectada en Tambopata.

📂 [Ver proyecto de ejemplo en Google Earth](https://earth.google.com/earth/d/18_ZX3kYWOcoB7FRS6zJ0g_RkDgYtIqV-?usp=sharing)

### Tips para un buen mapeo
- **El mapa es tan bueno como los puntos que proporcionas.**
- La clasificación es **iterativa**: revisa, corrige y vuelve a generar.
- Los resultados se basan en *embeddings* **anuales a 10 m**, así que los objetos muy pequeños pueden no distinguirse.

---

## 💬 Comentarios
Comparte tus sugerencias desde Google Earth web: **Ayuda → Enviar comentarios**, e incluye **#RedLAC26** en tu mensaje.
