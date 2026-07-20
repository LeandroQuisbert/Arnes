## VERSIÓN 1.7 – DOCUMENTACIÓN DE ARQUITECTURA DEL VISUALIZADOR DE ARNÉS

---

### 1. RESUMEN Y PROPÓSITO DEL SISTEMA

1.1. Este visualizador modela el arnés de una moto eléctrica como un grafo de **componentes físicos** (contenedores, conectores) unidos por **relaciones** (cables fijos, acoples enchufables) y organizados mediante **señales lógicas** (nets).  

1.2. Permite visualizar los elementos, moverlos en modo edición respetando la jerarquía de contención, observar cables flexibles y validar automáticamente la coherencia de géneros, pines y señales.

#### 1.3. Filosofía de Uso y Alcance

1.3.1. Esta aplicación no es un simulador eléctrico ni una herramienta de diseño de circuitos. Es una **plataforma de documentación visual interactiva** para técnicos y mecánicos, pensada para representar el cableado "como si estuviera extendido en el piso del taller".

1.3.2. La herramienta permite tanto **consultar** un arnés existente como **construir y modificar** su representación desde cero. Todas las entidades (contenedores, conectores, cables, acoples y nets) pueden crearse, editarse y eliminarse directamente desde la interfaz gráfica o mediante la carga de archivos de datos.

1.3.3. La relación entre la vista gráfica y los datos subyacentes es **bidireccional**:
- Cualquier cambio realizado en el dibujo (arrastre, redimensionamiento, creación de conexiones) se refleja automáticamente en las tablas de datos.
- Cualquier modificación en los datos (cambio de propiedades, adición de notas, ajuste de pines) actualiza inmediatamente la representación visual.

1.3.4. Los usos principales son:
- **Trazar rutas**: seguir visualmente el camino de una señal desde un pin de origen hasta su destino, incluso si atraviesa varios conectores y cables.
- **Identificar propiedades**: conocer de un vistazo el color, calibre, longitud y tipo de cada cable, así como los pines involucrados en cada conexión.
- **Filtrar y aislar**: ocultar subsistemas completos para centrarse en un circuito específico (por ejemplo, ver solo la ECU y la botonera).
- **Documentar y anotar**: añadir notas con historial a cualquier elemento, dejando registro de modificaciones, observaciones de mantenimiento o decisiones de diseño.
- **Gestionar el arnés completo**: crear, modificar y eliminar componentes, cables y conexiones, manteniendo siempre la coherencia entre la vista visual y la base de datos del proyecto.

1.3.5. En esencia, es un **plano dinámico e interactivo** que funciona como una capa intermedia entre el esquema técnico y la base de datos del arnés, permitiendo que el trabajo de documentación y consulta se realice de forma orgánica y visual.

**Memoria de diseño – Sección 1**  
Se mantiene el propósito original del MVP: ofrecer una vista interactiva del arnés con restricciones físicas realistas. La separación entre componentes, relaciones y señales sigue el patrón modelo-vista, donde los componentes tienen presencia gráfica, las relaciones definen vínculos y los nets permiten validación eléctrica. La decisión de mantener estos tres pilares se tomó para no mezclar responsabilidades y facilitar futuras ampliaciones como simulación de continuidad. La subsección 1.3 clarifica el alcance de la herramienta como documentación visual activa, no como simulador, y explicita la naturaleza bidireccional de la interfaz.

---

### 2. SISTEMA DE IDENTIFICADORES (ÍNDICE DE ENTIDADES)

2.1. Cada entidad del sistema recibe un **ID único** compuesto por una letra de tipo más un número de tres dígitos. La siguiente tabla actúa como índice general de todas las entidades modeladas.

**Tabla 1 – Índice de entidades y prefijos de identificación**

| Prefijo | Entidad                  | Rango numérico       | Ejemplo |
|---------|--------------------------|----------------------|---------|
| T       | Contenedor (system, enclosure, pcb) | 100 – 399 (subdividido) | T100, T200, T300 |
| C       | Conector (connector)     | 001 – 999            | C001    |
| W       | Cable fijo (wired)       | 001 – 999            | W001    |
| M       | Acople enchufable (mated)| 001 – 999            | M001    |
| N       | Señal lógica (net)       | 001 – 999            | N001    |

