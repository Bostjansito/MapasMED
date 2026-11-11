# Ruteo y búsqueda heurística en Medellín

Proyecto académico de **Fundamentos de IA** que explora métodos de búsqueda y heurísticas mediante el cálculo de rutas vehiculares en Medellín. Sobre la red vial de OpenStreetMap, compara rutas por distancia, tiempo estimado e inseguridad histórica, así como el costo y los nodos expandidos por distintos algoritmos.

## Enfoque

- **Búsqueda:** implementaciones propias de costo uniforme, primero el mejor y A*.
- **Heurísticas:** una referencia basada en distancia en línea recta y una propuesta propia que incorpora las características de las vías cercanas.
- **Modelo de inseguridad:** cuenta hurtos cercanos por franja —mañana (06:00–12:00), tarde (12:00–19:00) y noche (19:00–06:00)— y normaliza los conteos. El costo de seguridad aumenta con la inseguridad y aplica un descuento al peligro si hay una cámara cercana.
- **Tiempo:** utiliza el promedio de velocidad del sensor cercano o, en su ausencia, la velocidad estimada de OpenStreetMap.

El proyecto es **exploratorio, no un sistema de navegación ni una garantía de seguridad**. Los datos históricos, las aproximaciones espaciales y el supuesto de que una cámara reduce el peligro no describen necesariamente las condiciones actuales.

## Heurística propuesta

La heurística estima el costo restante multiplicando la distancia en línea recta por un resumen de las vías alcanzables en **uno y dos pasos** desde el nodo actual:

```text
h(n) = distancia_recta(n, destino) × 0,998 × (A × factor_medio + B × lentitud_media)
```

El **factor** refleja la penalización por inseguridad de la franja elegida; la **lentitud** compara la velocidad de cada vía con la máxima de la red. El cuaderno compara varias combinaciones y destaca `A = 0,5` y `B = 0,5`, que pondera ambas por igual. La búsqueda sigue optimizando el costo elegido: la heurística orienta la exploración, no cambia ese costo.

Frente a la heurística de distancia, esta estimación puede expandir menos nodos, pero también sobreestimar el costo restante. Por eso, **A* con esta heurística no garantiza la ruta óptima**. El cuaderno contrasta el ahorro de nodos y el sobrecosto frente a costo uniforme.

## Ejecución

Requiere Python y Jupyter Notebook o JupyterLab. Desde la raíz del proyecto:

```bash
python -m pip install jupyterlab
jupyter lab
```

1. Coloca los siguientes archivos CSV en `datos/`:
   - `hurto_a_persona.csv`, `hurto_de_moto.csv`, `hurto_de_carro.csv` y `hurto_por_pirateria_terrestre.csv`.
   - `velocidad_e_intensidad_vehicular_en_medellin.csv`.
   - `202608091356-ansv-fotodeteccion.csv`.
2. Abre `Ruteo_Multiobjetivo_Medellin.ipynb` y ejecuta las celdas en orden. El cuaderno instala las dependencias que falten; requiere conexión para instalarlas y para descargar el grafo de OpenStreetMap si no está en caché.
3. Ajusta `PARES`, `PAR` y `FRANJA_ACTUAL` en la sección 11 o usa `buscar_ruta()` en la sección 14. Los puntos pueden ser nombres de `LUGARES` o tuplas `(latitud, longitud)`.

Los CSV no se versionan. Las fuentes incluyen [Datos Abiertos Colombia](https://www.datos.gov.co/) para hurtos y [velocidad vehicular](https://www.datos.gov.co/d/4q5i-ub4e), y la [ANSV](https://www.ansv.gov.co/atencion_ciudadania/tramites/camaras/sistemas) para fotodetección. Para reproducir los resultados se necesitan los archivos y formatos usados por el cuaderno; las versiones actuales de las fuentes pueden diferir.

## Estructura y resultados

```text
Ruteo_Multiobjetivo_Medellin.ipynb  Análisis, búsquedas y visualizaciones
datos/                              CSV de entrada, no versionados
cache/                              Grafo y respuestas descargadas, no versionados
salidas/                            Figuras y mapa interactivo generados
referencia/                         Cuaderno de apoyo sobre mapas y rutas
```

El cuaderno genera comparaciones de rutas y franjas en PNG, además de `salidas/mapa_seguridad.html` con capas de vías, hurtos y cámaras. El HTML y la caché se regeneran y no se versionan.

**Autores:** Samuel Ramirez, Andres Montoya, Emanuel Quintero, Pablo Gil, Sebastian Perez y Pedro Rueda.
