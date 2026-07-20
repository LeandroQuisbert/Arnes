## VERSIÓN 1.9.1 – DOCUMENTACIÓN DE ARQUITECTURA DEL VISUALIZADOR DE ARNÉS

---

### 1. RESUMEN Y PROPÓSITO DEL SISTEMA

1.1. Este visualizador modela el arnés de una moto eléctrica como un grafo de **componentes físicos** (contenedores, conectores) unidos por **relaciones** (cables fijos, acoples enchufables) y organizados mediante **señales lógicas** (nets).  

1.2. Permite visualizar los elementos, moverlos en modo edición respetando la jerarquía de contención, observar cables flexibles y validar automáticamente la coherencia de géneros, pines, señales y enfrentamiento de conectores.

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

#### 1.4. Flujo de trabajo del técnico

1.4.1. La interfaz ofrece dos vistas complementarias y sincronizadas: el **lienzo gráfico 2D** y las **tablas de datos**. El usuario puede alternar entre ambas o trabajar con las dos simultáneamente. Cualquier cambio realizado en una vista se propaga instantáneamente a la otra.

1.4.2. **Panel de propiedades desplegable**: al hacer clic en cualquier elemento del lienzo (cable, conector, contenedor, M, N) se abre un panel lateral que muestra todos sus atributos. Este panel permite la edición directa de los campos. En modo solo lectura, los campos aparecen bloqueados; en modo edición, se habilitan para su modificación.

1.4.3. **Filtros rápidos**: el técnico puede aislar visualmente un cable, un net o un conjunto de conectores escribiendo su identificador o nombre en el cuadro de filtro. Por ejemplo, al escribir "W001" se resalta ese cable y se atenúa el resto, permitiendo seguir su ruta completa de extremo a extremo. El estado del filtro se conserva entre sesiones (ver 11.7).

1.4.4. **Persistencia del contexto**: los filtros aplicados, la posición del lienzo y el nivel de zoom se almacenan localmente para que el técnico recupere exactamente la vista en la que estaba trabajando.

**Memoria de diseño – Sección 1**  
Se mantiene el propósito original del MVP: ofrecer una vista interactiva del arnés con restricciones físicas realistas. La separación entre componentes, relaciones y señales sigue el patrón modelo-vista. La subsección 1.4 detalla el flujo de trabajo concreto del técnico. El panel de propiedades, los filtros y la doble vista son los mecanismos que materializan la filosofía de documentación visual activa.

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
- Todo componente con `parent_id` no nulo tiene posición relativa a su padre (atributos `offsetX`, `offsetY` para contenedores; `offset` para conectores fijos; los conectores volantes no almacenan posición propia).  

2.5.2. En la práctica actual, solo `T100` (system raíz) cumple `parent_id: null`. La regla está enunciada de forma genérica para admitir futuros componentes raíz sin modificar la lógica de posicionamiento.

2.5.3. Las dimensiones (`width`, `height`) se almacenan siempre en un objeto `size` separado de la posición, tanto para contenedores como para conectores. Esto diferencia conceptualmente la ubicación del size y facilita el mantenimiento del código.

**Memoria de diseño – Sección 2**  
La tabla única de entidades facilita la consulta rápida. Se conserva el campo `type` porque la letra `T` por sí sola no distingue entre sistema, caja o placa. La decisión de usar centenas como niveles jerárquicos permite un crecimiento holgado. La separación de `position` y `size` responde a la necesidad de tratar independientemente la ubicación y las dimensiones, siguiendo estándares de la industria.

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
| position    | object       | **Si `parent_id` es `null`**: `{ x, y }` en píxeles absolutos.<br>**Si `parent_id` no es `null`**: `{ offsetX, offsetY }` en píxeles relativos al padre. |
| size        | object       | `{ width, height }` en píxeles |
| notes       | array        | Histórico de notas (ver 3.1.1) |

##### 3.1.1 Estructura de notas