2.2. La letra **`T`** agrupa todos los contenedores (system, enclosure, pcb) bajo un mismo prefijo. La subdivisión del rango numérico indica el nivel jerárquico:  
 2.2.1. `1xx` → sistema raíz (system).  
 2.2.2. `2xx` → caja (enclosure).  
 2.2.3. `3xx` → placa de circuito (pcb).  

2.3. Esta organización permite hasta 99 contenedores de cada tipo (ej. T101, T201, T301…) y mantiene la coherencia conceptual: todos los contenedores comparten prefijo, pero su posición jerárquica es deducible del número.

2.4. El campo `type` (p. ej. `"system"`, `"enclosure"`, `"pcb"`) sigue presente en los datos para definir explícitamente el subtipo; el ID solo actúa como identificador único.

#### 2.5. Principio de posicionamiento

2.5.1. **Regla general**:  
- Todo componente con `parent_id: null` tiene posición absoluta respecto al lienzo (atributos `x`, `y`).  
- Todo componente con `parent_id` no nulo tiene posición relativa a su padre (atributos `offsetX`, `offsetY` para contenedores; `offset` para conectores).  

2.5.2. En la práctica actual, solo `T100` (system raíz) cumple `parent_id: null`. La regla está enunciada de forma genérica para admitir futuros componentes raíz sin modificar la lógica de posicionamiento.

**Memoria de diseño – Sección 2**  
La tabla única de entidades facilita la consulta rápida. La subdivisión de `T` se explica a continuación, manteniendo la tabla limpia. Se conserva el campo `type` porque la letra `T` por sí sola no distingue entre sistema, caja o placa; el rango numérico da una pista, pero el tipo explícito es necesario para el procesamiento. La decisión de usar centenas como niveles jerárquicos permite un crecimiento holgado (en la práctica rara vez se superan 10 elementos por tipo). Conectores, cables, acoples y nets tienen cada uno su propio prefijo porque representan conceptos fundamentalmente diferentes.  

La nueva subsección 2.5 formaliza el principio de posicionamiento relativo que se aplica en toda la arquitectura: solo la raíz del mundo tiene coordenadas absolutas. Esto evita ambigüedades y centraliza la responsabilidad del cálculo de posiciones en el motor de renderizado.

---

### 3. CONTENEDORES (SYSTEM, ENCLOSURE, PCB)

Son los **contenedores** que definen la estructura física del arnés. No tienen género ni pines; su función es agrupar conectores y limitar su movimiento.

#### 3.1 Atributos comunes

**Tabla 2 – Atributos de contenedores**

| Campo       | Tipo         | Descripción |
|-------------|--------------|-------------|
| id          | string       | T100, T200, T300… |
| type        | string       | `"system"`, `"enclosure"`, `"pcb"` |
| name        | string       | Nombre descriptivo (ej. "Caja 1") |
| parent_id   | string / null| ID del contenedor padre (`null` solo para el sistema raíz) |
| designator  | string       | Etiqueta técnica (ej. "BOX1", "PCB1") |
| position    | object       | **Si `parent_id` es `null`**: `{ x, y, width, height }` en píxeles absolutos.<br>**Si `parent_id` no es `null`**: `{ offsetX, offsetY, width, height }` en píxeles relativos al padre. |
| notes       | array        | Histórico de notas (ver 3.1.1) |

##### 3.1.1 Estructura de notas

Cada entrada del array `notes` es un objeto con el formato:
- `date` (string, ISO 8601): fecha de creación de la nota.
- `user` (string): identificador del usuario que la escribió.
- `text` (string, máximo 500 caracteres): contenido del comentario.

El campo `notes` está disponible en **todas** las entidades (contenedores, conectores, wires, mated, nets). Se concibe como un historial acumulativo de anotaciones técnicas.

#### 3.2 Catálogo de contenedores

**Tabla 3 – Lista de contenedores del ejemplo**

| ID   | Type       | Nombre         | Padre | Designator | Posición (offsetX, offsetY, width, height) | Notas |
|------|------------|----------------|-------|------------|-------------------------------------------|-------|
| T100 | system     | Moto Eléctrica | null  | –          | 50, 50, 2600, 1200 *(absoluto)* | – |
| T200 | enclosure  | Caja 1         | T100  | BOX1       | 90, 110, 950, 1000 | – |
| T201 | enclosure  | Caja 2         | T100  | BOX2       | 1593, 122, 950, 1000 | – |
| T300 | pcb        | PCB 1          | T200  | PCB1       | 80, 100, 368, 837 | – |
| T301 | pcb        | PCB 2          | T201  | PCB2       | 427, 117, 472, 817 | – |

