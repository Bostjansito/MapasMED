# Proyecto Waze — Ruteo urbano multiobjetivo en Medellín

Fundamentos de IA · Semestre 6

Notebook principal: **`Ruteo_Multiobjetivo_Medellin.ipynb`**

## Estructura de la carpeta

```
ProyectoWaze/
├── Ruteo_Multiobjetivo_Medellin.ipynb   <- notebook principal
├── README.md
├── datos/                               <- los 7 CSV de entrada
│   ├── hurto_a_persona.csv
│   ├── hurto_de_moto.csv
│   ├── hurto_de_carro.csv
│   ├── hurto_por_pirateria_terrestre.csv
│   ├── 202608091356-ansv-fotodeteccion.csv
│   ├── velocidad_e_intensidad_vehicular_en_medellin.csv
│   └── Criminalidad por Comuna_data.csv
├── referencia/
│   └── IA_EIA_LatLong_InformacionMapa.ipynb   (notebook de referencia del curso)
├── cache/                               <- grafo de OSM cacheado (se crea solo)
└── salidas/                             <- mapas HTML generados
```

## Los datos no están en el repositorio

Cuatro de los siete CSV superan los límites de GitHub, así que están excluidos vía
`.gitignore`. Hay que descargarlos y ponerlos en `datos/` con estos nombres exactos:

| Archivo | Tamaño | Fuente |
|---|---|---|
| `hurto_a_persona.csv` | 118,7 MB | Datos Abiertos de Medellín — Secretaría de Seguridad |
| `velocidad_e_intensidad_vehicular_en_medellin.csv` | 75,7 MB | Datos Abiertos de Medellín — Secretaría de Movilidad |
| `hurto_de_moto.csv` | 30,5 MB | Datos Abiertos de Medellín |
| `hurto_de_carro.csv` | 13,7 MB | Datos Abiertos de Medellín |
| `hurto_por_pirateria_terrestre.csv` | 0,8 MB | Datos Abiertos de Medellín |

Los dos pequeños sí vienen incluidos en el repositorio:
`Criminalidad por Comuna_data.csv` y `202608091356-ansv-fotodeteccion.csv` (ANSV).

> `hurto_a_persona.csv` pesa 118,7 MB y GitHub **rechaza** cualquier archivo de más de
> 100 MB. No es un aviso, es un bloqueo: si se intenta subir, el `push` falla. Por eso
> los datos se distribuyen aparte y no con Git LFS (la cuota gratuita de LFS es de 1 GB
> y se consumiría en unos pocos commits).

## Cómo ejecutarlo

1. Abrir el notebook desde esta carpeta (Jupyter, VS Code o Colab).
2. Descomentar y ejecutar la celda de instalación:
   `!pip install -q osmnx folium scikit-learn pyproj geopy`
   En **Google Colab hay que reiniciar el entorno** después de instalar `osmnx`.
3. Ejecutar todo. La primera vez descarga la red vial de Medellín desde OpenStreetMap
   (varios minutos); después queda cacheada en `cache/medellin_drive.graphml` y arranca al instante.

## Qué hace

Construye el grafo vial de Medellín con OSMnx y le añade a **cada arista** cuatro atributos
derivados de los datos:

| Atributo | Cómo se calcula |
|---|---|
| `riesgo` | Densidad de hurtos con kernel gaussiano (σ=150 m) sobre el punto medio de la arista, ponderada por gravedad del delito, mezclada 75/25 con el riesgo agregado de la comuna |
| `camaras` | Densidad de cámaras de fotodetección **activas** en un radio de 400 m |
| `vel_efectiva` | Velocidad realmente observada: sensor más cercano (<700 m) → promedio del corredor por nombre de calle → `maxspeed` de OSM |
| `congestion` | `1 − v_observada / v_libre` |

Y minimiza el costo compuesto:

```
c(e) = [(1−λ)·t̂(e) + λ·d̂(e)] · [1 + w_seg·riesgo + w_cong·congestión − w_cam·cámaras]
```

* **Costo base** — tiempo y distancia adimensionalizados por su media global; `λ` controla el trade-off.
* **Factor** — recortado a un mínimo de 0.15 para garantizar pesos ≥ 0 (requisito de Dijkstra y A\*).

## Uso rápido

```python
ruta, resumen, mapa = buscar_ruta(
    "Universidad de Antioquia",
    "Centro Comercial Santafé",
    perfil="Más segura",   # Más rápida | Más corta | Equilibrada | Más segura
    hora=18,               # opcional: usa el perfil horario de velocidad y de delitos
)
mapa
```

Los pesos se pueden sobrescribir uno a uno:
`buscar_ruta(origen, destino, w_seg=3.0, w_cam=0.8, lam=0.3)`

## Salidas

* `salidas/mapa_rutas.html` — mapa interactivo con las 4 rutas, heatmap de hurtos, cámaras y sensores
* `salidas/mapa_riesgo_red.html` — mapa de riesgo de toda la red vial

## Notas técnicas (trampas de los datos)

* Las coordenadas de `velocidad_e_intensidad_vehicular_en_medellin.csv` están en
  **EPSG:3116** (metros), no en grados; el notebook las reproyecta a EPSG:4326.
* Ese mismo archivo usa **coma decimal**.
* `Criminalidad por Comuna_data.csv` está separado por **`;`** y con BOM (`utf-8-sig`).
* Solo se cuentan como disuasorias las cámaras con estado *Operando*, *Autorizada instalar*
  o *Prorrogada* (98 de 201 en Antioquia).
* Se filtran hechos desde 2019 y coordenadas dentro del bounding box del Valle de Aburrá.
* La heurística de A\* (`c_min · haversine`) es admisible y consistente por construcción:
  devuelve el mismo óptimo que Dijkstra expandiendo ~48 % menos nodos.