Cada entrada del array `notes` es un objeto con el formato:
- `date` (string, ISO 8601): fecha de creación de la nota.
- `user` (string): identificador del usuario que la escribió.
- `text` (string, máximo 500 caracteres): contenido del comentario.

El campo `notes` está disponible en **todas** las entidades (contenedores, conectores, wires, mated, nets). Se concibe como un historial acumulativo de anotaciones técnicas.

#### 3.2 Catálogo de contenedores

**Tabla 3 – Lista de contenedores del ejemplo**

| ID   | Type       | Nombre         | Padre | Designator | Position (offsetX, offsetY) | Size (w, h) | Notas |
|------|------------|----------------|-------|------------|-----------------------------|--------------|-------|
| T100 | system     | Moto Eléctrica | null  | –          | x:50, y:50 *(absoluto)* | 2600, 1200 | – |
| T200 | enclosure  | Caja 1         | T100  | BOX1       | 90, 110 | 950, 1000 | – |
| T201 | enclosure  | Caja 2         | T100  | BOX2       | 1593, 122 | 950, 1000 | – |
| T300 | pcb        | PCB 1          | T200  | PCB1       | 80, 100 | 368, 837 | – |
| T301 | pcb        | PCB 2          | T201  | PCB2       | 427, 117 | 472, 817 | – |

**Memoria de diseño – Sección 3**  
El término "contenedores" agrupa system, enclosure y pcb bajo un mismo concepto. La jerarquía de `parent_id` nulo solo en el sistema raíz evita componentes huérfanos. Los valores de posición han sido convertidos a relativos (offsetX/offsetY) excepto para T100, que mantiene coordenadas absolutas por ser la raíz. La separación de `position` y `size` clarifica la estructura de datos.

#### 3.3 Movimiento y redimensionamiento

##### 3.3.1 Movimiento de contenedores

3.3.1.1. Al arrastrar un contenedor, todos sus descendientes se desplazan el mismo vector **(dx, dy) en tiempo real**, sin retardo.  
3.3.1.2. El contenedor arrastrado **no modifica su size** ni ninguna otra propiedad; solo cambian sus valores de posición (`offsetX`/`offsetY`, o `x`/`y` si es T100).  
3.3.1.3. Los conectores fijos (`mountType: "fixed"`) se reposicionan automáticamente sobre el borde del padre en el mismo fotograma. Los conectores volantes (`mountType: "flying"`) no están anclados cinemáticamente a su contenedor: su posición se recalcula en cada frame a partir de la posición del conector fijo al que están enchufados. Por tanto, un volante se mueve únicamente si el fijo se mueve (por arrastre directo o por movimiento del contenedor que contiene al fijo). Si el contenedor del volante se mueve pero el fijo no, el volante permanece en su sitio, lo que puede provocar que quede fuera de los límites de su contenedor (ver 12.9).

**Memoria de diseño – 3.3.1**  
El movimiento solidario en tiempo real evita parpadeos. Al ser todas las posiciones relativas, mover un contenedor no requiere actualizar las coordenadas de sus descendientes; el renderizado recalcula las posiciones absolutas a partir de los offsets en cada frame. No se modifica el size durante el arrastre porque esa operación tiene su propio modo (redimensionamiento con handle). La excepción para conectores volantes se introdujo en V1.9 para reflejar la realidad física: el extremo de un cable no está atornillado a la caja, sino que se mueve con el conector al que está enchufado. La redacción se refinó en V1.9.1 para eliminar cualquier ambigüedad.

##### 3.3.2 Redimensionamiento

3.3.2.1. Los contenedores pueden redimensionarse arrastrando **exclusivamente la esquina inferior derecha**.  
3.3.2.2. Durante el redimensionamiento:  
 a. Los conectores fijos (`mountType: "fixed"`) conservan su distancia a la esquina superior izquierda del contenedor, que es la esquina fija durante el estiramiento.  
 b. Los conectores volantes (`mountType: "flying"`) no se ven afectados por el redimensionamiento de su contenedor padre; su posición se recalcula a partir de su pareja fija en el M.