**Memoria de diseño – Sección 3**  
El término "contenedores" agrupa system, enclosure y pcb bajo un mismo concepto. La jerarquía de `parent_id` nulo solo en el sistema raíz evita componentes huérfanos. Los valores de posición han sido convertidos a relativos (offsetX/offsetY) excepto para T100, que mantiene coordenadas absolutas por ser la raíz. Las fórmulas de conversión desde la versión anterior son: `offsetX = x - x_padre`, `offsetY = y - y_padre`. El campo `notes` sigue la misma estructura que en el resto de entidades para mantener homogeneidad.

#### 3.3 Movimiento y redimensionamiento

##### 3.3.1 Movimiento de contenedores

3.3.1.1. Al arrastrar un contenedor, todos sus descendientes se desplazan el mismo vector **(dx, dy) en tiempo real**, sin retardo.  
3.3.1.2. El contenedor arrastrado **no modifica su tamaño** ni ninguna otra propiedad; solo cambian sus valores de posición (`offsetX`/`offsetY`, o `x`/`y` si es T100).  
3.3.1.3. Los conectores anclados (`edgeSide`) se reposicionan automáticamente sobre el borde del padre en el mismo fotograma.

**Memoria de diseño – 3.3.1**  
El movimiento solidario en tiempo real evita parpadeos. Al ser todas las posiciones relativas, mover un contenedor no requiere actualizar las coordenadas de sus descendientes; el renderizado recalcula las posiciones absolutas a partir de los offsets en cada frame. No se modifica el tamaño durante el arrastre porque esa operación tiene su propio modo (redimensionamiento con handles).

##### 3.3.2 Redimensionamiento

3.3.2.1. Los contenedores pueden redimensionarse arrastrando cualquier esquina o borde (handles en todo el perímetro).  
3.3.2.2. Durante el redimensionamiento:  
 a. Los conectores anclados ajustan su posición según la regla definida en **3.3.2.3**.  
 b. Los componentes internos no anclados conservan su distancia relativa al centro geométrico del contenedor.  
3.3.2.3. **Comportamiento de conectores anclados durante el redimensionamiento:**  
 3.3.2.3.1. Un conector anclado **conserva su distancia respecto a la esquina del borde que permanece fija durante el estiramiento**. La esquina fija es aquella que no es arrastrada por el tirador de redimensión.  
 3.3.2.3.2. **Ejemplo concreto** (borde derecho, estiramiento vertical hacia abajo):  
  a. Conector en el borde derecho (`edgeSide: "right"`) de un contenedor.  
  b. Si el contenedor se estira **solo hacia abajo** (aumenta su altura, manteniendo fija la esquina superior izquierda), el borde derecho se alarga.  
  c. La esquina superior derecha **no se mueve**.  
  d. La distancia desde el centro del conector hasta esa esquina superior derecha **permanece constante**.  
  e. **Por tanto, el conector no se desplaza.** El borde se estira por debajo de él, pero su posición absoluta no cambia.  
 3.3.2.3.3. Regla general para cualquier borde:  
  a. Si se estira un eje **paralelo** al borde donde está anclado el conector, se toma como referencia la esquina más cercana que **no** está siendo desplazada por el estiramiento. La distancia a esa esquina se mantiene, por lo que el conector **no se mueve** en la dirección del estiramiento.  
  b. Si el estiramiento es **perpendicular** al borde, el borde mismo no se desplaza lateralmente, así que el conector **no cambia de posición**.  
3.3.2.4. **Posicionamiento de handles en bordes con conectores:**  
 Para evitar interferencias visuales, los handles de redimensionamiento ubicados en un borde que contiene conectores anclados se sitúan automáticamente en el extremo opuesto del borde, dejando libre la zona donde se encuentra el conector. Por ejemplo, si un conector está anclado cerca de la parte superior de un borde derecho, el handle de ese borde aparecerá en la parte inferior.

**Memoria de diseño – 3.3.2**  
Redimensionamiento desde cualquier borde para máxima flexibilidad. La regla de distancia a esquina fija evita comportamientos contraintuitivos que tenía el método porcentual. El nuevo punto 3.3.2.4 resuelve un problema práctico de usabilidad: cuando un conector está cerca del handle, el usuario podía redimensionar sin querer al intentar mover el conector. Al desplazar el handle al extremo opuesto, se elimina esa interferencia.

