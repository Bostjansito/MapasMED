# Rutas en Medellín según la hora del día

Fundamentos de IA · Semestre 6

Notebook: **`Ruteo_Multiobjetivo_Medellin.ipynb`**

Una misma ruta no es igual de buena a las 8 de la mañana que a las 11 de la noche.
El proyecto construye el mapa de calles de Medellín, le pega a cada calle qué tan insegura
es en cada franja del día, y busca rutas con los algoritmos de búsqueda vistos en clase.

## Cómo está armado

1. **Todo en metros.** El mapa se proyecta a EPSG:3116, así que las distancias son
   euclidianas: `√((x₂-x₁)² + (y₂-y₁)²)`. No hay ni una fórmula de la esfera.
2. **Dos tablas.** Los nodos en un DataFrame y las aristas en otro. Toda la información
   se le agrega a la tabla de aristas.
3. **Un peso simple.** `costo = distancia × (1 + 4·inseguridad − 0.2·tiene_camara)`.
   En una frase: *una calle insegura se siente más larga; una con cámara, un poco más corta.*
4. **Algoritmos escritos a mano.** Costo uniforme, primero el mejor y A\*, sin `networkx`.

## Las franjas

| Franja | Horas | Robos (2019+) |
|---|---|---|
| Mañana | 06:00 – 11:59 | 44.472 |
| Tarde | 12:00 – 18:59 | 51.150 |
| Noche | 19:00 – 05:59 | 74.834 |

Cambiar `FRANJA_ACTUAL` en la sección 11 para rutear a otra hora.

## Resultados de la corrida

Mapa: 18.314 nodos, 43.100 aristas.

**San Javier → Parque de El Poblado, por franja:**

| Franja | Distancia | Tiempo | Inseguridad media |
|---|---|---|---|
| Mañana | 9,09 km | 22,3 min | 0,531 |
| Tarde | 9,21 km | 22,5 min | 0,506 |
| Noche | 11,37 km | 22,8 min | 0,392 |

De noche el algoritmo da un rodeo de 2,3 km para bajar la inseguridad un 26 %. Las tres
franjas dan rutas distintas, que era lo que había que demostrar.

**Los tres algoritmos, mismo problema:**

| Algoritmo | Nodos expandidos | Costo | ¿Óptimo? |
|---|---|---|---|
| Costo uniforme | 14.953 | 28.717,7 | Sí |
| A\* | 11.492 | 28.717,7 | Sí |
| Primero el mejor | 167 | 36.234,6 | No, un 26 % peor |

A\* llega al mismo óptimo que costo uniforme abriendo un 23 % menos de nodos. Primero el
mejor abre 90 veces menos que A\* pero devuelve una ruta peor: el ejemplo de libro de por
qué la heurística sola no basta.

**Por qué el peso de inseguridad es 4 y no 1** (sección 11.1):

| Peso | Distancia | Inseguridad |
|---|---|---|
| 0 | 9,03 km | 0,569 |
| 1 | 9,03 km | 0,568 |
| 2 | 9,03 km | 0,568 |
| 4 | 11,37 km | 0,392 |
| 8 | 11,49 km | 0,385 |

Hasta 2 la ruta ni se mueve: no le compensa desviarse. Es un resultado medido, no una
suposición.

## Lo que genera en `salidas/`

| Archivo | Qué es |
|---|---|
| **`ruteador.html`** | **Mini front interactivo.** Se abre con doble clic, se hace clic en el mapa para poner origen y destino, y calcula la ruta al instante. Se puede cambiar la franja, el criterio y el algoritmo sin recargar. |
| `mapa_seguridad.html` | Mapa de calor: zonas inseguras por franja, vías pintadas según su peligro y las cámaras. |
| `rutas.html` | Las tres rutas comparadas del notebook. |

El ruteador lleva **A\* escrito en JavaScript** dentro del propio archivo: el notebook vuelca
el grafo a JSON (18.314 nodos, 43.100 aristas) y lo incrusta en la página. Pesa 1,7 MB y
funciona sin Python. Resuelve una ruta en menos de 30 ms.

Está verificado contra la implementación de Python con Node: mismas rutas, mismos tramos y el
mismo óptimo en los tres algoritmos. Las diferencias de costo (0,07 %) vienen de redondear las
coordenadas a 5 decimales al exportar.

## Estructura

```
ProyectoWaze/
├── Ruteo_Multiobjetivo_Medellin.ipynb
├── README.md
├── datos/          <- los 6 CSV de entrada
├── referencia/     <- notebook de referencia del curso
├── cache/          <- mapa de OSM descargado (se crea solo)
└── salidas/        <- ruteador.html, mapa_seguridad.html, rutas.html
```