**Memoria de diseño – 3.3.2**  
La restricción del redimensionamiento a una única esquina (inferior derecha) simplifica la implementación y evita los problemas de redimensionamiento inverso detectados en versiones anteriores. La esquina superior izquierda actúa como referencia fija, y el offset de los conectores fijos se mantiene respecto a esa esquina. Esta decisión de diseño prioriza la estabilidad y la simplicidad frente a la flexibilidad total, que añadía complejidad sin un beneficio práctico significativo para el técnico.

---

### 4. CONECTORES

Los conectores son los puntos de conexión eléctrica. Cada uno tiene un género, un tipo de montaje y pertenece a un contenedor físico.

#### 4.1 Atributos

**Tabla 4 – Atributos de conectores**

| Campo      | Tipo          | Descripción |
|------------|---------------|-------------|
| id         | string        | C001, C002… |
| type       | string        | `"connector"` |
| name       | string        | Nombre descriptivo |
| parent_id  | string        | ID del contenedor donde está ubicado físicamente |
| designator | string        | Etiqueta técnica (ej. "J1") |
| pins       | number        | Cantidad de pines |
| gender     | string        | `"male"` o `"female"` (obligatorio) |
| mountType  | string        | **Obligatorio.** `"fixed"` (atornillado/soldado al contenedor) o `"flying"` (extremo de cable, se mueve con su pareja) |
| edgeSide   | string        | **Obligatorio.** `"left"`, `"right"`, `"top"`, `"bottom"` |
| offset     | number / null | Distancia desde el extremo de referencia del borde (ver 4.1.2). Solo para `mountType: "fixed"`. `null` en volantes. |
| size       | object        | `{ width, height }` en píxeles (size del conector) |
| matedId    | string / null | ID del acople M al que pertenece, o `null` si está libre |
| notes      | array         | Histórico de notas (ver 3.1.1) |

4.1.1. `mountType` define la cinemática del conector:  
 a. **Fijo (`"fixed"`)**: el conector está atornillado o soldado a su contenedor padre. Su posición se calcula a partir de `edgeSide` y `offset`. Al mover el contenedor, el conector se mueve con él.  
 b. **Volante (`"flying"`)**: el conector es el extremo de un cable. No está fijado mecánicamente a su contenedor; su `parent_id` es declarativo/organizativo (indica en qué caja está físicamente el cable, permite calcular el ancestro común para los wires y valida la compatibilidad jerárquica), pero no influye en su posición. Esta se calcula a partir del conector fijo al que está enchufado (su pareja en el M). Al mover el contenedor padre, el conector volante **no se desplaza** con él; sigue a su pareja fija. Si como resultado queda fuera de los límites de su contenedor, el sistema registra una advertencia en el futuro sistema de logs (ver 12.9), pero no bloquea la acción.

4.1.2. Definición de `offset` (solo para `mountType: "fixed"`):  
 a. Para `"left"` o `"right"`: `offset` es la distancia desde el borde superior del contenedor padre hasta el borde superior del conector.  
 b. Para `"top"` o `"bottom"`: `offset` es la distancia desde el borde izquierdo del contenedor padre hasta el borde izquierdo del conector.

4.1.3. Cálculo de la posición absoluta en runtime para conectores **fijos**:

| `edgeSide` | `x_global` | `y_global` |
|------------|------------|------------|
| `"left"`   | `padre.x` | `padre.y + offset` |
| `"right"`  | `padre.x + padre.width - conector.width` | `padre.y + offset` |
| `"top"`    | `padre.x + offset` | `padre.y` |
| `"bottom"` | `padre.x + offset` | `padre.y + padre.height - conector.height` |

Donde `padre.width` y `padre.height` provienen del `size` del contenedor padre, y `conector.width` y `conector.height` del `size` del conector.