---

### 4. CONECTORES

Los conectores son los puntos de conexión eléctrica. Cada uno tiene un género y pertenece a un contenedor físico.

#### 4.1 Atributos

**Tabla 4 – Atributos de conectores**

| Campo      | Tipo          | Descripción |
|------------|---------------|-------------|
| id         | string        | C001, C002… |
| type       | string        | `"connector"` |
| name       | string        | Nombre descriptivo |
| parent_id  | string        | ID del contenedor donde está montado |
| designator | string        | Etiqueta técnica (ej. "J1") |
| pins       | number        | Cantidad de pines |
| gender     | string        | `"male"` o `"female"` (obligatorio) |
| edgeSide   | string        | **Obligatorio.** `"left"`, `"right"`, `"top"`, `"bottom"` |
| offset     | number        | Distancia desde el extremo de referencia del borde (ver 4.1.1) |
| position   | object        | `{ width, height }` en píxeles (tamaño del conector, sin coordenadas x,y) |
| matedId    | string / null | ID del acople M al que pertenece, o `null` si está libre |
| notes      | array         | Histórico de notas (ver 3.1.1) |

4.1.1. Definición de `offset` según `edgeSide`:  
 a. Para `"left"` o `"right"`: `offset` es la distancia desde el borde superior del contenedor padre hasta el borde superior del conector.  
 b. Para `"top"` o `"bottom"`: `offset` es la distancia desde el borde izquierdo del contenedor padre hasta el borde izquierdo del conector.

4.1.2. Cálculo de la posición absoluta en runtime:

| `edgeSide` | `x_global` | `y_global` |
|------------|------------|------------|
| `"left"`   | `padre.x` | `padre.y + offset` |
| `"right"`  | `padre.x + padre.width - conector.width` | `padre.y + offset` |
| `"top"`    | `padre.x + offset` | `padre.y` |
| `"bottom"` | `padre.x + offset` | `padre.y + padre.height - conector.height` |

4.1.3. `matedId` vincula el conector con el acople M que lo une a su pareja. Si el conector participa en un M, aquí se almacena el ID de dicho M. Este enfoque sustituye al antiguo `expectedPair`.

###### 4.1.3.1 Migración desde expectedPair

En versiones anteriores se usaba `expectedPair` (ID del conector esperado). A partir de la versión 1.3 se adopta `matedId` (ID del M). La migración consiste en buscar el M que conecta ambos conectores y asignar ese ID. Si no hay M, se deja `null`.

**Memoria de diseño – 4.1**  
El cambio a `matedId` unifica la fuente de verdad: el M es quien define la relación, y el conector simplemente referencia a ese M. Se eliminan los campos `position.x` e `position.y`, reemplazados por un único valor `offset` que, combinado con `edgeSide`, permite calcular la posición global. `edgeSide` pasa a ser obligatorio, eliminando el concepto de conector libre. Todos los conectores están anclados a un borde de su contenedor, reflejando la realidad física de los arneses. La posición de un conector nunca depende de su pareja en un M; cada conector se posiciona exclusivamente por su relación geométrica con su contenedor padre. Esto garantiza que al mover una caja, todos sus conectores se desplacen correctamente con ella.

#### 4.2 Género y validación

4.2.1. Todo conector tiene género obligatorio (`male` o `female`).  
4.2.2. Los acoples **M** requieren géneros opuestos. No importa cuál es `from` o `to`.  
4.2.3. El sistema verificará la coherencia de géneros al cargar o editar.

**Memoria de diseño – 4.2**  
Se eliminó la restricción direccional de género; solo importa que sean complementarios.

#### 4.3 Catálogo completo de conectores

**Tabla 5 – Conectores del ejemplo**