## Los datos no están en el repositorio

**Ningún CSV se versiona.** Hay que bajarlos y ponerlos en `datos/` con estos nombres exactos
antes de ejecutar el notebook:

| Archivo | Tamaño | Fuente |
|---|---|---|
| `hurto_a_persona.csv` | 118,7 MB | Datos Abiertos de Medellín |
| `velocidad_e_intensidad_vehicular_en_medellin.csv` | 75,7 MB | Datos Abiertos de Medellín |
| `hurto_de_moto.csv` | 30,5 MB | Datos Abiertos de Medellín |
| `hurto_de_carro.csv` | 13,7 MB | Datos Abiertos de Medellín |
| `hurto_por_pirateria_terrestre.csv` | 0,8 MB | Datos Abiertos de Medellín |
| `202608091356-ansv-fotodeteccion.csv` | 0,4 MB | Agencia Nacional de Seguridad Vial |

> `hurto_a_persona.csv` pesa 118,7 MB y GitHub **rechaza** cualquier archivo de más de
> 100 MB. No es un aviso, es un bloqueo: el `push` falla.

Mientras tanto se puede abrir `salidas/ruteador.html`, que no necesita los CSV para nada:
lleva el grafo ya calculado dentro.

## Cómo ejecutarlo

1. Abrir el notebook desde esta carpeta.
2. Ejecutar la **primera celda**: instala lo que falte y verifica las 12 dependencias.
3. **Si dice que reinicies el kernel, hazlo.** osmnx decide si puede dibujar en el momento
   en que se importa; si se importa antes que matplotlib, se queda sin dibujo toda la sesión.
4. Ejecutar todo. La primera vez descarga el mapa de OpenStreetMap (unos minutos); después
   queda en `cache/` y arranca al instante.

### Si los mapas salen en blanco

Sale el aviso *"Make this Notebook Trusted to load map"*. Jupyter no ejecuta el JavaScript de
un notebook sin firmar:

```bash
python -m nbformat.sign "Ruteo_Multiobjetivo_Medellin.ipynb"
```

Cerrar y reabrir. Alternativa que siempre funciona: abrir `salidas/rutas.html` en el navegador.

## Uso

```python
camino, resumen, mapa = buscar_ruta(
    "Universidad de Antioquia",
    "Centro Comercial Santafé",
    franja="noche",              # manana | tarde | noche
    criterio="seguridad",        # distancia | tiempo | seguridad
    algoritmo="a_estrella",      # a_estrella | costo_uniforme | primero_el_mejor
)
mapa
```

## Trampas de los datos

* `velocidad_e_intensidad_vehicular_en_medellin.csv` tiene las columnas `longitud` y `latitud`
  en **metros EPSG:3116**, no en grados. Como el proyecto trabaja en ese mismo sistema, entran
  directo. Usa además **coma decimal**.
* De las cámaras solo se cuentan las de **Medellín** y en estado *Operando*, *Autorizada
  instalar* o *Prorrogada*.
* La inseguridad se escala con el **percentil 95**, no con el máximo: el centro es un caso
  tan extremo que si se dividiera por el máximo el resto de la ciudad quedaría indistinguible.
* Contar robos cerca de una calle mide **exposición**, no solo peligro. Por el centro pasa
  muchísima gente y por eso acumula más denuncias.
* Los datos de velocidad son de 2020, en pandemia, y solo de 93 días (julio a septiembre).
* **La proyección estira un poco las distancias.** EPSG:3116 tiene su meridiano central en
  −74,08 y Medellín está a unos 165 km de ahí, así que las medidas salen hasta un **0,155 %**
  largas. En 9.335 de 43.100 calles eso hace que la línea recta supere a la distancia real de
  OSM, lo cual rompería formalmente la garantía de A\*. Es despreciable en la práctica, pero
  la heurística lleva un factor de 0,998 para cerrar el hueco. Está medido en el notebook,
  no supuesto.
* Las cámaras de fotodetección **no vigilan hurtos**: detectan exceso de velocidad y semáforo
  en rojo. Que disuadan atracos es una hipótesis nuestra. Conviene decirlo en la sustentación.

## Pendiente

* El `.gitignore` excluye `salidas/*.html`. Si se quieren ver los mapas en GitHub, quitar
  esa línea (aunque GitHub no renderiza HTML, solo lo muestra como código).
* Afinar el peso con parámetros ajustables por criterio, en vez de un solo número.