**Nota importante:** `padre.x` y `padre.y` en las fórmulas anteriores son las coordenadas absolutas del contenedor padre, calculadas recursivamente a partir de su `position` y las de sus ancestros. No deben confundirse con los valores `offsetX`/`offsetY` almacenados en el padre.

4.1.4. Cálculo de la posición absoluta para conectores **volantes**:  
 a. Se localiza el conector fijo con el que comparte `matedId`.  
 b. Se aplica la orientación opuesta: si el fijo es `"right"`, el volante se coloca con su borde izquierdo tocando el borde derecho del fijo. Si el fijo es `"left"`, el volante se coloca con su borde derecho tocando el borde izquierdo del fijo. Análogo para `"top"` y `"bottom"`.  
 c. La coordenada perpendicular se alinea para que los pines coincidan (ver 10.10).

4.1.5. `matedId` vincula el conector con el acople M que lo une a su pareja. Si el conector participa en un M, aquí se almacena el ID de dicho M.

**Memoria de diseño – 4.1**  
La introducción de `mountType` en V1.9 resuelve el conflicto cinemático detectado en la auditoría de la V1.8.2: un conector en el extremo de un cable no está fijado a su caja, por lo que no debe moverse con ella si su pareja está en otro contenedor. Esta distinción refleja fielmente la realidad física. Los conectores volantes no almacenan `offset`; su posición es siempre derivada de su pareja fija. La advertencia por quedar fuera del contenedor padre no bloquea la operación (el usuario puede luego agrandar la caja manualmente). En V1.9.1 se añadió la aclaración explícita de que el `parent_id` de un volante es declarativo: no afecta a su posición, pero es necesario para el cálculo del ancestro común de los wires y para la validación jerárquica.

#### 4.2 Género y validación

4.2.1. Todo conector tiene género obligatorio (`male` o `female`).  
4.2.2. Los acoples **M** requieren géneros opuestos. No importa cuál es `from` o `to`.  
4.2.3. El sistema verificará la coherencia de géneros al cargar o editar.

#### 4.3 Catálogo completo de conectores

**Tabla 5 – Conectores del ejemplo**

| ID   | Nombre       | Padre | Designator | Pines | Género | MountType | EdgeSide | Offset | Size (w, h) | matedId | Notes |
|------|--------------|-------|------------|-------|--------|-----------|----------|--------|--------------|---------|-------|
| C001 | Molex 2P     | T300  | J1         | 2     | male   | fixed     | right    | 100    | 180, 115     | M001    | –     |
| C002 | Molex 2P     | T200  | J2         | 2     | female | flying    | left     | null   | 180, 115     | M001    | –     |
| C003 | GX12         | T200  | J3         | 2     | female | fixed     | right    | 740    | 180, 115     | M002    | –     |
| C004 | GX12         | T100  | J4         | 2     | male   | flying    | left     | null   | 180, 115     | M002    | –     |
| C005 | GX12 4P      | T100  | J5         | 4     | male   | flying    | right    | null   | 180, 115     | M003    | –     |
| C006 | GX12 4P      | T201  | J6         | 4     | female | fixed     | left     | 728    | 180, 115     | M003    | –     |
| C007 | Molex 5P     | T201  | J7         | 5     | male   | flying    | left     | null   | 180, 115     | M004    | –     |
| C008 | Molex 5P     | T301  | J8         | 5     | female | fixed     | right    | 71     | 180, 115     | M004    | –     |
| C009 | Molex 2P     | T301  | J9         | 2     | female | fixed     | left     | 251    | 180, 115     | null    | [{"date":"2026-07-12","user":"Leo","text":"Reserva para faro auxiliar"}] |

**Memoria de diseño – 4.3**  
Los conectores C002, C004, C005 y C007 son volantes: son los extremos de los cables W001, W002, W002 y W003 respectivamente. No almacenan `offset`. Los conectores C001, C003, C006, C008 y C009 son fijos: están soldados o atornillados a sus placas o paneles. Las orientaciones cumplen la regla de enfrentamiento 10.10.