| ID   | Nombre       | Padre | Designator | Pines | Género | EdgeSide | Offset | Tamaño (w, h) | matedId | Notes |
|------|--------------|-------|------------|-------|--------|----------|--------|----------------|---------|-------|
| C001 | Molex 2P     | T300  | J1         | 2     | male   | right    | 100    | 180, 115       | M001    | –     |
| C002 | Molex 2P     | T200  | J2         | 2     | female | right    | 200    | 180, 115       | M001    | –     |
| C003 | GX12         | T200  | J3         | 2     | female | right    | 740    | 180, 115       | M002    | –     |
| C004 | GX12         | T100  | J4         | 2     | male   | left     | 850    | 180, 115       | M002    | –     |
| C005 | GX12 4P      | T100  | J5         | 4     | male   | right    | 850    | 180, 115       | M003    | –     |
| C006 | GX12 4P      | T201  | J6         | 4     | female | left     | 728    | 180, 115       | M003    | –     |
| C007 | Molex 5P     | T201  | J7         | 5     | male   | left     | 188    | 180, 115       | M004    | –     |
| C008 | Molex 5P     | T301  | J8         | 5     | female | left     | 71     | 180, 115       | M004    | –     |
| C009 | Molex 2P     | T301  | J9         | 2     | female | left     | 251    | 180, 115       | null    | [{"date":"2026-07-12","user":"Leo","text":"Reserva para faro auxiliar"}] |

**Memoria de diseño – 4.3**  
Los conectores `C002`, `C004`, `C005` y `C007`, que antes eran libres (`edgeSide: null`), han sido migrados a `edgeSide` según su posición relativa a la caja o al sistema. Por ejemplo, `C004` y `C005` están en `T100` y se ubican en bordes opuestos (`left` y `right`) para representar el latiguillo externo. `C002` se mantiene en el borde derecho de la Caja 1, cerca de `C003`. Los offsets se han calculado con las fórmulas de migración: para bordes verticales, `offset = y - y_padre`; para horizontales, `offset = x - x_padre`. El campo `hidden` ha sido eliminado; la visibilidad se controla con filtros de vista.

#### 4.4 Posicionamiento de conectores

4.4.1. Conector anclado: completamente dentro del contenedor, con la cara de pines exactamente sobre el borde indicado por `edgeSide`.  
4.4.2. Ejemplo: `edgeSide: "right"` → borde derecho del conector toca el borde derecho del contenedor.

#### 4.5 Movimiento de conectores

4.5.1. Todos los conectores están anclados a un borde. Pueden **deslizarse a lo largo del borde** (cambiando su `offset`) o **cambiarse a otro borde** del mismo contenedor si el cursor supera 30 px de distancia perpendicular al borde actual.  
4.5.2. **Propagación rígida del movimiento:**  
 4.5.2.1. Solo los conectores unidos mediante un **M** se mueven solidariamente (todos los M existentes se consideran acoples activos).  
 4.5.2.2. Al arrastrar un conector, se mueve también el otro extremo si comparten el mismo M.  
 4.5.2.3. Los conectores unidos solo por wires no se arrastran entre sí; el cable se redibuja.

---

### 5. CABLES FIJOS (WIRED) — ENTIDAD TIPO W

Representan un conductor físico real (cable soldado o crimpado). No transmiten movimiento mecánico.

#### 5.1 Atributos

**Tabla 6 – Atributos de un wire**

| Campo     | Tipo   | Descripción |
|-----------|--------|-------------|
| id        | string | W001, W002… |
| type      | string | `"wired"` |
| from      | object | `{ connector: "C002", pin: 1 }` |
| to        | object | `{ connector: "C003", pin: 1 }` |
| net       | string | ID del net que transporta (obligatorio) |
| length    | number | Longitud real en mm (informativo) |
| gauge     | number | Sección AWG/mm² (opcional) |
| color     | string | Color de línea (defecto: azul) |
| thickness | number | Grosor en px (opcional) |
| notes     | array  | Histórico de notas |

5.1.1. La ubicación física se deduce automáticamente del ancestro común más bajo de los dos conectores extremos.

**Memoria de diseño – 5.1**  
Se eliminó `parent_id` para evitar inconsistencias en cables que cruzan contenedores.

#### 5.2 Tabla de wires del ejemplo

**Tabla 7 – Wires del arnés**

| ID   | From (C, pin) | To (C, pin) | Net  | Longitud | Descripción |
|------|---------------|-------------|------|----------|-------------|
| W001 | C002, 1       | C003, 1     | N001 | 150 mm   | Cable interno Caja 1 |
| W002 | C004, 1       | C005, 1     | N001 | 400 mm   | Latiguillo externo |
| W003 | C006, 1       | C007, 2     | N001 | 180 mm   | Cable interno Caja 2 |

#### 5.3 Visualización y comportamiento

5.3.1. Línea azul continua y flexible entre los dos puntos.  
5.3.2. Siempre por encima de cajas y conectores.  
5.3.3. Se redibuja como curva adaptable al mover un extremo.  
5.3.4. La línea se acorta/estira libremente; `length` es solo informativo.

---

### 6. ACOPLES ENCHUFABLES (MATED) — ENTIDAD TIPO M

Define una unión macho‑hembra entre dos conectores. Unión rígida: los conectores se mueven juntos, sin línea visual adicional.

#### 6.1 Atributos

**Tabla 8 – Atributos de un mated**

| Campo      | Tipo   | Descripción |
|------------|--------|-------------|
| id         | string | M001, M002… |
| type       | string | `"mated"` |
| from       | object | `{ connector: "C001", pin: 1 }` |
| to         | object | `{ connector: "C002", pin: 1 }` |
| net        | string | ID del net que transporta |
| pinMapping | string / null | `"direct"` (pin a pin igual), `"reversed"` (invertido), o `null` si no se especifica (opcional) |
| notes      | array  | Histórico de notas |

**Importante:** A partir de la versión 1.6, un M listado en el array de conexiones **siempre representa un acople conectado**. Se eliminan los estados `planned`, `disconnected` y `obsolete`. La presencia del M es condición necesaria y suficiente para considerar la unión activa.

#### 6.2 Validaciones

6.2.1. Los dos conectores deben tener géneros opuestos.  
6.2.2. Si un conector tiene `matedId`, ese ID debe coincidir con el M que lo incluye.  
6.2.3. El sistema verificará la coherencia al cargar: cada M debe ser referenciado por sus dos conectores.  
6.2.4. **Mapeo de pines:**  
 a. Si `pinMapping` es `"direct"`, el acople respeta el orden natural: pin 1 con pin 1, pin 2 con pin 2, etc. El sistema validará que los pines declarados en `from` y `to` cumplan esta correspondencia.  
 b. Si `pinMapping` es `"reversed"`, el orden se invierte: pin 1 con pin N, pin 2 con N‑1, etc., siendo N el número de pines del conector más pequeño.  
 c. Si `pinMapping` es `null` o no se define, no se aplica esta validación automática.  
 d. Si los conectores tienen distinto número de pines y `pinMapping` es `"direct"` o `"reversed"`, el sistema emitirá una advertencia informativa indicando cuántos pines del conector más grande quedan sin asignar en este acople.

#### 6.3 Tabla de mated del ejemplo

**Tabla 9 – Acoples mated**

| ID   | From (C, pin) | To (C, pin) | Net  | pinMapping | Descripción |
|------|---------------|-------------|------|------------|-------------|
| M001 | C001, 1       | C002, 1     | N001 | direct     | J1‑J2 (2 pines, 1 a 1) |
| M002 | C004, 1       | C003, 1     | N001 | direct     | J4‑J3 (2 pines, 1 a 1) |
| M003 | C005, 1       | C006, 1     | N001 | direct     | J5‑J6 (4 pines, por ahora solo se usa pin 1) |
| M004 | C007, 2       | C008, 2     | N001 | null       | J7‑J8 (conexión explícita pin 2‑2) |

**Memoria de diseño – Sección 6**  
La simplificación de los estados de M responde a la realidad del MVP: no se ha implementado ninguna funcionalidad que requiera distinguir entre acoples planeados, desconectados u obsoletos. Mantener esos estados generaba complejidad innecesaria y posibles incoherencias con `matedId`. Si en el futuro se necesita modelar otros estados, se reintroducirán con un enfoque más robusto.

#### 6.4 Visualización

6.4.1. Sin línea; conectores enfrentados borde con borde.  
6.4.2. La unión rígida se manifiesta en el movimiento solidario.

---

### 7. BLOQUEO DINÁMICO (`lockedWith`)

#### 7.1 Generación en runtime

7.1.1. `lockedWith` no se almacena; se calcula a partir de todos los M existentes (todos son activos por definición).  
7.1.2. Cada M hace que ambos conectores se incluyan mutuamente en `lockedWith`.  
7.1.3. Los wires no contribuyen.

#### 7.2 Cadenas rígidas

7.2.1. En el ejemplo:  
 a. C001‑C002 (M001)  
 b. C003‑C004 (M002)  
 c. C005‑C006 (M003)  
 d. C007‑C008 (M004)  
7.2.2. Los wires unen estas cadenas pero no las solidarizan mecánicamente.

---

### 8. SEÑALES LÓGICAS (NET) — ENTIDAD TIPO N