#### 4.4 Posicionamiento de conectores

4.4.1. Conector fijo: completamente dentro del contenedor, con la cara de pines exactamente sobre el borde indicado por `edgeSide`.  
4.4.2. Conector volante: su posición se calcula para quedar enfrentado a su pareja fija (ver 4.1.4). Puede quedar fuera de los límites de su contenedor padre; esto genera una advertencia en el futuro sistema de logs (12.9) pero no se bloquea.  
4.4.3. Ejemplo: `edgeSide: "right"` en un fijo → borde derecho del conector toca el borde derecho del contenedor.

#### 4.5 Movimiento de conectores

4.5.1. **Conectores fijos**: pueden **deslizarse a lo largo del borde** (cambiando su `offset`) o **cambiarse a otro borde** del mismo contenedor si el cursor supera 30 px de distancia perpendicular al borde actual.  
4.5.2. **Conectores volantes**: no pueden arrastrarse directamente. Su posición se actualiza automáticamente al mover el conector fijo al que están enchufados.  
4.5.3. **Propagación rígida del movimiento:**  
 4.5.3.1. Los conectores unidos mediante un **M** se mueven solidariamente. Al mover un conector fijo (directamente o moviendo su contenedor), el volante asociado se reposiciona automáticamente para mantener el acople, ignorando su propio contenedor a efectos de posición.  
 4.5.3.2. Los conectores unidos solo por wires no se arrastran entre sí; el cable se redibuja.

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

#### 6.2 Validaciones

6.2.1. Los dos conectores deben tener géneros opuestos.  
6.2.2. Si un conector tiene `matedId`, ese ID debe coincidir con el M que lo incluye. Recíprocamente, si un conector aparece en el `from` o `to` de un M, debe tener ese `matedId`. El sistema rechazará cualquier configuración que no cumpla esta correspondencia bidireccional.  
6.2.3. El sistema verificará la coherencia al cargar: cada M debe ser referenciado por sus dos conectores.  
6.2.4. **Composición del M:** un M siempre conecta un conector fijo (`mountType: "fixed"`) con uno volante (`mountType: "flying"`). No se permiten M entre dos fijos ni entre dos volantes.  
6.2.5. **Mapeo de pines:**  
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

#### 6.4 Visualización