Un net es una señal eléctrica declarada una vez. Wires y mateds lo referencian; la topología emerge de los puntos compartidos.

#### 8.1 Atributos

**Tabla 10 – Atributos de un net**

| Campo      | Tipo   | Descripción |
|------------|--------|-------------|
| id         | string | N001, N002… |
| name       | string | Nombre descriptivo |
| type       | string | `"power"`, `"ground"`, `"data"`, etc. |
| voltage    | string | Tensión nominal (opcional) |
| pair       | string | ID del net complementario (diferencial) |
| notes      | array  | Histórico de notas |

#### 8.2 Tabla de nets del ejemplo

**Tabla 11 – Nets definidos**

| ID   | Nombre               | Type  | Voltage |
|------|----------------------|-------|---------|
| N001 | Señal PCB1 → PCB2    | data  | –       |

#### 8.3 Construcción automática del grafo

8.3.1. El sistema examina wires y mateds con el mismo net y construye un grafo de nodos (conector, pin).  
8.3.2. La ruta del ejemplo: C001.pin1 – C002.pin1 – C003.pin1 – C004.pin1 – C005.pin1 – C006.pin1 – C007.pin2 – C008.pin2.

#### 8.4 Uso de nets

8.4.1. Verificar continuidad eléctrica.  
8.4.2. Validar coherencia de señales.  
8.4.3. Resaltado visual al seleccionar un net.  
8.4.4. Futuro: puntos de chasis implícitos.

---

### 9. ÁRBOL DE CONTENCIÓN (JERARQUÍA FÍSICA)

#### 9.1 Diagrama jerárquico

```
T100 (Moto)
├── T200 (Caja 1)
│   ├── T300 (PCB 1)
│   │   └── C001 (J1, male, anclado right)
│   ├── C002 (J2, female, anclado right)
│   └── C003 (J3, female, anclado right)
├── T201 (Caja 2)
│   ├── T301 (PCB 2)
│   │   ├── C008 (J8, female, anclado left)
│   │   └── C009 (J9, female, anclado left)
│   ├── C006 (J6, female, anclado left)
│   └── C007 (J7, male, anclado left)
├── C004 (J4, male, anclado left)
└── C005 (J5, male, anclado right)
```

#### 9.2 Ubicación efectiva de los wires

9.2.1. W001: dentro de T200.  
9.2.2. W002: en T100.  
9.2.3. W003: en T201.

---

### 10. REGLAS DE CONSISTENCIA Y VALIDACIONES

10.1. **Género en M:** géneros opuestos obligatorios.  
10.2. **Integridad de matedId:** si un conector tiene `matedId`, ese M debe existir, contener al conector, y ambos conectores del M deben tener el mismo `matedId`.  
10.3. **Obligatoriedad de edgeSide:** todo conector debe tener `edgeSide` definido (`"left"`, `"right"`, `"top"`, `"bottom"`). No se admite valor `null`.  
10.4. **Pines válidos:** los pines referenciados deben existir en los conectores.  
10.5. **Longitud de wire:** `length` es informativo, sin restricción.  
10.6. **Compatibilidad jerárquica:** dos conectores pueden conectarse si comparten un ancestro contenedor común.  
10.7. **Continuidad de nets:** el grafo del net debe ser conexo; se detectan conflictos de señales.  
10.8. **Mapeo de pines (opcional):** si `pinMapping` es `"direct"` o `"reversed"`, los pines `from` y `to` deben cumplir el orden declarado. Si los conectores tienen distinto número de pines, se emitirá una advertencia informativa sobre los pines no asignados.

**Memoria de diseño – 10.2**  
La validación de `matedId` mantiene la coherencia del modelo. Con la eliminación del campo `status`, todos los M son activos, lo que simplifica la verificación: basta con que el M exista y sea referenciado. La nueva regla 10.3 formaliza la obligatoriedad de `edgeSide`, cerrando la posibilidad de conectores sin ubicación definida.

---

### 11. VISUALIZACIÓN GENERAL