6.4.1. Sin línea; conectores enfrentados borde con borde.  
6.4.2. La unión rígida se manifiesta en el movimiento solidario.  
6.4.3. **Bloque rígido visual**: dado que un acople M representa una unión física real e inseparable, en la vista gráfica ambos conectores deben aparecer estrictamente enfrentados y sin separación. El usuario no puede separarlos arrastrándolos en modo edición; el volante sigue siempre al fijo. Si al cargar o editar datos la posición matemática de ambos conectores deja un espacio entre ellos, el sistema emitirá la advertencia correspondiente (ver 10.10).

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
│   │   └── C001 (J1, male, fixed, right)
│   ├── C002 (J2, female, flying, left)
│   └── C003 (J3, female, fixed, right)
├── T201 (Caja 2)
│   ├── T301 (PCB 2)
│   │   ├── C008 (J8, female, fixed, right)
│   │   └── C009 (J9, female, fixed, left)
│   ├── C006 (J6, female, fixed, left)
│   └── C007 (J7, male, flying, left)
├── C004 (J4, male, flying, left)
└── C005 (J5, male, flying, right)
```

#### 9.2 Ubicación efectiva de los wires

9.2.1. W001: dentro de T200.  
9.2.2. W002: en T100.  
9.2.3. W003: en T201.

---

### 10. REGLAS DE CONSISTENCIA Y VALIDACIONES

10.1. **Género en M:** géneros opuestos obligatorios.  
10.2. **Integridad de matedId:** si un conector tiene `matedId`, ese M debe existir, contener al conector, y ambos conectores del M deben tener el mismo `matedId`. Recíprocamente, si un conector aparece en el `from` o `to` de un M, debe tener ese `matedId`; el sistema rechazará cualquier configuración que no cumpla esta correspondencia bidireccional.  
10.3. **Obligatoriedad de edgeSide y mountType:** todo conector debe tener `edgeSide` y `mountType` definidos.  
10.4. **Composición de M:** un M conecta un fijo con un volante.  
10.5. **Pines válidos:** los pines referenciados deben existir en los conectores.  
10.6. **Compatibilidad jerárquica:** dos conectores solo pueden conectarse (mediante un cable W o un acople M) si comparten un ancestro contenedor común en el árbol de contención. En caso contrario, el sistema rechazará la conexión.  
10.7. **Longitud de wire:** `length` es informativo, sin restricción.  
10.8. **Continuidad de nets:** el grafo del net debe ser conexo; se detectan conflictos de señales.  
10.9. **Mapeo de pines (opcional):** si `pinMapping` es `"direct"` o `"reversed"`, los pines `from` y `to` deben cumplir el orden declarado. Si los conectores tienen distinto número de pines, se emitirá una advertencia informativa sobre los pines no asignados.  
10.10. **Enfrentamiento de pines en M:** los dos conectores que comparten un `matedId` deben tener orientaciones opuestas. Adicionalmente, el sistema verificará que las posiciones estén alineadas en la coordenada perpendicular al borde. Para acoples left/right, la diferencia entre sus coordenadas Y globales no debe superar los 2 píxeles. Para acoples top/bottom, la diferencia entre sus coordenadas X globales no debe superar los 2 píxeles. Si se supera este umbral, se emitirá una advertencia de desalineación.

---

### 11. VISUALIZACIÓN GENERAL

11.1. **Cables fijos:** líneas azules curvas, siempre visibles.  
11.2. **Acoples M:** sin línea; conectores enfrentados y bloqueados como un único conjunto rígido.  
11.3. **Conectores:** dentro del contenedor (fijos) o siguiendo a su pareja (volantes); pines sobre el borde.  
11.4. **Panel de configuración y modo edición:**  
 11.4.1. Existe un botón de configuración (ícono de engranaje) en la barra superior.  
 11.4.2. Al pulsarlo se despliega un panel que contiene, entre otras opciones, el control de **modo edición** y la **opción de rendimiento** para el recálculo de posiciones (continuo o al finalizar el movimiento).  
 11.4.3. El modo edición se activa/desactiva con un interruptor en ese panel, o mediante el atajo **Ctrl+Shift+E**.  
 11.4.4. Por defecto, el sistema arranca en modo solo lectura.  
 11.4.5. En modo edición se habilitan las interacciones de arrastre y redimensionamiento.  
11.5. **Redimensionamiento:** handle exclusivamente en la esquina inferior derecha. Los conectores fijos mantienen su offset respecto a la esquina superior izquierda. Los volantes no se ven afectados.  
11.6. **Resaltado de nets:** al seleccionar un net, sus wires cambian de color.  
11.7. **Filtros de vista:** la ocultación de conectores o subsistemas se realiza mediante filtros dinámicos en la interfaz. El estado de los filtros se guarda automáticamente en `localStorage` del navegador vinculado al identificador del proyecto, restaurándose al recargar la página. Existe un control para restablecer todos los filtros.  
11.8. **Panel de propiedades:** al hacer clic en cualquier elemento del lienzo (conector, contenedor, cable, M, N) se despliega un panel lateral con todos sus atributos, permitiendo su consulta en modo solo lectura y su edición en modo edición. Este panel está sincronizado con la vista de tablas: cualquier cambio realizado en él se refleja en las tablas y viceversa.

---

### 12. NOTAS PARA FUTURAS VERSIONES

12.1. Configuración de modo de arranque (lectura/edición) en preferencias.  
12.2. Enrutamiento automático de wires con evasión de obstáculos.  
12.3. Size visual de conectores adaptable a pines o designator.  
12.4. Panel de validación global.  
12.5. Soportar pares trenzados (campo `pair` en nets).  
12.6. Puntos de chasis como nodos de tierra implícitos.  
12.7. Integración con sistemas Kanban para seguimiento de tareas basadas en notas.  
12.8. En caso de requerirse, reintroducción de estados en M (planificado, desconectado, obsoleto) con un modelo más robusto.  
12.9. **Sistema de registro de eventos (logs):** para visualizar incidencias como conectores no enfrentados en un M, conectores volantes fuera de su contenedor padre, advertencias de validación y acciones del usuario, facilitando el diagnóstico y la auditoría.  
12.10. **Umbral de desalineación configurable:** el umbral de 2 píxeles para la validación de alineación de offsets (10.10) podrá ajustarse en el panel de configuración para adaptarse a distintas resoluciones de lienzo o niveles de zoom.

---

### 13. HISTORIAL DE VERSIONES

**Versión 1.0** – Documentación inicial con sistema de IDs, jerarquía de contenedores, conectores con expectedPair, cálculo dinámico de lockedWith, nets como etiquetas.

**Versión 1.1** – Mejoras estructurales: tabla única de entidades, sección "Contenedores".

**Versión 1.2** – Modo edición con toggle (Ctrl+Shift+E), eliminación de estados explícitos en conectores, adición de campos `notes` y `hidden`.

**Versión 1.3** – Refactorización mayor: sustitución de `expectedPair` por `matedId`, estructura histórica de notas, panel de configuración, corrección de numeración duplicada.

**Versión 1.4** – Adición de subsección 1.3 "Filosofía de Uso y Alcance".

**Versión 1.5** – Corrección de errores de coherencia y adición de `pinMapping` en M.

**Versión 1.6** – Simplificación del modelo: eliminación del campo `hidden` en conectores y de los estados en M.

**Versión 1.7** – Refactorización del sistema de posicionamiento: coordenadas relativas para todos los componentes excepto aquellos con `parent_id: null`, eliminación del conector libre (`edgeSide` obligatorio), reemplazo de `x`/`y` por `offset` en conectores, nuevas validaciones y persistencia de filtros.

**Versión 1.8** – Mejoras de contexto y refinamiento de datos: separación de `position` y `size`, nueva validación de enfrentamiento de pines en M y regla de bloque rígido visual, nueva subsección 1.4 "Flujo de trabajo del técnico", panel de propiedades, filtros rápidos.

**Versión 1.8.1** – Correcciones menores: eliminada la referencia a migración automática, unificado el término "size", corregido el ejemplo de conectores para cumplir la regla de enfrentamiento, eliminado el punto obsoleto sobre componentes no anclados.

**Versión 1.8.2** – Limpieza terminológica: eliminada toda referencia a "conectores anclados", eliminada la definición de "Libre (free)", simplificado el diagrama jerárquico.

**Versión 1.9** – Resolución del conflicto cinemático y simplificación del redimensionamiento: introducción de `mountType` (`"fixed"` / `"flying"`) para distinguir conectores atornillados de extremos de cable; conectores volantes sin `offset` propio; redimensionamiento limitado a la esquina inferior derecha; corrección de la validación de `matedId` (bidireccional) y de la alineación de offsets; nota aclaratoria sobre coordenadas absolutas en 4.1.3.

**Versión 1.9.1** – Correcciones de consistencia y claridad:  
- Restaurada la regla 10.6 (Compatibilidad jerárquica), que se había omitido accidentalmente en V1.9.  
- Mejorada la redacción de 3.3.1.3 y 4.5.3.1 para eliminar cualquier ambigüedad sobre el comportamiento cinemático de los conectores volantes.  
- Añadida aclaración en 4.1.1.b de que el `parent_id` de un volante es declarativo/organizativo.  
- Añadida nota 12.10 sobre la futura configurabilidad del umbral de desalineación.  
- Todos los errores detectados en la auditoría de V1.8.2 quedan definitivamente resueltos.