11.1. **Cables fijos:** líneas azules curvas, siempre visibles.  
11.2. **Acoples M:** sin línea; conectores enfrentados.  
11.3. **Conectores anclados:** dentro del contenedor, pines sobre el borde.  
11.4. **Panel de configuración y modo edición:**  
 11.4.1. Existe un botón de configuración (ícono de engranaje) en la barra superior.  
 11.4.2. Al pulsarlo se despliega un panel que contiene, entre otras opciones, el control de **modo edición** y la **opción de rendimiento** para el recálculo de posiciones (continuo o al finalizar el movimiento).  
 11.4.3. El modo edición se activa/desactiva con un interruptor en ese panel, o mediante el atajo **Ctrl+Shift+E**.  
 11.4.4. Por defecto, el sistema arranca en modo solo lectura.  
 11.4.5. En modo edición se habilitan las interacciones de arrastre y redimensionamiento.  
11.5. **Redimensionamiento:** handles en bordes y esquinas; los conectores anclados permanecen fijos respecto a la esquina de referencia (ver 3.3.2.3). Los handles en bordes con conectores se sitúan en el extremo opuesto para evitar interferencias (ver 3.3.2.4).  
11.6. **Resaltado de nets:** al seleccionar un net, sus wires cambian de color.  
11.7. **Filtros de vista:** la ocultación de conectores o subsistemas se realiza mediante filtros dinámicos en la interfaz. El estado de los filtros se guarda automáticamente en `localStorage` del navegador vinculado al identificador del proyecto, restaurándose al recargar la página. Existe un control para restablecer todos los filtros.

---

### 12. NOTAS PARA FUTURAS VERSIONES

12.1. Configuración de modo de arranque (lectura/edición) en preferencias.  
12.2. Enrutamiento automático de wires con evasión de obstáculos.  
12.3. Tamaño visual de conectores adaptable a pines o designator.  
12.4. Panel de validación global.  
12.5. Soportar pares trenzados (campo `pair` en nets).  
12.6. Puntos de chasis como nodos de tierra implícitos.  
12.7. Integración con sistemas Kanban para seguimiento de tareas basadas en notas.  
12.8. En caso de requerirse, reintroducción de estados en M (planificado, desconectado, obsoleto) con un modelo más robusto de gestión de ciclo de vida de conexiones.  
12.9. **Sistema de registro de eventos (logs):** para visualizar incidencias como conectores no enfrentados en un M, advertencias de validación y acciones del usuario, facilitando el diagnóstico y la auditoría.

---

### 13. HISTORIAL DE VERSIONES

**Versión 1.0** – Documentación inicial con sistema de IDs, jerarquía de contenedores, conectores con expectedPair, cálculo dinámico de lockedWith, nets como etiquetas.

**Versión 1.1** – Mejoras estructurales: tabla única de entidades, sección "Contenedores".

**Versión 1.2** – Modo edición con toggle (Ctrl+Shift+E), eliminación de estados explícitos en conectores, adición de campos `notes` y `hidden`.

**Versión 1.3** – Refactorización mayor: sustitución de `expectedPair` por `matedId`, estructura histórica de notas, panel de configuración, corrección de numeración duplicada.

**Versión 1.4** – Adición de subsección 1.3 "Filosofía de Uso y Alcance" para clarificar que la herramienta es una plataforma de documentación visual interactiva, con capacidades de creación y edición completas y sincronización bidireccional entre vista gráfica y datos.

**Versión 1.5** – Corrección de errores de coherencia y adición de funcionalidad: regla de redimensionamiento corregida, coordenadas de conectores ajustadas, árbol jerárquico actualizado, campo opcional `pinMapping` en M.

**Versión 1.6** – Simplificación del modelo: eliminación del campo `hidden` en conectores y de los estados en M.

**Versión 1.7** – Refactorización del sistema de posicionamiento:  
- Coordenadas relativas para todos los componentes excepto aquellos con `parent_id: null` (solo T100 en la práctica actual).  
- Eliminación del concepto de conector libre: `edgeSide` pasa a ser obligatorio para todos los conectores.  
- Eliminación de los campos `position.x`/`position.y` en conectores, reemplazados por `offset`.  
- Contenedores no raíz almacenan `offsetX`/`offsetY` en lugar de `x`/`y` absolutos.  
- Migración automática desde archivos V1.6.  
- Nuevas validaciones: `edgeSide` obligatorio, consistencia estricta de `matedId`.  
- Advertencia de `pinMapping` asimétrico.  
- Eliminación de la terminología "M activo".  
- Persistencia de filtros de vista en `localStorage`.  
- Opción de rendimiento en panel de configuración.  
- Handles de redimensionamiento adaptativos en bordes con conectores.  
- Nota futura sobre sistema de logs.
