# VERSIÓN 2.3 – DOCUMENTACIÓN DE ARQUITECTURA DEL VISUALIZADOR DE ARNÉS

---

## 1. RESUMEN Y PROPÓSITO DEL SISTEMA

1.1. Este visualizador modela el arnés de una moto eléctrica como un grafo de **componentes físicos** (contenedores, conectores) unidos por **relaciones** (cables fijos, acoples enchufables) y organizados mediante **señales lógicas** (nets).

1.2. Permite visualizar los elementos, moverlos en modo edición respetando la jerarquía de contención, observar cables flexibles y validar automáticamente la coherencia de géneros, pines, señales y enfrentamiento de conectores.

### 1.3 Filosofía de Uso y Alcance

1.3.1. Esta aplicación no es un simulador eléctrico ni una herramienta de diseño de circuitos. Es una **plataforma de documentación visual interactiva** para técnicos y mecánicos, pensada para representar el cableado "como si estuviera extendido en el piso del taller".

1.3.2. La herramienta permite tanto **consultar** un arnés existente como **construir y modificar** su representación desde cero. Todas las entidades (contenedores, conectores, cables, acoples) pueden crearse, editarse y eliminarse directamente desde la interfaz gráfica o mediante la carga de archivos de datos.

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

1.3.6. A partir de la V2.0, el sistema incorpora **catálogos reutilizables**, un **esquema de validación dinámica** y un sistema de **gestión de usuarios y responsables**.

1.3.7. A partir de la V2.1, se simplifica la arquitectura: las **nets se mueven a catálogos**, se **elimina `metadata.uiSettings`**, se **especifica SVG nativo** como tecnología de visualización.

1.3.8. A partir de la V2.2, se aplican mejoras de usabilidad y simplificación: **toggle de modo edición en el header**, **barra lateral con memoria de estado**, **validación unificada**, **eliminación de la regla de coherencia modelo/instancia**, **eliminación de `sectionRef` en conectores**, **gauge como campo recomendado**, **filtro intermedio**, y **documentación explícita del flujo de GitHub Pages**.

1.3.9. A partir de la V2.3, se incorporan: **principio de inferencia desde catálogos** (autocompletado automático), **catálogo `colorPalette`** para renderizado SVG, **schema específico por entidad**, **simplificación del catálogo `people`**, **eliminación de `shield` como tipo de señal**, **límite de profundidad jerárquica (4 niveles) con detección de ciclos**, **validación 100% custom** (sin librerías externas), y **arquitectura single-file** (`index.html` + `db.json`).

### 1.4 Flujo de trabajo del técnico

1.4.1. La interfaz ofrece dos vistas complementarias y sincronizadas: el **lienzo gráfico 2D (SVG)** y las **tablas de datos**. El usuario puede alternar entre ambas o trabajar con las dos simultáneamente. Cualquier cambio realizado en una vista se propaga instantáneamente a la otra.

1.4.2. **Panel de propiedades desplegable**: al hacer clic en cualquier elemento del lienzo (cable, conector, contenedor, M) se abre un panel lateral que muestra todos sus atributos. Este panel permite la edición directa de los campos. En modo solo lectura, los campos aparecen bloqueados; en modo edición, se habilitan para su modificación.

1.4.3. **Filtros intermedios (V2.2)**: el técnico puede aislar visualmente elementos mediante una combinación de filtros simples que se aplican simultáneamente (AND):
- Campo de texto para búsqueda por ID, nombre o designator.
- Dropdown para filtrar por tipo de entidad (Todos / Contenedores / Conectores / Wires / Mates).
- Dropdown para filtrar por net (Todas / lista de nets del catálogo).
- Dropdown para filtrar por sección (Todas / lista de secciones del catálogo).
- Botón "Limpiar filtros" para restablecer.

Los elementos que no coinciden se atenúan en la vista visual (opacidad reducida) y se ocultan en la vista tabla. El estado de los filtros se conserva en `localStorage`.

1.4.4. **Persistencia del contexto**: los filtros aplicados, la posición del lienzo, el nivel de zoom y el tema se almacenan en `localStorage` del navegador para que el técnico recupere exactamente la vista en la que estaba trabajando.

### 1.5 Principio de Inferencia desde Catálogos (V2.3)

1.5.1. La aplicación sigue el principio de **"inferir antes que preguntar"**: si un campo de una entidad puede deducirse a partir de un catálogo referenciado, la aplicación lo **autocompleta automáticamente** al crear o editar la entidad. El usuario siempre puede sobreescribir el valor autocompletado.

1.5.2. Este principio aplica en los siguientes casos:

| Campo a autocompletar | Se infiere de | Condición |
|---|---|---|
| `wires.gaugeUnit` | `wireTypeRef.unit` | Si `wireTypeRef` existe y tiene `unit` |
| `wires.color` | `nets.colorCode` de la net asignada | Si el wire tiene `net` y la net tiene `colorCode` en `colorPalette`. Si no, default `"black"`. |
| `connectors.pins` | `modelRef.pins` | Si `modelRef` existe y tiene `pins` |
| `connectors.gender` | `modelRef.gender` | Si `modelRef` existe y tiene `gender` |
| `mates.pinMapping` | Valor por defecto `"direct"` | Al crear un mate nuevo **desde la UI**. Mates existentes en el JSON mantienen su valor original. |
| `wires.gaugeUnit` | `"mm2"` | Si no hay `wireTypeRef` (default global) |

1.5.3. El autocompletado se dispara:
- Al **crear** una nueva entidad (se rellenan los campos inferibles).
- Al **cambiar una referencia** (ej: cambiar `wireTypeRef` actualiza `gaugeUnit` si no fue sobreescrito manualmente).
- El usuario puede **sobreescribir** cualquier valor autocompletado. Una vez sobreescrito, el campo se considera "manual" y no se vuelve a autocompletar automáticamente a menos que el usuario lo restablezca explícitamente.

1.5.4. Este principio **no aplica** a campos que son fuente de verdad de la instancia. Por ejemplo, `connectors.pins` se autocompleta desde `modelRef.pins`, pero si el usuario lo cambia, el valor del usuario prevalece y no se revierte al cambiar el modelo.

1.5.5. El autocompletado es una **ayuda de velocidad**, no una restricción. El usuario tiene siempre la última palabra sobre el valor de cualquier campo.

**Memoria de diseño – Sección 1**
Se mantiene el propósito original del MVP: ofrecer una vista interactiva del arnés con restricciones físicas realistas. La separación entre componentes, relaciones y señales sigue el patrón modelo-vista. La V2.3 introduce el principio de inferencia desde catálogos como filosofía transversal de la aplicación, priorizando la agilidad del usuario sin sacrificar control.

---

## 2. SISTEMA DE IDENTIFICADORES (ÍNDICE DE ENTIDADES)

2.1. Cada entidad del sistema recibe un **ID único** compuesto por una letra de tipo más un número de tres dígitos. La siguiente tabla actúa como índice general de todas las entidades modeladas.

**Tabla 1 – Índice de entidades y prefijos de identificación**

| Prefijo | Entidad                  | Rango numérico       | Ejemplo |
|---------|--------------------------|----------------------|---------|
| T       | Contenedor (system, enclosure, pcb) | 100 – 399 (subdividido) | T100, T200, T300 |
| C       | Conector (connector)     | 001 – 999            | C001    |
| W       | Cable fijo (wired)       | 001 – 999            | W001    |
| M       | Acople enchufable (mated)| 001 – 999            | M001    |

> **Nota (V2.1):** La entidad `N` (net) ha sido eliminada del índice de entidades. Las nets ya no son entidades con ID propio (`N001`, `N002`), sino entradas en el catálogo `metadata.catalogs.nets` con nombres descriptivos (`GND`, `+12V`, `CAN_H`). Los wires y mates las referencian por su nombre/ID en el catálogo.

2.2. La letra **`T`** agrupa todos los contenedores (system, enclosure, pcb) bajo un mismo prefijo. La subdivisión del rango numérico indica el nivel jerárquico:
 2.2.1. `1xx` → sistema raíz (system).
 2.2.2. `2xx` → caja (enclosure).
 2.2.3. `3xx` → placa de circuito (pcb).

2.3. Esta organización permite hasta 99 contenedores de cada tipo (ej. T101, T201, T301…) y mantiene la coherencia conceptual: todos los contenedores comparten prefijo, pero su posición jerárquica es deducible del número.

2.4. El campo `type` (p. ej. `"system"`, `"enclosure"`, `"pcb"`) sigue presente en los datos para definir explícitamente el subtipo; el ID solo actúa como identificador único.

2.5. **Convención de rangos (V2.2):** Los rangos numéricos (T: 100-399, C/W/M: 001-999) son una **guía organizativa**, no una restricción técnica. El patrón de validación `^[TCWM]\d{3}$` acepta cualquier número de tres dígitos. No se validan rangos específicos en el schema ni en el código.

### 2.6 Principio de posicionamiento

2.6.1. **Regla general**:
- Todo componente con `parent_id: null` tiene posición absoluta respecto al lienzo (atributos `x`, `y`).
- Todo componente con `parent_id` no nulo tiene posición relativa a su padre (atributos `offsetX`, `offsetY` para contenedores; `offset` para conectores fijos; los conectores volantes no almacenan posición propia).

2.6.2. En la práctica actual, solo `T100` (system raíz) cumple `parent_id: null`. Los conectores como C004 y C005 tienen `parent_id: "T100"` (es decir, tienen padre; no son `null`). La regla está enunciada de forma genérica para admitir futuros componentes raíz sin modificar la lógica de posicionamiento.

2.6.3. Las dimensiones (`width`, `height`) se almacenan siempre en un objeto `size` separado de la posición, tanto para contenedores como para conectores. Esto diferencia conceptualmente la ubicación del size y facilita el mantenimiento del código.

### 2.7 Estructura del Archivo de Datos (`db.json`)

2.7.1. El proyecto se almacena en un único archivo JSON llamado **`db.json`** con dos secciones raíz: `metadata` y `data`.

2.7.2. **`metadata`**: Contiene toda la información de contexto del proyecto: descripción, autoría, trazabilidad de guardado, reglas de validación y catálogos reutilizables (incluyendo las nets y la paleta de colores).

2.7.3. **`data`**: Contiene las instancias físicas reales del arnés: contenedores, conectores, cables y acoples. Esta sección es la fuente de verdad del modelo.

2.7.4. **Estructura del árbol**:
```
db.json
├── metadata
│   ├── projectInfo        (objeto: nombre, descripción, fecha, modelo)
│   ├── lastSave           (string ISO 8601: timestamp del último guardado)
│   ├── savedBy            (string: ref a catalogs.people, quién guardó)
│   ├── version            (number: contador de guardados, incrementa en 1)
│   ├── schema             (objeto: reglas de validación por entidad)
│   └── catalogs           (objeto: catálogos reutilizables)
│       ├── people
│       ├── sections
│       ├── connectorModels
│       ├── wireTypes
│       ├── nets
│       └── colorPalette   ← Nuevo en V2.3
└── data
    ├── containers         (array de contenedores T)
    ├── connectors         (array de conectores C)
    ├── wires              (array de cables W)
    └── mates              (array de acoples M)
```

2.7.5. El archivo **NO debe contener comentarios** (`//` ni `/* */`). JSON estándar no los soporta. Si se necesitan anotaciones, se usa el campo `notes` de cada entidad o un campo `"_comment"` a nivel raíz.

2.7.6. **Campo `metadata.version` (V2.2):** Es un **contador de guardados** (number), no una versión del esquema. Se incrementa en 1 cada vez que el usuario exporta/guarda el archivo. Comienza en 1. Si el usuario necesita registrar la versión del proyecto (ej: "MotoStudents v2.0"), puede hacerlo en `metadata.projectInfo.description` o en un campo custom dentro de `projectInfo`. No existe lógica de migración basada en este campo.

> **Nota histórica (V2.3):** En V2.2, el campo `revision` fue renombrado a `version`. La funcionalidad es idéntica: contador de guardados que incrementa en 1.

2.7.7. **No existe `metadata.uiSettings`** (eliminado en V2.1). Todas las preferencias de interfaz (tema, zoom, posición del lienzo, filtros, vista activa, estado de expansión de contenedores) se almacenan en `localStorage` del navegador. Ver sección 11.9.

2.7.8. **Arquitectura single-file (V2.3):** La aplicación se entrega como un único archivo **`index.html`** autocontenido (HTML + CSS embebido en `<style>` + JavaScript embebido en `<script>`) más el archivo **`db.json`** externo con los datos del proyecto. No se requieren bundlers, frameworks ni servidores backend. El despliegue en GitHub Pages consiste en subir ambos archivos al repositorio.

### 2.8 Catálogos (`metadata.catalogs`)

2.8.1. Los catálogos son colecciones de **modelos genéricos y datos reutilizables**. No representan instancias físicas del arnés, sino plantillas y referencias que enriquecen a las entidades en `data`.

2.8.2. **Principio fundamental**: los catálogos son **opcionales y complementarios**. Si un archivo JSON no contiene `catalogs` o alguna de sus sub-secciones, la aplicación funciona correctamente. Los campos `modelRef`, `wireTypeRef` y `owner` en `data` son todos opcionales.

2.8.3. **Regla de precedencia**: los campos de la instancia en `data` son siempre la fuente de verdad. El catálogo solo aporta información documental y valores para autocompletado (ver 1.5). No se valida coherencia entre catálogo e instancia.

2.8.4. **Sub-catálogos**:

**a) `people`** — Catálogo de personas del proyecto (simplificado en V2.3).
```
"<person_id>": {
  "name": "string (obligatorio)"
}
```
Se referencia desde `metadata.savedBy` y desde el campo `owner` de cualquier entidad en `data`. El catálogo solo almacena el nombre. No se incluyen alias, roles ni notas: si una persona cumple una función o es responsable, se indica mediante el campo `owner` en la entidad correspondiente. Todos los campos `owner` son opcionales.

**Ejemplo:**
```json
"people": {
  "leo": { "name": "Leo" },
  "martin": { "name": "Martin" },
  "nico": { "name": "Nico" }
}
```

**b) `sections`** — Secciones funcionales del sistema (simplificado en V2.3).
```
"<section_id>": {
  "name": "string (obligatorio)"
}
```
Se referencia desde el campo `sectionRef` de **contenedores** (los conectores heredan la sección de su contenedor padre, ver 2.10.4). Permite agrupar entidades por subsistema (CCU, sensores, potencia, etc.). El catálogo solo almacena el nombre de la sección. La pertenencia de personas a secciones se deduce de los campos `owner` y `sectionRef` en las entidades de `data`.

**Ejemplo:**
```json
"sections": {
  "ccu": { "name": "Central Control Unit" },
  "sensors": { "name": "Safety Sensors" },
  "power": { "name": "Power Distribution" }
}
```

**c) `connectorModels`** — Modelos genéricos de conectores.
```
"<model_id>": {
  "manufacturer": "string (opcional)",
  "partNumber": "string (opcional)",
  "pins": "number (opcional)",
  "gender": "'male' | 'female' (opcional)",
  "type": "string (opcional)",
  "datasheetUrl": "string / null (opcional)",
  "specs": "object (opcional, libre)"
}
```
Se referencia desde `connectors.modelRef`. Aporta información documental (datasheet, specs técnicas) que se muestra en el panel de detalles en modo solo lectura. Los campos `pins` y `gender` se usan para autocompletado (ver 1.5).

**Nota importante**: un mismo modelo físico con géneros opuestos (ej: Molex 2P macho y Molex 2P hembra) debe representarse como **dos entradas separadas** en el catálogo, ya que generalmente tienen part numbers distintos.

**d) `wireTypes`** — Tipos genéricos de cable.
```
"<type_id>": {
  "unit": "'AWG' | 'mm2' (obligatorio)",
  "shielded": "boolean (opcional)",
  "insulationType": "string (opcional)",
  "temperatureRating": "string (opcional)",
  "standards": ["array de strings (opcional)"]
}
```
Se referencia desde `wires.wireTypeRef`. Aporta información documental sobre el tipo de cable. **No contiene el valor de `gauge`**; ese valor vive exclusivamente en la instancia (`data.wires.gauge`). El campo `unit` se usa para autocompletar `gaugeUnit` en la instancia (ver 1.5).

**e) `nets`** — Señales eléctricas estándar (V2.1).
```
"<net_id>": {
  "name": "string (obligatorio)",
  "signalType": "'power' | 'ground' | 'data' | 'communication' | 'analog' (obligatorio)",
  "voltage": "string (opcional)",
  "standard": "string (opcional)",
  "colorCode": "string (opcional, ref a colorPalette)",
  "description": "string (opcional)"
}
```
Las nets son señales estándar reutilizables (GND, +12V, +5V, CAN_H, CAN_L, etc.). Se referencian desde `wires.net` y `mates.net` por su ID en el catálogo. **No existe `data.nets`** (eliminado en V2.1).

> **Nota (V2.3):** El valor `"shield"` ha sido eliminado de `signalType`. El blindaje/malla es una propiedad física del cable, no una señal eléctrica. Se documenta mediante `wireTypes.shielded`. La lista de `signalType` es extensible: nuevos tipos pueden añadirse al catálogo en versiones futuras o mediante edición directa del JSON.

**f) `colorPalette`** — Paleta de colores para renderizado SVG (V2.3).
```
"<color_name>": "string (hexadecimal)"
```
Mapea nombres de color descriptivos a valores hexadecimales para el renderizado SVG. Los campos `color` en wires y `colorCode` en nets referencian claves de este catálogo.

**Ejemplo:**
```json
"colorPalette": {
  "black": "#000000",
  "red": "#dc2626",
  "blue": "#2563eb",
  "green": "#16a34a",
  "yellow": "#eab308",
  "orange": "#ea580c",
  "white": "#f5f5f5",
  "gray": "#6b7280",
  "brown": "#92400e",
  "violet": "#7c3aed",
  "Green/White": "#22c55e",
  "White/Green": "#e5e7eb"
}
```

**Reglas de uso:**
- Si un campo `color` o `colorCode` referencia una clave que **no existe** en `colorPalette`, se usa el color gris por defecto (`#6b7280`) y se emite una **Advertencia** en el panel de logs.
- El usuario puede personalizar colores editando el catálogo directamente en el JSON.
- La paleta es extensible: se pueden añadir nuevas entradas libremente.

### 2.9 Esquema de Validación (`metadata.schema`)

2.9.1. El objeto `schema` define reglas de validación **dinámicas** que la aplicación interpreta automáticamente al cargar o modificar el JSON. Esto permite cambiar reglas de validación sin modificar el código JavaScript.

2.9.2. **Implementación (V2.3):** La validación se implementa en **JavaScript 100% custom**, sin librerías externas. La función `validateProject()` maneja todas las reglas (required, recommended, pattern, unique, ref) y las reglas de negocio (secciones 10.1 a 10.12 y 10.14 a 10.16). No se utiliza AJV ni ninguna otra librería de validación.

2.9.3. **Validación unificada (V2.2):** Toda la validación se ejecuta desde una **única función `validateProject()`** que:
1. Itera por cada array de `data` (containers, connectors, wires, mates).
2. Aplica el sub-schema correspondiente a cada entidad (required, recommended).
3. Aplica las reglas globales (unique, pattern, ref).
4. Aplica las reglas de negocio (10.1 a 10.12 y 10.14 a 10.16).
5. Retorna un array de objetos `{ level: "error"|"warning"|"info", message: string, entityId: string }`.

Esta función se llama:
- **Al cargar** el archivo: validación completa.
- **Al guardar/exportar**: validación completa.
- **En tiempo real**: solo se valida la entidad que se está editando en el panel `details` (validación incremental, para mejorar rendimiento).

2.9.4. **Schema específico por entidad (V2.3):** El schema se estructura por tipo de entidad, ya que cada una tiene campos obligatorios y recomendados distintos:

```json
"schema": {
  "containers": {
    "required": ["id", "type", "position", "size"],
    "recommended": ["name", "owner", "sectionRef"]
  },
  "connectors": {
    "required": ["id", "type", "gender", "mountType", "edgeSide", "parent_id", "pins"],
    "recommended": ["name", "owner", "modelRef"]
  },
  "wires": {
    "required": ["id", "type", "from", "to", "net"],
    "recommended": ["name", "owner", "gauge", "color", "wireTypeRef"]
  },
  "mates": {
    "required": ["id", "type", "from", "to", "net"],
    "recommended": ["name", "owner", "pinMapping"]
  },
  "rules": {
    "id": { "unique": true, "pattern": "^[TCWM]\\d{3}$" },
    "containers.parent_id": { "ref": "containers" },
    "connectors.parent_id": { "ref": "containers" },
    "connectors.modelRef": { "ref": "connectorModels" },
    "connectors.owner": { "ref": "people" },
    "wires.from.connector": { "ref": "connectors" },
    "wires.to.connector": { "ref": "connectors" },
    "wires.net": { "ref": "nets" },
    "wires.wireTypeRef": { "ref": "wireTypes" },
    "wires.owner": { "ref": "people" },
    "mates.from.connector": { "ref": "connectors" },
    "mates.to.connector": { "ref": "connectors" },
    "mates.net": { "ref": "nets" },
    "mates.owner": { "ref": "people" }
  }
}
```

2.9.5. **Campos del schema por entidad**:
- `required`: Array de strings con nombres de campos que deben existir en toda entidad **de ese tipo**. Si falta un campo required, se genera un **Error**.
- `recommended`: Array de strings con nombres de campos cuya ausencia genera una **Advertencia** (no error).

2.9.6. **Tipos de regla** (en `rules`):
- `"unique": true` → El valor debe ser único en todo el array de esa entidad. **Validación custom**.
- `"pattern": "regex"` → El valor debe cumplir la expresión regular. **Validación custom**.
- `"ref": "catalog_name"` → El valor debe coincidir con una clave existente en el catálogo especificado (dentro de `metadata.catalogs`) o en el array correspondiente de `data`. **Validación custom**.

2.9.7. **Convenciones de paths en `rules`**:
- La regla `"id"` sin prefijo de entidad aplica a **todas** las entidades de todos los arrays en `data`.
- `"containers.parent_id"` aplica solo al campo `parent_id` dentro de los objetos del array `data.containers`.
- `"ref": "containers"` valida contra los IDs en `data.containers`.
- `"ref": "connectorModels"` valida contra las claves en `metadata.catalogs.connectorModels`.
- `"ref": "nets"` valida contra las claves en `metadata.catalogs.nets`.
- `"ref": "people"` valida contra las claves en `metadata.catalogs.people`.

2.9.8. **Criterio de separación schema vs código**:
- **Schema (validación custom):** Validaciones de **integridad referencial** y **formato** (unique, pattern, ref, required, recommended).
- **Código (reglas 10.1–10.12 y 10.14–10.16):** Validaciones de **lógica de negocio** (géneros opuestos, cinemática, composición de M, compatibilidad jerárquica, conectores volantes sin pareja, ciclos jerárquicos).

### 2.10 Gestión de Usuarios y Responsables

2.10.1. **`metadata.savedBy`**: Registra el `person_id` de la persona que realizó el último guardado del archivo. Se actualiza automáticamente al exportar.

2.10.2. **Campo `owner`**: Disponible en todas las entidades de `data` (containers, connectors, wires, mates). Indica la persona responsable de esa entidad específica. Es opcional y refiere a `catalogs.people`.

2.10.3. **Campo `sectionRef`**: Disponible **solo en contenedores** (V2.2). Vincula el contenedor a una sección funcional definida en `catalogs.sections`. Permite filtrar y agrupar entidades por subsistema.

2.10.4. **Herencia de sección en conectores (V2.2):** Los conectores **no tienen campo `sectionRef` propio**. Siempre heredan la sección de su contenedor padre. Si el contenedor padre tiene `sectionRef: "ccu"`, todos sus conectores descendientes pertenecen a la sección "ccu". Si el contenedor padre no tiene `sectionRef`, se hereda del abuelo, y así sucesivamente hasta la raíz. Si ningún ancestro tiene `sectionRef`, el conector no tiene sección asignada. La cadena de herencia está limitada a **4 niveles** de profundidad (ver regla 10.16).

2.10.5. La interfaz muestra el nombre del usuario actual (desde `catalogs.people`) y permite cambiarlo desde el panel de configuración. Al guardar, el `savedBy` se actualiza automáticamente con el usuario activo.

**Memoria de diseño – Sección 2**
La tabla única de entidades facilita la consulta rápida. Se conserva el campo `type` porque la letra `T` por sí sola no distingue entre sistema, caja o placa. La decisión de usar centenas como niveles jerárquicos permite un crecimiento holgado. La separación de `position` y `size` responde a la necesidad de tratar independientemente la ubicación y las dimensiones. En V2.3 se simplifica el catálogo `people` (solo nombre), se elimina `owners` de `sections`, se añade `colorPalette`, y se adopta schema específico por entidad con validación 100% custom.

---

## 3. CONTENEDORES (SYSTEM, ENCLOSURE, PCB)

Son los **contenedores** que definen la estructura física del arnés. No tienen género ni pines; su función es agrupar conectores y limitar su movimiento.

### 3.1 Atributos comunes

**Tabla 2 – Atributos de contenedores**

| Campo       | Tipo         | Descripción |
|-------------|--------------|-------------|
| id          | string       | T100, T200, T300… |
| type        | string       | `"system"`, `"enclosure"`, `"pcb"` |
| name        | string       | Nombre descriptivo (ej. "Caja 1") |
| parent_id   | string / null| ID del contenedor padre (`null` solo para el sistema raíz) |
| designator  | string       | Etiqueta técnica (ej. "BOX1", "PCB1") |
| position    | object       | **Si `parent_id` es `null`**: `{ x, y }` en píxeles absolutos. **Si `parent_id` no es `null`**: `{ offsetX, offsetY }` en píxeles relativos al padre. |
| size        | object       | `{ width, height }` en píxeles |
| owner       | string / null| Ref a `catalogs.people`. Responsable del contenedor. Opcional. |
| sectionRef  | string / null| Ref a `catalogs.sections`. Sección funcional a la que pertenece. Opcional. |
| notes       | array        | Histórico de notas (ver 3.1.1) |

#### 3.1.1 Estructura de notas

Cada entrada del array `notes` es un objeto con el formato:
- `date` (string, ISO 8601): fecha de creación de la nota.
- `user` (string): identificador del usuario que la escribió.
- `text` (string, máximo 500 caracteres): contenido del comentario.

El campo `notes` está disponible en **todas** las entidades (contenedores, conectores, wires, mated). Se concibe como un historial acumulativo de anotaciones técnicas.

### 3.2 Catálogo de contenedores

**Tabla 3 – Lista de contenedores del ejemplo**

| ID   | Type       | Nombre         | Padre | Designator | Position (offsetX, offsetY) | Size (w, h) | Owner  | SectionRef | Notas |
|------|------------|----------------|-------|------------|-----------------------------|-------------|--------|------------|-------|
| T100 | system     | Moto Eléctrica | null  | EMOTO-Z1   | x:50, y:50 *(absoluto)*     | 2600, 1200  | leo    | –          | –     |
| T200 | enclosure  | Caja 1         | T100  | BOX1       | 90, 110                     | 950, 1000   | martin | ccu        | –     |
| T201 | enclosure  | Caja 2         | T100  | BOX2       | 1593, 122                   | 950, 1000   | nico   | sensors    | –     |
| T300 | pcb        | PCB 1          | T200  | PCB1       | 80, 100                     | 368, 837    | martin | ccu        | –     |
| T301 | pcb        | PCB 2          | T201  | PCB2       | 427, 117                    | 472, 817    | nico   | sensors    | –     |

**Memoria de diseño – Sección 3**
El término "contenedores" agrupa system, enclosure y pcb bajo un mismo concepto. La jerarquía de `parent_id` nulo solo en el sistema raíz evita componentes huérfanos. Los valores de posición han sido convertidos a relativos (offsetX/offsetY) excepto para T100, que mantiene coordenadas absolutas por ser la raíz. La separación de `position` y `size` clarifica la estructura de datos.

### 3.3 Movimiento y redimensionamiento

#### 3.3.1 Movimiento de contenedores

3.3.1.1. Al arrastrar un contenedor, todos sus descendientes se desplazan el mismo vector **(dx, dy) en tiempo real**, sin retardo.
3.3.1.2. El contenedor arrastrado **no modifica su size** ni ninguna otra propiedad; solo cambian sus valores de posición (`offsetX`/`offsetY`, o `x`/`y` si es T100).
3.3.1.3. Los conectores fijos (`mountType: "fixed"`) se reposicionan automáticamente sobre el borde del padre en el mismo fotograma. Los conectores volantes (`mountType: "flying"`) no están anclados cinemáticamente a su contenedor: su posición se recalcula en cada frame a partir de la posición del conector fijo al que están enchufados. Por tanto, un volante se mueve únicamente si el fijo se mueve (por arrastre directo o por movimiento del contenedor que contiene al fijo). Si el contenedor del volante se mueve pero el fijo no, el volante permanece en su sitio, lo que puede provocar que quede fuera de los límites de su contenedor (ver 10.13, panel de logs).

**Memoria de diseño – 3.3.1**
El movimiento solidario en tiempo real evita parpadeos. Al ser todas las posiciones relativas, mover un contenedor no requiere actualizar las coordenadas de sus descendientes; el renderizado recalcula las posiciones absolutas a partir de los offsets en cada frame.

#### 3.3.2 Redimensionamiento

3.3.2.1. Los contenedores pueden redimensionarse arrastrando **exclusivamente la esquina inferior derecha**.
3.3.2.2. Durante el redimensionamiento:
 a. Los conectores fijos (`mountType: "fixed"`) conservan su distancia a la esquina superior izquierda del contenedor, que es la esquina fija durante el estiramiento.
 b. Los conectores volantes (`mountType: "flying"`) no se ven afectados por el redimensionamiento de su contenedor padre; su posición se recalcula a partir de su pareja fija en el M.

**Memoria de diseño – 3.3.2**
La restricción del redimensionamiento a una única esquina (inferior derecha) simplifica la implementación y evita los problemas de redimensionamiento inverso detectados en versiones anteriores.

---

## 4. CONECTORES

Los conectores son los puntos de conexión eléctrica. Cada uno tiene un género, un tipo de montaje y pertenece a un contenedor físico.

### 4.1 Atributos

**Tabla 4 – Atributos de conectores**

| Campo      | Tipo          | Descripción |
|------------|---------------|-------------|
| id         | string        | C001, C002… |
| type       | string        | Siempre `"connector"` |
| name       | string        | Nombre descriptivo |
| parent_id  | string        | ID del contenedor donde está ubicado físicamente |
| designator | string        | Etiqueta técnica (ej. "J1") |
| pins       | number        | Cantidad de pines |
| gender     | string        | `"male"` o `"female"` (obligatorio) |
| mountType  | string        | **Obligatorio.** `"fixed"` o `"flying"` |
| edgeSide   | string        | **Obligatorio.** `"left"`, `"right"`, `"top"`, `"bottom"` |
| offset     | number / null | Distancia desde el extremo de referencia del borde. **Solo operativo para `mountType: "fixed"`.** Para `"flying"`, este campo es **ignorado** y puede ser `null` o cualquier valor; no tiene efecto en el posicionamiento ni genera advertencias. |
| size       | object        | `{ width, height }` en píxeles |
| matedId    | string / null | ID del acople M al que pertenece, o `null` si está libre |
| modelRef   | string / null | Ref a `catalogs.connectorModels`. **Opcional, informativo.** |
| owner      | string / null | Ref a `catalogs.people`. Responsable del conector. Opcional. |
| notes      | array         | Histórico de notas (ver 3.1.1) |

> **Nota (V2.2):** Los conectores **no tienen campo `sectionRef`**. Siempre heredan la sección de su contenedor padre (ver 2.10.4).

> **⚠️ Nota crítica de precedencia (V2.0):** Los campos `name`, `pins` y `gender` de la instancia en `data` son la **fuente de verdad absoluta**. `modelRef` solo enriquece la vista del panel de detalles con datasheet, fabricante y especificaciones técnicas del catálogo, y proporciona valores para autocompletado (ver 1.5). No se valida coherencia entre catálogo e instancia.

4.1.1. `mountType` define la cinemática del conector:
 a. **Fijo (`"fixed"`)**: el conector está atornillado o soldado a su contenedor padre. Su posición se calcula a partir de `edgeSide` y `offset`. Al mover el contenedor, el conector se mueve con él.
 b. **Volante (`"flying"`)**: el conector es el extremo de un cable. No está fijado mecánicamente a su contenedor; su `parent_id` es declarativo/organizativo (indica en qué caja está físicamente el cable, permite calcular el ancestro común para los wires y valida la compatibilidad jerárquica), pero no influye en su posición. Esta se calcula a partir del conector fijo al que está enchufado (su pareja en el M). Al mover el contenedor padre, el conector volante **no se desplaza** con él; sigue a su pareja fija. Si como resultado queda fuera de los límites de su contenedor, el sistema registra una advertencia en el panel de logs (ver 10.13, panel de logs), pero no bloquea la acción.

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
 c. La coordenada perpendicular se alinea por **pin 1**: el pin 1 de ambos conectores queda a la misma altura (o posición horizontal, según la orientación). Los pines adicionales del conector más grande se extienden en la dirección correspondiente.

> **Nota (V2.3):** La alineación por pin 1 asume que ambos conectores tienen la misma orientación de numeración (pin 1 en el extremo superior/izquierdo). Si un conector tiene una numeración no estándar, usar `pinMapping: null` en el acople y especificar los pines manualmente en cada wire/mate.

4.1.5. `matedId` vincula el conector con el acople M que lo une a su pareja. Si el conector participa en un M, aquí se almacena el ID de dicho M.

4.1.6. **Conectores volantes sin pareja (V2.1):** Si un conector tiene `mountType: "flying"` y `matedId: null` (o `matedId` apunta a un M inexistente), se aplica la regla 10.14: se genera un Error en logs, se omite en la visualización SVG, y se resalta en la tabla. El usuario puede editarlo, pero permanecerá marcado hasta que tenga una pareja válida.

4.1.7. **Conectores fijos sin pareja (V2.2):** Si un conector tiene `mountType: "fixed"` y `matedId: null`, es una situación **válida** (conector soldado pero no conectado, reserva para expansión futura). Se genera un mensaje **Info** en el panel de logs: "Conector fijo [ID] no tiene pareja (puede ser intencional)". El conector se dibuja normalmente en el SVG y no se resalta en la tabla.

4.1.8. **Distribución de pines dentro del conector (V2.3):**
- Los pines se distribuyen **verticalmente** dentro del conector (para `edgeSide: "left"` o `"right"`) u **horizontalmente** (para `"top"` o `"bottom"`).
- Espaciado entre pines: `conectorHeight / (pins + 1)` para distribución vertical, o `conectorWidth / (pins + 1)` para distribución horizontal.
- El pin 1 está en la posición más cercana al borde superior (o izquierdo).
- Cada pin se dibuja como un `<circle>` de radio 5px en el borde del conector.
- El punto de conexión del wire es el centro del círculo del pin.

**Memoria de diseño – 4.1**
La introducción de `mountType` en V1.9 resuelve el conflicto cinemático. Los conectores volantes no almacenan `offset` operativo; su posición es siempre derivada de su pareja fija. Los campos `modelRef` y `owner` se añaden en V2.0 como opcionales aditivos. En V2.1 se especifica la alineación por pin 1 y el comportamiento de conectores volantes sin pareja. En V2.2 se elimina `sectionRef` de conectores y se define el comportamiento de conectores fijos sin pareja. En V2.3 se aclara que `offset` es ignorado en volantes, se especifica la distribución de pines, y se añade nota sobre orientación de numeración.

### 4.2 Género y validación

4.2.1. Todo conector tiene género obligatorio (`male` o `female`).
4.2.2. Los acoples **M** requieren géneros opuestos. No importa cuál es `from` o `to`.
4.2.3. El sistema verificará la coherencia de géneros al cargar o editar.

### 4.3 Catálogo completo de conectores

**Tabla 5 – Conectores del ejemplo**

| ID   | Nombre     | Padre | Designator | Pines | Género | MountType | EdgeSide | Offset | Size (w,h) | matedId | ModelRef           | Owner  | Notes |
|------|------------|-------|------------|-------|--------|-----------|----------|--------|------------|---------|--------------------| ------ |-------|
| C001 | Molex 2P   | T300  | J1         | 2     | male   | fixed     | right    | 100    | 180, 115   | M001    | MOLEX_2P_MALE      | martin | –     |
| C002 | Molex 2P   | T200  | J2         | 2     | female | flying    | left     | null   | 180, 115   | M001    | MOLEX_2P_FEMALE    | martin | –     |
| C003 | GX12       | T200  | J3         | 2     | female | fixed     | right    | 740    | 180, 115   | M002    | GX12_2P_FEMALE     | martin | –     |
| C004 | GX12       | T100  | J4         | 2     | male   | flying    | left     | null   | 180, 115   | M002    | GX12_2P_MALE       | leo    | –     |
| C005 | GX12 4P    | T100  | J5         | 4     | male   | flying    | right    | null   | 180, 115   | M003    | GX12_4P_MALE       | leo    | –     |
| C006 | GX12 4P    | T201  | J6         | 4     | female | fixed     | left     | 728    | 180, 115   | M003    | GX12_4P_FEMALE     | nico   | –     |
| C007 | Molex 5P   | T201  | J7         | 5     | male   | flying    | left     | null   | 180, 115   | M004    | MOLEX_5P_MALE      | nico   | –     |
| C008 | Molex 5P   | T301  | J8         | 5     | female | fixed     | right    | 71     | 180, 115   | M004    | MOLEX_5P_FEMALE    | nico   | –     |
| C009 | Molex 2P   | T301  | J9         | 2     | female | fixed     | left     | 251    | 180, 115   | null    | MOLEX_2P_FEMALE    | nico   | Ver nota |

**Nota para C009**: `[{"date":"2026-07-12","user":"leo","text":"Reserva para faro auxiliar"}]`

**Memoria de diseño – 4.3**
Los conectores C002, C004, C005 y C007 son volantes: son los extremos de los cables. No almacenan `offset` operativo. Los conectores C001, C003, C006, C008 y C009 son fijos. Se añaden `modelRef` y `owner` en V2.0. Cada género del mismo modelo físico tiene su propia entrada en `connectorModels`. C009 es un conector fijo sin pareja (reserva), situación válida que genera Info en logs.

### 4.4 Posicionamiento de conectores

4.4.1. Conector fijo: completamente dentro del contenedor, con la cara de pines exactamente sobre el borde indicado por `edgeSide`.
4.4.2. Conector volante: su posición se calcula para quedar enfrentado a su pareja fija (ver 4.1.4). Puede quedar fuera de los límites de su contenedor padre; esto genera una advertencia en el panel de logs (10.13, panel de logs) pero no se bloquea.
4.4.3. Ejemplo: `edgeSide: "right"` en un fijo → borde derecho del conector toca el borde derecho del contenedor.

### 4.5 Movimiento de conectores

4.5.1. **Conectores fijos**: pueden **deslizarse a lo largo del borde** (cambiando su `offset`) o **cambiarse a otro borde** del mismo contenedor si el cursor supera 30 px de distancia perpendicular al borde actual.
4.5.2. **Conectores volantes**: no pueden arrastrarse directamente. Su posición se actualiza automáticamente al mover el conector fijo al que están enchufados.
4.5.3. **Propagación rígida del movimiento:**
 4.5.3.1. Los conectores unidos mediante un **M** se mueven solidariamente. Al mover un conector fijo (directamente o moviendo su contenedor), el volante asociado se reposiciona automáticamente para mantener el acople, ignorando su propio contenedor a efectos de posición.
 4.5.3.2. Los conectores unidos solo por wires no se arrastran entre sí; el cable se redibuja.

---

## 5. CABLES FIJOS (WIRED) — ENTIDAD TIPO W

Representan un conductor físico real (cable soldado o crimpado). No transmiten movimiento mecánico.

### 5.1 Atributos

**Tabla 6 – Atributos de un wire**

| Campo       | Tipo   | Descripción |
|-------------|--------|-------------|
| id          | string | W001, W002… |
| type        | string | Siempre `"wired"` |
| from        | object | `{ connector: "C002", pin: 1 }` |
| to          | object | `{ connector: "C003", pin: 1 }` |
| net         | string | ID del net que transporta (ref a `catalogs.nets`). Obligatorio. |
| length      | number | Longitud real en mm. **Campo directo y altamente editable.** |
| gauge       | number / null | Valor numérico del calibre. **Recomendado, no obligatorio.** Si falta, se genera Advertencia en logs. |
| gaugeUnit   | string / null | `"AWG"` o `"mm2"`. **Opcional.** Si `wireTypeRef` existe, se infiere del catálogo (ver 1.5). Si no existe, defaultea a `"mm2"`. |
| color       | string | Color de línea (ref a `catalogs.colorPalette`). Defecto: `"black"`. Si el wire tiene `net` y la net tiene `colorCode`, el color se autocompleta desde la net (ver 1.5). |
| thickness   | number | Grosor en px (opcional) |
| wireTypeRef | string / null | Ref a `catalogs.wireTypes`. Opcional, informativo. |
| owner       | string / null | Ref a `catalogs.people`. Opcional. |
| notes       | array  | Histórico de notas |

> **Nota (V2.3):** El campo `gauge` es **recomendado pero no obligatorio**. La sección del cable no se representa visualmente en el dibujo SVG (el grosor visual lo controla `thickness`), por lo que la ausencia de `gauge` no impide el renderizado. Si `gauge` falta, se genera una **Advertencia** en el panel de logs para recordar al usuario documentarlo. En el ejemplo, todos los wires tienen `gauge` rellenado como buena práctica de documentación.

> **Nota (V2.3):** El campo `gaugeUnit` es **opcional**. Si el wire tiene `wireTypeRef` y el tipo de cable tiene `unit`, `gaugeUnit` se **autocompleta** con ese valor (principio de inferencia, ver 1.5). Si no hay `wireTypeRef`, `gaugeUnit` defaultea a `"mm2"`. El usuario puede sobreescribir el valor autocompletado.

> **Nota (V2.1):** El campo `gauge` es un **number** que vive exclusivamente en la instancia. El catálogo `wireTypes` **no contiene** el valor de gauge; solo aporta información documental (aislación, estándares, temperatura) y la unidad (`unit`).

5.1.1. La ubicación física se deduce automáticamente del ancestro común más bajo de los dos conectores extremos.

### 5.2 Tabla de wires del ejemplo

**Tabla 7 – Wires del arnés**

| ID   | From (C, pin) | To (C, pin) | Net | Longitud | Gauge | GaugeUnit | WireTypeRef        | Owner  |
|------|---------------|-------------|-----|----------|-------|-----------|--------------------|--------|
| W001 | C002, 1       | C003, 1     | GND | 150 mm   | 22    | AWG       | AWG22_SHIELDED     | martin |
| W002 | C004, 1       | C005, 1     | GND | 400 mm   | 22    | AWG       | AWG22_UNSHIELDED   | leo    |
| W003 | C006, 1       | C007, 2     | GND | 180 mm   | 22    | AWG       | AWG22_SHIELDED     | nico   |

### 5.3 Visualización y comportamiento

5.3.1. Línea curva (Bézier cúbica en SVG) del color especificado en `color` (resuelto mediante `catalogs.colorPalette`), siempre visible.
5.3.2. Siempre por encima de cajas y conectores.
5.3.3. Se redibuja como curva adaptable al mover un extremo.
5.3.4. La línea se acorta/estira libremente; `length` es solo informativo.
5.3.5. **Wires con extremos inválidos (V2.2):** Si alguno de los dos conectores extremos no tiene posición válida (ej: conector volante sin pareja, regla 10.14), el wire **no se dibuja** en el SVG. El wire sigue apareciendo en la tabla de datos, pero no en la vista visual. No se genera un error adicional por el wire; el error ya se genera por el conector inválido.

5.3.6. **Fórmula de curvas Bézier (V2.3):**
- Punto de inicio (`x1, y1`): centro del pin de origen (en el borde del conector).
- Punto de fin (`x2, y2`): centro del pin de destino.
- Puntos de control: desplazados según el `edgeSide` del conector correspondiente.
  - Si el conector origen tiene `edgeSide: "right"`, el primer punto de control se desplaza en +X.
  - Si el conector origen tiene `edgeSide: "left"`, el primer punto de control se desplaza en -X.
  - Análogo para el conector destino.
- Magnitud del desplazamiento: `cx = Math.max(50, Math.abs(x2 - x1) * 0.4)`.
- Path SVG: `M x1,y1 C x1+cx,y1 x2-cx,y2 x2,y2` (para conectores enfrentados left/right).
- Para orientaciones top/bottom, el desplazamiento es en Y en lugar de X.

---

## 6. ACOPLES ENCHUFABLES (MATED) — ENTIDAD TIPO M

Define una unión macho‑hembra entre dos conectores. Unión rígida: los conectores se mueven juntos, sin línea visual adicional.

### 6.1 Atributos

**Tabla 8 – Atributos de un mated**

| Campo      | Tipo   | Descripción |
|------------|--------|-------------|
| id         | string | M001, M002… |
| type       | string | Siempre `"mated"` |
| from       | object | `{ connector: "C001", pin: 1 }` |
| to         | object | `{ connector: "C002", pin: 1 }` |
| net        | string | ID del net que transporta (ref a `catalogs.nets`) |
| pinMapping | string / null | `"direct"`, `"reversed"`, o `null`. Si no se especifica, se trata como `null`. |
| owner      | string / null | Ref a `catalogs.people`. Opcional. |
| notes      | array  | Histórico de notas |

### 6.2 Validaciones

6.2.1. Los dos conectores deben tener géneros opuestos.
6.2.2. Si un conector tiene `matedId`, ese ID debe coincidir con el M que lo incluye. Recíprocamente, si un conector aparece en el `from` o `to` de un M, debe tener ese `matedId`. El sistema rechazará cualquier configuración que no cumpla esta correspondencia bidireccional.
6.2.3. El sistema verificará la coherencia al cargar: cada M debe ser referenciado por sus dos conectores.
6.2.4. **Composición del M:** un M siempre conecta un conector fijo (`mountType: "fixed"`) con uno volante (`mountType: "flying"`). No se permiten M entre dos fijos ni entre dos volantes.
6.2.5. **Mapeo de pines:**
 a. Si `pinMapping` es `"direct"`, el acople respeta el orden natural: pin 1 con pin 1, pin 2 con pin 2, etc. El sistema validará que los pines declarados en `from` y `to` cumplan esta correspondencia.
 b. Si `pinMapping` es `"reversed"`, el orden se invierte: pin 1 con pin N, pin 2 con N‑1, etc., siendo N el número de pines del conector más pequeño.
 c. Si `pinMapping` es `null` o no está presente, no se aplica validación automática de mapeo.
 d. Si los conectores tienen distinto número de pines y `pinMapping` es `"direct"` o `"reversed"`, el sistema emitirá una advertencia informativa indicando cuántos pines del conector más grande quedan sin asignar en este acople.

6.2.6. **Autocompletado de `pinMapping` (V2.3):** Al crear un nuevo mate desde la interfaz, el campo `pinMapping` se autocompleta con `"direct"` como valor sugerido (principio de inferencia, ver 1.5). El usuario puede cambiarlo a `"reversed"` o `null` según necesite. La validación de mapeo solo se activa si el valor es explícitamente `"direct"` o `"reversed"`. Este autocompletado aplica solo a mates **nuevos creados desde la interfaz**. Los mates existentes en el JSON mantienen su valor original (ej: M004 con `pinMapping: null`).

6.2.7. **Limitación del modelo de acople (V2.3):** Un M conecta **exactamente dos conectores** (un `from` y un `to`). Casos atípicos como un conector de N pines conectado a dos conectores de N/2 pines deben representarse como **dos M separados**, cada uno con `pinMapping: null` (sin validación automática, porque el mapeo es parcial). Este caso es extremadamente raro en arneses de motos eléctricas y no se añade complejidad al modelo para soportarlo nativamente.

### 6.3 Tabla de mated del ejemplo

**Tabla 9 – Acoples mated**

| ID   | From (C, pin) | To (C, pin) | Net | pinMapping | Owner  |
|------|---------------|-------------|-----|------------|--------|
| M001 | C001, 1       | C002, 1     | GND | direct     | martin |
| M002 | C004, 1       | C003, 1     | GND | direct     | leo    |
| M003 | C005, 1       | C006, 1     | GND | direct     | nico   |
| M004 | C007, 2       | C008, 2     | GND | null       | nico   |

> **Nota (V2.3):** En el ejemplo base, todos los wires y mates comparten la misma net (`GND`), reflejando el caso del MVP original donde todas las conexiones formaban parte de un mismo circuito. Si se desea mostrar diversidad de señales (power, data, communication), se debe crear un ejemplo adicional separado con componentes ficticios, sin modificar el ejemplo base.

### 6.4 Visualización

6.4.1. Sin línea; conectores enfrentados borde con borde.
6.4.2. La unión rígida se manifiesta en el movimiento solidario.
6.4.3. **Bloque rígido visual**: dado que un acople M representa una unión física real e inseparable, en la vista gráfica ambos conectores deben aparecer estrictamente enfrentados y sin separación. El usuario no puede separarlos arrastrándolos en modo edición; el volante sigue siempre al fijo. Si la posición matemática deja un espacio, se emite advertencia (ver 10.10).

---

## 7. BLOQUEO DINÁMICO (`lockedWith`)

### 7.1 Generación en runtime

7.1.1. `lockedWith` no se almacena; se calcula a partir de todos los M existentes (todos son activos por definición).
7.1.2. Cada M hace que ambos conectores se incluyan mutuamente en `lockedWith`.
7.1.3. Los wires no contribuyen.

### 7.2 Cadenas rígidas

7.2.1. En el ejemplo:
 a. C001‑C002 (M001)
 b. C003‑C004 (M002)
 c. C005‑C006 (M003)
 d. C007‑C008 (M004)
7.2.2. Los wires unen estas cadenas pero no las solidarizan mecánicamente.

---

## 8. SEÑALES LÓGICAS (NETS) — CATÁLOGO

### 8.1 Naturaleza de las nets (V2.1)

8.1.1. A partir de la V2.1, las nets **no son entidades de datos** sino **entradas en el catálogo** `metadata.catalogs.nets`. Esto refleja su naturaleza real: las señales eléctricas (GND, +12V, +5V, CAN_H, CAN_L, etc.) son estándares reutilizables entre proyectos, no datos específicos de un arnés.

8.1.2. **No existe `data.nets`.** Los wires y mates referencian las nets por su ID en el catálogo (ej: `"net": "GND"`, `"net": "+12V"`).

8.1.3. **No existe la entidad tipo `N`** en el índice de entidades (Tabla 1). Las nets no tienen ID con prefijo `N001`; usan nombres descriptivos como claves del catálogo.

### 8.2 Atributos de una net en el catálogo

**Tabla 10 – Atributos de una net en `catalogs.nets`**

| Campo       | Tipo          | Descripción |
|-------------|---------------|-------------|
| name        | string        | Nombre descriptivo (obligatorio) |
| signalType  | string        | Categoría: `"power"`, `"ground"`, `"data"`, `"communication"`, `"analog"` (obligatorio). Lista extensible. |
| voltage     | string / null | Tensión nominal (opcional) |
| standard    | string / null | Estándar aplicable (ej: "ISO 11898-2" para CAN) |
| colorCode   | string / null | Color sugerido para visualización (ref a `catalogs.colorPalette`) |
| description | string / null | Descripción adicional |

> **Nota (V2.3):** El valor `"shield"` ha sido eliminado de `signalType`. El blindaje es una propiedad física del cable (`wireTypes.shielded`), no una señal. La lista de `signalType` es extensible: nuevos tipos pueden añadirse al catálogo en versiones futuras o mediante edición directa del JSON.

### 8.3 Catálogo de nets del ejemplo

**Tabla 11 – Nets definidas en `catalogs.nets`**

| ID (clave) | Nombre          | SignalType | Voltage | Standard    | ColorCode |
|------------|-----------------|------------|---------|-------------|-----------|
| GND        | System Ground   | ground     | 0V      | –           | black     |

> **Nota (V2.3):** El ejemplo base utiliza una única net (`GND`) para todas las conexiones, manteniendo coherencia con el MVP original. El catálogo puede contener más nets (ej: `+12V`, `+5V`, `CAN_H`, `CAN_L`) para uso en proyectos reales o ejemplos adicionales, pero el ejemplo base solo usa `GND`.

### 8.4 Uso de nets

8.4.1. Verificar continuidad eléctrica: el sistema examina wires y mateds con el mismo net y construye un grafo de nodos (conector, pin).
8.4.2. Validar coherencia de señales: todos los wires y mates que comparten un net deben ser eléctricamente compatibles.
8.4.3. Resaltado visual al seleccionar un net: se resaltan todos los wires y mates que transportan esa señal.
8.4.4. Autocompletado: al crear un wire o mate, el usuario selecciona el net desde el catálogo (dropdown con todas las nets disponibles). El campo `color` del wire se autocompleta con el `colorCode` de la net seleccionada (ver 1.5).
8.4.5. Futuro: puntos de chasis implícitos.

### 8.5 Construcción automática del grafo

8.5.1. El sistema examina wires y mateds con el mismo net y construye un grafo de nodos (conector, pin).
8.5.2. La ruta del ejemplo para `GND`: C001.pin1 – C002.pin1 – C003.pin1 – C004.pin1 – C005.pin1 – C006.pin1 – C007.pin2 – C008.pin2.

> **Nota (V2.3):** En el ejemplo, todos los tramos comparten la net `GND`, reflejando un circuito único. En un arnés real, cada tramo llevaría su señal específica (power, data, ground, etc.). La diversidad de señales se muestra en ejemplos adicionales, no en el ejemplo base.

---

## 9. ÁRBOL DE CONTENCIÓN (JERARQUÍA FÍSICA)

### 9.1 Diagrama jerárquico

```
T100 (Moto) [owner: leo]
├── T200 (Caja 1) [owner: martin, section: ccu]
│   ├── T300 (PCB 1) [owner: martin, section: ccu]
│   │   └── C001 (J1, male, fixed, right) → hereda section: ccu
│   ├── C002 (J2, female, flying, left) → hereda section: ccu
│   └── C003 (J3, female, fixed, right) → hereda section: ccu
├── T201 (Caja 2) [owner: nico, section: sensors]
│   ├── T301 (PCB 2) [owner: nico, section: sensors]
│   │   ├── C008 (J8, female, fixed, right) → hereda section: sensors
│   │   └── C009 (J9, female, fixed, left) → hereda section: sensors
│   ├── C006 (J6, female, fixed, left) → hereda section: sensors
│   └── C007 (J7, male, flying, left) → hereda section: sensors
├── C004 (J4, male, flying, left) → hereda section: null (T100 no tiene sectionRef)
└── C005 (J5, male, flying, right) → hereda section: null
```

### 9.2 Ubicación efectiva de los wires

9.2.1. W001: dentro de T200.
9.2.2. W002: en T100.
9.2.3. W003: en T201.

---

## 10. REGLAS DE CONSISTENCIA Y VALIDACIONES

10.1. **Género en M:** géneros opuestos obligatorios.
10.2. **Integridad de matedId:** si un conector tiene `matedId`, ese M debe existir, contener al conector, y ambos conectores del M deben tener el mismo `matedId`. Recíprocamente, si un conector aparece en el `from` o `to` de un M, debe tener ese `matedId`; el sistema rechazará cualquier configuración que no cumpla esta correspondencia bidireccional.
10.3. **Obligatoriedad de edgeSide y mountType:** todo conector debe tener `edgeSide` y `mountType` definidos.
10.4. **Composición de M:** un M conecta un fijo con un volante.
10.5. **Pines válidos:** los pines referenciados deben existir en los conectores (el número de pin debe ser ≤ `pins` del conector).
10.6. **Compatibilidad jerárquica:** dos conectores solo pueden conectarse (mediante un cable W o un acople M) si comparten un ancestro contenedor común en el árbol de contención. En caso contrario, el sistema rechazará la conexión.
10.7. **Longitud de wire:** `length` es informativo, sin restricción.
10.8. **Continuidad de nets:** el grafo del net debe ser conexo; se detectan conflictos de señales.
10.9. **Mapeo de pines (opcional):** si `pinMapping` es `"direct"` o `"reversed"`, los pines `from` y `to` deben cumplir el orden declarado. Si los conectores tienen distinto número de pines, se emitirá una advertencia informativa sobre los pines no asignados.
10.10. **Enfrentamiento de pines en M:** los dos conectores que comparten un `matedId` deben tener orientaciones opuestas. Adicionalmente, el sistema verificará que las posiciones estén alineadas en la coordenada perpendicular al borde. Para acoples left/right, la diferencia entre sus coordenadas Y globales no debe superar los 2 píxeles. Para acoples top/bottom, la diferencia entre sus coordenadas X globales no debe superar los 2 píxeles. Si se supera este umbral, se emitirá una advertencia de desalineación.

### 10.11 Validación de Referencias (`ref`)

10.11.1. El sistema debe verificar que todos los campos con regla `ref` en `metadata.schema` apunten a IDs existentes en su destino correspondiente:
- `parent_id` → ID existente en `data.containers`.
- `modelRef` → clave existente en `metadata.catalogs.connectorModels`.
- `wireTypeRef` → clave existente en `metadata.catalogs.wireTypes`.
- `sectionRef` → clave existente en `metadata.catalogs.sections` (solo en contenedores).
- `owner` y `savedBy` → clave existente en `metadata.catalogs.people`.
- `net` → clave existente en `metadata.catalogs.nets`.
- `from.connector` y `to.connector` → ID existente en `data.connectors`.
- `color` y `colorCode` → clave existente en `metadata.catalogs.colorPalette`.

10.11.2. Si una referencia apunta a un ID inexistente, se genera un **Error** en el panel de logs.
10.11.3. Si el campo es `null` o no existe y es opcional, no se genera error.
10.11.4. Si un campo `color` o `colorCode` referencia una clave inexistente en `colorPalette`, se usa el color gris por defecto (`#6b7280`) y se emite una **Advertencia**.

### 10.12 Validación de Unicidad (`unique`)

10.12.1. Los campos marcados con `"unique": true` en `metadata.schema` deben tener valores distintos en todas las entidades del mismo array.
10.12.2. Si se detecta un valor duplicado, se genera un **Error** en el panel de logs indicando cuáles entidades comparten el mismo valor.
10.12.3. La regla `"id"` sin prefijo aplica a todas las entidades de todos los arrays en `data`.

### 10.13 Panel de Logs (Activo)

10.13.1. El panel de logs es un contenedor ubicado en la base de la pantalla, colapsable hacia abajo mediante transición CSS (`translate-y-full`).

10.13.2. **Tres niveles de severidad:**
- **Error** (rojo `#ef4444`): Problemas que impiden la coherencia del modelo (referencias rotas, unicidad violada, géneros iguales en un M, conectores volantes sin pareja, ciclos jerárquicos, etc.).
- **Advertencia** (amarillo `#ffcc00`): Inconsistencias que no rompen el modelo pero merecen atención (conector volante fuera de contenedor padre, desalineación de M, campos recomendados ausentes como `gauge` o `owner`, color inexistente en colorPalette).
- **Info** (azul `#3b82f6`): Mensajes informativos (carga exitosa, exportación completada, pines no asignados en un M con distinto número de pines, conector fijo sin pareja).

10.13.3. El panel integra automáticamente los resultados de la función unificada `validateProject()` (ver 2.9.3), que incluye:
- La validación del `schema` custom (required, recommended, pattern, unique, ref).
- Las reglas de negocio (secciones 10.1 a 10.12 y 10.14 a 10.16). La sección 10.13 describe el panel de logs en sí, no una regla de validación.
- Eventos de usuario relevantes (importación, exportación, borrado).

10.13.4. El panel incluye:
- Filtro por tipo de mensaje (mostrar/ocultar errores, advertencias, info).
- Botón para limpiar todos los logs.
- Contador de mensajes por nivel.
- Timestamp en cada entrada.

### 10.14 Conectores Volantes sin Pareja (V2.1)

10.14.1. Si un conector tiene `mountType: "flying"` y `matedId: null` (o `matedId` apunta a un M inexistente), se genera un **Error** en el panel de logs con el mensaje: "Conector volante [ID] no tiene pareja válida (matedId nulo o inexistente)".

10.14.2. **En la vista visual (SVG):** el conector **no se dibuja**. Se omite completamente del renderizado SVG. Los wires conectados a ese conector tampoco se dibujan (ver 5.3.5).

10.14.3. **En la vista tabla:** la fila del conector se **resalta con un borde rojo** (`border: 2px solid #ef4444`) y un ícono de error. El resaltado persiste hasta que el conector tenga un `matedId` válido.

10.14.4. El usuario **puede editar** el conector desde el panel `details` o la tabla (cambiar `matedId`, `mountType`, etc.). Una vez que el conector tiene un `matedId` válido, el error desaparece del log, el resaltado se elimina, y el conector se dibuja normalmente en el SVG.

10.14.5. Esta regla no bloquea la carga del archivo ni impide otras operaciones. Es una advertencia visual persistente.

### 10.15 Offset en Conectores Volantes (V2.3)

10.15.1. Si un conector tiene `mountType: "flying"`, el campo `offset` es **ignorado** por el sistema. Puede ser `null`, `0`, o cualquier valor numérico; no tiene efecto en el posicionamiento ni genera advertencias ni errores.

10.15.2. Esta regla es de **tolerancia**: permite que archivos JSON con datos heredados o editados manualmente no generen ruido innecesario en los logs por un campo que no aplica a conectores volantes.

### 10.16 Límite de Profundidad Jerárquica y Detección de Ciclos (V2.3)

10.16.1. **Límite de profundidad:** La cadena de `parent_id` en contenedores tiene un límite máximo de **4 niveles** de profundidad (contando desde 0). Esto cubre el modelo actual (system → enclosure → pcb, profundidad 2) con dos niveles de margen para expansión (ej: system → enclosure → sub-enclosure → pcb, profundidad 3). Si se supera el límite, se emite un **Error**: "Jerarquía demasiado profunda en [ID] (máximo 4 niveles)".

10.16.2. **Detección de ciclos:** La cadena de `parent_id` no debe contener ciclos. Un ciclo ocurre cuando un contenedor es ancestro de sí mismo (ej: T200.parent_id = T300 y T300.parent_id = T200). El sistema detecta ciclos mediante un conjunto de IDs visitados durante el recorrido recursivo. Si se detecta un ciclo, se emite un **Error**: "Ciclo jerárquico detectado en [ID]" y se detiene el cálculo de posiciones y herencia para los contenedores involucrados.

10.16.3. **Herencia de `sectionRef`:** La cadena de herencia de sección (ver 2.10.4) respeta el mismo límite de 4 niveles. Si se supera, se emite Error y se detiene la herencia.

10.16.4. **Cálculo de posiciones absolutas:** El cálculo recursivo de posiciones absolutas (ver 4.1.3) también respeta el límite de 4 niveles y la detección de ciclos. Si se detecta un ciclo o se supera el límite, los contenedores afectados se posicionan en (0, 0) como fallback y se emite Error.

---

## 11. VISUALIZACIÓN GENERAL E INTERFAZ

### 11.1 Tecnología de Visualización: SVG Nativo (V2.1)

11.1.1. La vista visual se implementa con **SVG nativo** (no Canvas). Todos los elementos gráficos se dibujan como elementos SVG manipulados directamente desde JavaScript mediante `document.createElementNS`.

11.1.2. **Elementos SVG utilizados:**
- **Contenedores:** `<rect>` con bordes redondeados, relleno semitransparente y etiqueta `<text>`.
- **Conectores:** `<rect>` con pines representados como `<circle>` pequeños en el borde correspondiente.
- **Cables (wires):** `<path>` con curvas Bézier cúbicas (comando `C`). El color del trazo se resuelve mediante `catalogs.colorPalette`.
- **Acoples (M):** Sin línea. Los conectores se dibujan enfrentados borde con borde.
- **Pines:** `<circle>` de radio 5px, clickeables individualmente.
- **Etiquetas:** `<text>` con tipografía clara (mínimo 14px en etiquetas, 18-22px en títulos de contenedores).

11.1.3. **Ventajas de SVG sobre Canvas para esta aplicación:**
- Interacción directa con elementos (click, hover, drag) sin necesidad de hit-testing manual.
- Escalado perfecto con zoom (no se pixela).
- Estilos CSS aplicables directamente (colores, grosores, animaciones, filtros de glow).
- Depuración más sencilla (inspeccionar elementos en el DOM del navegador).

11.1.4. **Cables fijos:** curvas Bézier cúbicas en SVG, del color especificado en el campo `color` (resuelto vía `colorPalette`), siempre visibles. Se redibujan dinámicamente al mover conectores.

11.1.5. **Acoples M:** sin línea; conectores enfrentados y bloqueados como un único conjunto rígido.

11.1.6. **Conectores:** dentro del contenedor (fijos) o siguiendo a su pareja (volantes); pines sobre el borde. Los conectores volantes sin pareja válida (regla 10.14) **no se dibujan**. Los conectores fijos sin pareja (regla 4.1.7) **sí se dibujan** normalmente.

11.1.7. **Orden de renderizado (Z-index, V2.3):** De abajo hacia arriba:
1. Fondo del SVG (rectángulo negro).
2. Contenedores (por jerarquía: system → enclosure → pcb).
3. Conectores (fijos y volantes).
4. Acoples M (sin línea, solo conectores enfrentados).
5. Cables W (curvas Bézier).
6. Etiquetas de cables (net, longitud).
7. Pines (círculos pequeños, clickeables).
8. Handles de redimensionamiento (solo en modo edición).
9. Overlay de selección (borde amarillo flúor + glow).

11.1.8. **Zoom y Pan (V2.3):**
- Se utiliza un elemento `<g id="viewport">` que envuelve todo el contenido del SVG.
- **Zoom:** Rueda del ratón. Modifica `transform: scale(z)` del viewport, centrado en la posición del cursor. Rango: 0.2 a 3.0.
- **Pan:** Arrastre con **botón medio del mouse** (rueda). Modifica `transform: translate(px, py)` del viewport. Alternativa: `Shift` + botón izquierdo.
- **Seleccionar:** Click izquierdo simple sobre un elemento.
- **Mover:** Click izquierdo mantenido + arrastre sobre un elemento arrastrable (contenedor o conector fijo).
- **Deseleccionar:** Click izquierdo sobre fondo vacío, o tecla `Esc`.
- El estado de zoom y pan se guarda en `localStorage`.

### 11.2 Header Minimalista (V2.2)

11.2.1. El header ocupa el ancho completo superior de la pantalla, es estático y contiene exactamente cinco zonas:

- **Izquierda**: Nombre de la aplicación (**ArnesViz**) seguido del nombre del archivo abierto (desde `metadata.projectInfo.name`).
- **Centro**: Selector de vista con dos pestañas/botones: **Tabla** y **Visual**.
- **Derecha (de izquierda a derecha)**:
  - **Toggle de Modo Edición** (interruptor visible siempre, V2.2). Permite activar/desactivar el modo edición sin abrir el panel de configuración. Atajo: `Ctrl+Shift+E`.
  - **Botón Config** (ícono de engranaje ⚙). Abre la barra lateral en estado `config`.

11.2.2. El header tiene estética neumórfica oscura (ver 11.8) y altura fija mínima para no restar espacio al área de trabajo.

### 11.3 Barra Lateral Única y Reutilizable (V2.2)

11.3.1. Existe **un único panel lateral** ubicado en el borde derecho de la pantalla. Este panel tiene **dos estados mutuamente excluyentes**: `config` y `details`.

11.3.2. **Apertura y cierre:**
- Al pulsar el botón **Config** del header, el panel se abre (o cambia) al estado `config`.
- Al hacer clic en una entidad del lienzo o la tabla, el panel se abre (o cambia) al estado `details`.
- El panel incluye un **botón de cerrar (✕)** en su esquina superior derecha. Al pulsarlo, el panel se cierra.
- **NO existe un botón lateral flotante (◀) para reabrir el panel.** El panel se reabre únicamente al pulsar Config o al seleccionar una entidad.

11.3.3. **Memoria de estado (V2.2):**
- Al cerrar el panel, la app **recuerda el último estado** (última entidad seleccionada en `details`, o `config`).
- Al reabrir el panel (pulsando Config o seleccionando una entidad), se muestra el contenido correspondiente.
- El último estado se guarda en memoria (no en localStorage), por lo que se pierde al recargar la página.

11.3.4. **Transición entre estados:**
- Al cambiar de `config` a `details` (o viceversa), el contenido del panel se reemplaza con una transición suave (`0.25s ease`).
- La tecla `Esc` cierra el panel sin importar su estado actual.

11.3.5. El ancho del panel es fijo (aproximadamente 380px) y ocupa toda la altura disponible debajo del header.

### 11.4 Estado `details` (Panel de Propiedades)

11.4.1. Se activa al seleccionar una entidad (T, C, W, M) en la vista visual o en la tabla.
11.4.2. Muestra **todos los campos** de la entidad seleccionada organizados en secciones.
11.4.3. En **modo solo lectura**, todos los campos aparecen como texto no editable.
11.4.4. En **modo edición**, los campos se transforman en campos editables (inputs, selects, textareas). Los cambios se sincronizan bidireccionalmente con las tablas y el lienzo.
11.4.5. Si la entidad tiene `modelRef` o `wireTypeRef`, se muestra una sección adicional en **solo lectura** con la información del catálogo: fabricante, part number, datasheet (enlace), especificaciones.
11.4.6. Si la entidad tiene `owner`, se muestra un selector desplegable con las personas de `catalogs.people`.
11.4.7. El historial de `notes` se muestra en la parte inferior del panel, con opción de añadir nuevas notas (que se sellan automáticamente con fecha y usuario actual).
11.4.8. **Validación incremental (V2.2):** Mientras el usuario edita campos en el panel `details`, se valida **solo la entidad actual** en tiempo real. Los errores y advertencias se muestran inline (junto al campo) y se actualizan en el panel de logs. No se revalida todo el proyecto en cada cambio.
11.4.9. **Autocompletado (V2.3):** Al crear una nueva entidad o al cambiar una referencia (ej: `modelRef`, `wireTypeRef`, `net`), los campos inferibles se autocompletan automáticamente según el principio de inferencia (ver 1.5). El usuario puede sobreescribir cualquier valor autocompletado.
11.4.10. **Botones de acción (V2.3):**
- **Eliminar:** Solo en modo edición. Abre modal de confirmación. Si la entidad tiene referencias (ej: conector usado en wires), muestra advertencia con lista de entidades afectadas.
- **Duplicar:** Crea una copia de la entidad con nuevo ID auto-asignado.

### 11.5 Estado `config` (Panel de Configuración)

11.5.1. Se activa al pulsar el botón **Config** (⚙) del header.
11.5.2. Contiene las siguientes secciones, organizadas de arriba a abajo:

**a) Usuario actual:**
- Selector desplegable con las personas de `catalogs.people`.
- Al cambiar, se actualiza `metadata.savedBy`.

**b) Vista y Renderizado:**
- Selector de columnas visibles (para la vista tabla).
- Opciones de zoom (entrada numérica + botones +/−).
- Toggle para mostrar/ocultar nombres de conectores, designadores, longitud de cables, etc.
- Selector de tema (Oscuro / Claro). **Esta preferencia se guarda en `localStorage`, no en el JSON.**

**c) Gestión de Datos:**
- Botón **Importar JSON** (atajo `Ctrl+O`).
- Botón **Exportar JSON** (atajo `Ctrl+S`).
- Botón **Borrar todo**.

**d) Gestión de Catálogos (solo lectura en V2.2):**
- Lista de personas (`catalogs.people`): ver.
- Lista de secciones (`catalogs.sections`): ver.
- Lista de modelos de conectores (`catalogs.connectorModels`): ver.
- Lista de tipos de cable (`catalogs.wireTypes`): ver.
- Lista de nets (`catalogs.nets`): ver.
- Lista de colores (`catalogs.colorPalette`): ver.

> **Nota (V2.2):** La edición de catálogos desde la UI se pospone para una versión futura (ver sección 12.13). Por ahora, los catálogos se editan directamente en el archivo JSON. La interfaz solo permite visualizarlos.

11.5.3. **Flujo de Importar:**
1. Si `data` contiene entidades (arrays no vacíos) **y hay cambios sin guardar**, se abre un **modal de confirmación**:
   - Título: "Cambios sin guardar"
   - Mensaje: "Hay cambios sin guardar. ¿Deseas exportar el archivo actual antes de importar?"
   - Opciones: **[Exportar y continuar]** | **[Descartar y continuar]** | **[Cancelar]**
2. Si el usuario elige "Exportar y continuar", se ejecuta automáticamente una exportación del JSON actual antes de proceder.
3. Si el usuario elige "Cancelar", se aborta la importación.
4. Si `data` está vacío **o no hay cambios sin guardar**, se importa directamente sin preguntar.

11.5.4. **Detección de archivos incompatibles (V2.2):**
1. Al importar un archivo JSON, el sistema verifica la presencia de campos obsoletos: `data.nets`, `metadata.uiSettings`, `signalTypeRef`, `sectionRef` en conectores, `revision` en metadata, `shield` en signalType de nets.
2. Si se detecta alguno de estos campos, se **rechaza la importación** y se muestra un modal:
   - Título: "Formato no compatible"
   - Mensaje: "Este archivo parece ser de una versión anterior. No se puede importar directamente. Actualice el archivo manualmente al formato V2.3."
   - Opciones: **[Entendido]**
3. No se intenta migración automática.

11.5.5. **Flujo de Borrar todo:**
1. Se abre un **primer modal**: "¿Estás seguro de que quieres borrar todos los datos actuales?"
2. Si confirma, se abre un **segundo modal**: "¿Deseas exportar una copia de seguridad antes de borrar?"
3. Opciones: **[Exportar y borrar]** | **[Borrar sin guardar]** | **[Cancelar]**
4. Solo tras esta doble confirmación se limpian todos los datos y se reinicia la vista.

11.5.6. **Protección de catálogos con referencias (V2.2):**
1. Si el usuario intenta eliminar una entrada de catálogo (persona, modelo, tipo de cable, net, sección, color) que está siendo referenciada por alguna entidad en `data`, el sistema **impide la eliminación**.
2. Se muestra un modal: "No se puede eliminar. Este elemento está siendo usado por [N] entidades: [lista de IDs]."
3. Opciones: **[Entendido]**
4. Si la entrada no tiene referencias, se elimina sin confirmación adicional.

### 11.6 Filtros Intermedios (V2.2)

11.6.1. La interfaz incluye un **panel de filtros** accesible desde la barra de herramientas (sobre el área de trabajo) o desde el panel `config`.

11.6.2. **Controles de filtro:**
- **Campo de texto**: búsqueda por ID, nombre o designator. Coincidencia parcial (substring, case-insensitive).
- **Dropdown "Tipo"**: Todos / Contenedores (T) / Conectores (C) / Wires (W) / Mates (M).
- **Dropdown "Net"**: Todas / (lista de nets del catálogo `catalogs.nets`).
- **Dropdown "Sección"**: Todas / (lista de secciones del catálogo `catalogs.sections`).
- **Botón "Limpiar filtros"**: restablece todos los filtros a su valor por defecto.

11.6.3. **Combinación de filtros:** Los filtros se combinan con lógica **AND**. Un elemento debe cumplir **todos** los filtros activos para ser mostrado.

11.6.4. **Efecto en la vista visual (SVG):**
- Los elementos que **coinciden** con los filtros se muestran con opacidad normal.
- Los elementos que **no coinciden** se atenúan (opacidad `0.15`) pero no se ocultan completamente, para mantener el contexto visual.
- Los cables conectados a conectores atenuados también se atenúan.

11.6.5. **Efecto en la vista tabla:**
- Las filas que **coinciden** con los filtros se muestran normalmente.
- Las filas que **no coinciden** se **ocultan** (no se muestran).
- Se muestra un contador: "Mostrando X de Y entidades".

11.6.6. **Persistencia:** El estado de los filtros se guarda en `localStorage` bajo la clave `arnesviz.filters`. Se restaura al recargar la página.

11.6.7. **Atajo:** La tecla `F` activa/desactiva el foco en el campo de texto de búsqueda.

### 11.7 Atajos de Teclado

| Atajo          | Acción                                    |
|----------------|-------------------------------------------|
| `Ctrl+Shift+E` | Alternar modo edición (toggle en header)  |
| `Ctrl+S`       | Exportar JSON                             |
| `Ctrl+O`       | Importar JSON                             |
| `Esc`          | Cerrar panel lateral / Deseleccionar todo |
| `F`            | Activar/desactivar filtro de búsqueda     |

### 11.8 Estética Visual

11.8.1. **Tema Oscuro (por defecto):**
- Fondo general: `#000000` (negro puro).
- Tarjetas / paneles: `#121212` a `#1e1e1e`.
- Bordes sutiles: `#2a2a2a`.
- Texto principal: `#f0f0f0` (blanco suave).
- Texto secundario: `#9ca3af`.
- Acentos: **Amarillo flúor** (`#ffff00`, `#ffcc00`) sobre negro/gris oscuro.
- Error: `#ef4444` (rojo).
- Advertencia: `#ffcc00` (amarillo).
- Info: `#3b82f6` (azul).

11.8.2. **Efecto Neumórfico sutil:**
- Sombras dobles sobre elementos elevados: `box-shadow: 6px 6px 12px rgba(0,0,0,0.6), -6px -6px 12px rgba(40,50,80,0.15);`
- Elementos hundidos (inputs): `box-shadow: inset 4px 4px 8px rgba(0,0,0,0.5), inset -4px -4px 8px rgba(40,50,80,0.1);`
- Bordes redondeados: `border-radius: 1rem` a `1.5rem`.

11.8.3. **Tema Claro (opcional, menor prioridad):**
- Fondo: `#ffffff`.
- Tarjetas: `#f5f5f5`.
- Acentos: Azul oscuro (`#1e3a8a`) o verde (`#059669`).
- Texto: `#000000` / `#333333`.

11.8.4. **Transiciones:** `transition: all 0.25s ease` en hover, selección, apertura/cierre de paneles.

11.8.5. **Feedback visual:** Hover sobre elementos interactivos (cambio sutil de brillo/sombra). Selección activa (borde amarillo flúor + glow). Elemento en arrastre (opacidad reducida + sombra más profunda).

### 11.9 Persistencia: JSON vs localStorage (V2.1)

11.9.1. **Separación estricta de responsabilidades:**

| Almacenamiento | Contenido | Persistencia |
|----------------|-----------|--------------|
| **`db.json`** (archivo) | Datos del proyecto: `metadata` (projectInfo, schema, catalogs, savedBy, lastSave, version) y `data` (containers, connectors, wires, mates). | Se guarda al exportar. Se versiona en Git. |
| **`localStorage`** (navegador) | Preferencias del usuario: tema, zoom actual, posición del lienzo (pan), filtros aplicados, última vista usada (tabla/visual), estado de expansión de contenedores. | Persiste entre sesiones del mismo navegador. No se exporta. |

11.9.2. **No existe `metadata.uiSettings`** en el JSON. Todas las preferencias de UI son locales del usuario y se guardan en `localStorage`.

11.9.3. **Claves de localStorage sugeridas:**
- `arnesviz.theme`: `"dark"` | `"light"`
- `arnesviz.zoom`: number
- `arnesviz.pan`: `{ x, y }`
- `arnesviz.filters`: objeto con estado de filtros
- `arnesviz.activeView`: `"table"` | `"visual"`
- `arnesviz.expandedContainers`: objeto con IDs de contenedores expandidos
- `arnesviz.lastProject`: blob o referencia al último JSON cargado (para restaurar sesión)
- `arnesviz.autosave`: copia de seguridad del estado actual (ver 11.10.5)

11.9.4. **Manejo de errores de localStorage (V2.2):**
- Todas las operaciones de escritura en `localStorage` se envuelven en `try/catch`.
- Si la escritura falla (ej: cuota de almacenamiento excedida), se muestra una **Advertencia** en el panel de logs: "No se pudieron guardar las preferencias. El almacenamiento local puede estar lleno."
- La aplicación sigue funcionando normalmente; solo se pierde la persistencia de preferencias.
- **Nota para futuras versiones:** Para proyectos muy grandes que excedan la cuota de `localStorage` (~5-10 MB), se sugiere migrar a `IndexedDB` (límite ~50 MB+). Ver sección 12.14.

### 11.10 Gestión de Cierre y Guardado (V2.1)

11.10.1. **Contexto de despliegue (V2.2):** La aplicación **ArnesViz** está alojada en **GitHub Pages** (o similar) y es una aplicación **estática**. Lee un archivo **`db.json`** desde la raíz del repositorio al iniciar.

> **⚠️ Importante (V2.2):** GitHub Pages es un servicio de alojamiento **estático**. No permite escritura directa al repositorio desde el navegador. Para guardar cambios en el archivo `db.json`, el usuario debe:
> 1. Pulsar **Exportar JSON** en la app (se descarga el archivo actualizado).
> 2. Subir manualmente el archivo al repositorio de GitHub (mediante commit + push, o la interfaz web de GitHub).
>
> El autosave en `localStorage` es un mecanismo de seguridad **local** para recuperar el trabajo en caso de cierre inesperado del navegador. **No reemplaza** el commit en Git ni la subida manual del archivo.

11.10.2. **Flujo de inicio:**
1. Al cargar la página, la app intenta fetch de `db.json` desde la raíz del repositorio.
2. Si existe y es válido, se carga el proyecto.
3. Si no existe o es inválido, se muestra una pantalla de bienvenida con botones: **"Importar JSON"** y **"Crear nuevo proyecto"**.

11.10.3. **Flujo de cierre / navegación away:**
1. Si **no hay cambios sin guardar** en los datos del proyecto (`isDirty === false`), la app permite cerrar o navegar sin preguntar.
2. Si **hay cambios sin guardar** (`isDirty === true`), se muestra un modal: "Hay cambios sin guardar. ¿Deseas exportar el archivo actual?"
   - Opciones: **[Exportar]** | **[Descartar y cerrar]** | **[Cancelar]**
3. Si el usuario elige "Exportar", se descarga el JSON actualizado y luego se permite cerrar.
4. Si el usuario elige "Descartar y cerrar", se cierra sin guardar.
5. Si el usuario elige "Cancelar", se vuelve a la app.

11.10.4. **Detección de cambios:** La app mantiene un flag `isDirty` que se activa con cualquier modificación a los datos del proyecto (containers, connectors, wires, mates, catalogs, metadata). Se desactiva al exportar o al importar un nuevo archivo.

11.10.5. **Guardado automático en localStorage:** Como mecanismo de seguridad, la app guarda periódicamente (cada 30 segundos o en cada cambio significativo) una copia del estado actual en `localStorage` bajo la clave `arnesviz.autosave`. Si la app se cierra inesperadamente (crash del navegador), al reiniciar se ofrece restaurar desde el autosave.

### 11.11 Creación y Eliminación de Entidades (V2.3)

11.11.1. **Crear entidad:**
- Botón **"+ Añadir"** en la barra de herramientas de la tabla (visible en modo edición).
- Al pulsarlo, se abre el panel `details` con un formulario vacío del tipo de entidad seleccionado (T, C, W, M).
- El ID se asigna automáticamente: se busca el siguiente número disponible para el prefijo correspondiente (ej: si existen C001-C009, el nuevo será C010).
- Los campos inferibles se autocompletan según el principio de inferencia (ver 1.5).
- Al guardar, la entidad se añade a `data` y se re-renderizan la tabla y el SVG.

11.11.2. **Eliminar entidad:**
- Botón **"Eliminar"** en el panel `details` (solo en modo edición).
- Abre modal de confirmación: "¿Eliminar [tipo] [ID]? Esta acción no se puede deshacer."
- Si la entidad tiene referencias (ej: un conector usado en wires o mates), el modal muestra: "Advertencia: [ID] está referenciado por [lista de IDs]. Eliminarlo causará errores de referencia."
- Opciones: **[Eliminar igualmente]** | **[Cancelar]**.
- Al eliminar, se remueve de `data`, se re-renderizan tabla y SVG, y se revalida el proyecto.

11.11.3. **Duplicar entidad:**
- Botón **"Duplicar"** en el panel `details` (solo en modo edición).
- Crea una copia exacta de la entidad con un nuevo ID auto-asignado.
- Las referencias internas (ej: `from.connector`, `to.connector`) se mantienen apuntando a las mismas entidades originales.
- El usuario debe ajustar manualmente las referencias si es necesario.

### 11.12 Tabla de Datos (V2.3)

11.12.1. La tabla se implementa con **HTML nativo** (`<table>`, `<tr>`, `<td>`) y JavaScript vanilla. No se utilizan librerías externas.

11.12.2. **Estructura:**
- Barra de herramientas superior: selector de tipo de entidad (T / C / W / M), botón "+ Añadir", campo de búsqueda rápida.
- Cuerpo de la tabla: una fila por entidad, con columnas según la configuración de columnas visibles.
- Columnas por defecto: `id`, `name`, `type`, `owner`.

11.12.3. **Edición inline:**
- En modo edición, al hacer doble-click en una celda, se transforma en un input editable.
- `Enter` guarda el cambio. `Esc` cancela.
- Los cambios se sincronizan con el SVG y el panel `details`.

11.12.4. **Resaltado de errores:**
- Las filas de entidades con errores de validación se resaltan con borde rojo (`border: 2px solid #ef4444`).
- Las filas de entidades con advertencias se resaltan con borde amarillo (`border: 2px solid #ffcc00`).
- Al hacer hover sobre la fila, se muestra un tooltip con el mensaje de error/advertencia.

11.12.5. **Sincronización bidireccional:**
- Cambios en la tabla actualizan el SVG y el panel `details`.
- Cambios en el SVG (arrastre, selección) actualizan la tabla y el panel `details`.
- Cambios en el panel `details` actualizan la tabla y el SVG.

### 11.13 Responsive (V2.3)

11.13.1. El panel lateral es **overlay** (flota sobre el contenido, no lo empuja).
11.13.2. En pantallas < 768px, el panel lateral ocupa el **100% del ancho** de la pantalla.
11.13.3. El header se mantiene fijo. El área de trabajo ocupa el resto de la altura.
11.13.4. Los filtros se colapsan en un botón (ícono de embudo) en pantallas pequeñas. Al pulsarlo, se despliega un panel flotante con los controles de filtro.

---

## 12. NOTAS PARA FUTURAS VERSIONES

12.1. Configuración de modo de arranque (lectura/edición) en preferencias.
12.2. Enrutamiento automático de wires con evasión de obstáculos.
12.3. Size visual de conectores adaptable a pines o designator.
12.4. Soportar pares trenzados (campo `pair` en nets del catálogo).
12.5. Puntos de chasis como nodos de tierra implícitos.
12.6. Integración con sistemas Kanban para seguimiento de tareas basadas en notas.
12.7. En caso de requerirse, reintroducción de estados en M (planificado, desconectado, obsoleto) con un modelo más robusto.
12.8. **Umbral de desalineación configurable:** el umbral de 2 píxeles para la validación de alineación de offsets (10.10) podrá ajustarse en el panel de configuración para adaptarse a distintas resoluciones de lienzo o niveles de zoom.
12.9. Sistema de deshacer/rehacer (Undo/Redo) con `Ctrl+Z` / `Ctrl+Y`.
12.10. Exportación a formatos adicionales (PDF, imagen SVG, BOM).
12.11. Integración con herramientas CAD (KiCad, Altium) para importación de netlists.
12.12. Sincronización en tiempo real multiusuario (colaboración).
12.13. **Edición completa de catálogos desde la UI (V2.2):** En V2.2/V2.3, los catálogos son de solo lectura en la interfaz. La edición se realiza directamente en el archivo JSON. En una versión futura, se implementará una UI completa para añadir, editar y eliminar entradas de catálogos (people, sections, connectorModels, wireTypes, nets, colorPalette) con validación de referencias y protección contra eliminación de elementos en uso. Esto incluye la posibilidad de añadir o quitar valores de `signalType` y personalizar la paleta de colores.
12.14. **Migración a IndexedDB (V2.2):** El autosave y las preferencias se guardan en `localStorage`, que tiene un límite de ~5-10 MB. Para proyectos muy grandes, se sugiere migrar a `IndexedDB` (límite ~50 MB+) en una versión futura.
12.15. **Herramienta de migración de archivos (V2.2):** Actualmente, los archivos de versiones anteriores (V1.9.1, V2.0, V2.1, V2.2) son rechazados al importar. En una versión futura, se podría implementar una herramienta de migración automática que convierta archivos antiguos al formato vigente.

> **Nota:** El antiguo punto 12.9 de V1.9.1 (Sistema de registro de eventos / logs) fue promovido a implementación activa en la sección 10.13 de V2.0.

---

## 13. HISTORIAL DE VERSIONES

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

**Versión 1.9** – Resolución del conflicto cinemático y simplificación del redimensionamiento: introducción de `mountType` (`"fixed"` / `"flying"`), conectores volantes sin `offset` propio, redimensionamiento limitado a la esquina inferior derecha, corrección de la validación de `matedId` (bidireccional) y de la alineación de offsets.

**Versión 1.9.1** – Correcciones de consistencia y claridad: restaurada la regla 10.6, mejorada la redacción de 3.3.1.3 y 4.5.3.1, añadida aclaración sobre `parent_id` declarativo en volantes, añadida nota 12.10 sobre umbral configurable.

**Versión 2.0** – Estructura de archivo JSON formal (`metadata` + `data`), catálogos reutilizables, esquema de validación dinámica, gestión de usuarios y responsables, campos aditivos opcionales, interfaz minimalista con header y barra lateral única reutilizable, panel de logs activo con 3 niveles, flujos de importación y borrado con confirmación, atajos de teclado, estética dark mode neumórfica.

**Versión 2.1** – Nets movidas a catálogos, gauge simplificado, conectores volantes sin pareja (regla 10.14), SVG nativo especificado, eliminado `metadata.uiSettings`, separación JSON vs localStorage, flujo de cierre inteligente, herencia de `sectionRef`, alineación por pin 1.

**Versión 2.2** – Toggle de modo edición en header, barra lateral con memoria de estado, eliminada regla de coherencia modelo/instancia, eliminado `sectionRef` de conectores, `metadata.revision` renombrado a `version`, gauge como campo recomendado, conectores fijos sin pareja (Info en logs), wires con extremos inválidos omitidos en SVG, validación unificada, filtros intermedios, protección de catálogos, detección de archivos incompatibles, documentación de GitHub Pages, manejo de errores de localStorage.

**Versión 2.3** – Principio de inferencia desde catálogos (autocompletado automático, sección 1.5), catálogo `colorPalette` para renderizado SVG, schema específico por entidad (required/recommended por tipo), simplificación del catálogo `people` (solo nombre), eliminación de `owners` en `sections`, eliminación de `shield` de `signalType`, `gaugeUnit` opcional con inferencia desde catálogo (default `"mm2"`), `pinMapping` autocompletado como `"direct"` en UI (solo para mates nuevos), limitación de M documentada (dos conectores por acople), `offset` ignorado en volantes (sin advertencia), límite de profundidad jerárquica de 4 niveles con detección de ciclos (regla 10.16), validación 100% custom (sin AJV ni librerías externas), arquitectura single-file (`index.html` + `db.json`), mecánica de pan con botón medio del mouse, fórmula de curvas Bézier documentada, distribución de pines documentada, orden de renderizado (Z-index) documentado, creación/eliminación/duplicación de entidades documentada, tabla nativa HTML documentada, responsive con panel overlay, ejemplo base restaurado (todos los M/W comparten net `GND`), correcciones de referencias cruzadas (3.3.1.3, 4.1.1.b, 4.4.2, 10.13.3), nota histórica sobre renombramiento revision→version, color por defecto cambiado a `"black"`.

---

## APÉNDICE A – JSON DE EJEMPLO COMPLETO (V2.3)

A continuación se muestra un archivo `db.json` completo y válido que implementa todos los conceptos de esta documentación. Representa fielmente los datos del ejemplo de la V1.9.1 (5 contenedores, 9 conectores, 3 wires, 4 mates) enriquecidos con catálogos, schema y las mejoras de V2.3.

> **Nota:** El valor `version: 5` es ilustrativo y representa un proyecto que ha sido guardado 5 veces.

```json
{
  "metadata": {
    "projectInfo": {
      "name": "Moto Eléctrica Z1 - Arnés Principal",
      "description": "Arnés central, desde la ECU hasta sensores críticos. Proyecto MotoStudents.",
      "startDate": "2026-01-15",
      "vehicleModel": "EMoto-Z1 Prototype v1"
    },
    "lastSave": "2026-07-22T10:00:00Z",
    "savedBy": "leo",
    "version": 5,
    "schema": {
      "containers": {
        "required": ["id", "type", "position", "size"],
        "recommended": ["name", "owner", "sectionRef"]
      },
      "connectors": {
        "required": ["id", "type", "gender", "mountType", "edgeSide", "parent_id", "pins"],
        "recommended": ["name", "owner", "modelRef"]
      },
      "wires": {
        "required": ["id", "type", "from", "to", "net"],
        "recommended": ["name", "owner", "gauge", "color", "wireTypeRef"]
      },
      "mates": {
        "required": ["id", "type", "from", "to", "net"],
        "recommended": ["name", "owner", "pinMapping"]
      },
      "rules": {
        "id": { "unique": true, "pattern": "^[TCWM]\\d{3}$" },
        "containers.parent_id": { "ref": "containers" },
        "connectors.parent_id": { "ref": "containers" },
        "connectors.modelRef": { "ref": "connectorModels" },
        "connectors.owner": { "ref": "people" },
        "wires.from.connector": { "ref": "connectors" },
        "wires.to.connector": { "ref": "connectors" },
        "wires.net": { "ref": "nets" },
        "wires.wireTypeRef": { "ref": "wireTypes" },
        "wires.owner": { "ref": "people" },
        "mates.from.connector": { "ref": "connectors" },
        "mates.to.connector": { "ref": "connectors" },
        "mates.net": { "ref": "nets" },
        "mates.owner": { "ref": "people" }
      }
    },
    "catalogs": {
      "people": {
        "leo": { "name": "Leo" },
        "martin": { "name": "Martin" },
        "nico": { "name": "Nico" }
      },
      "sections": {
        "ccu": { "name": "Central Control Unit" },
        "sensors": { "name": "Safety Sensors" },
        "power": { "name": "Power Distribution" }
      },
      "connectorModels": {
        "MOLEX_2P_MALE": {
          "manufacturer": "Molex",
          "partNumber": "02-99-1021",
          "pins": 2,
          "gender": "male",
          "type": "Wire-to-Board",
          "datasheetUrl": "http://example.com/molex_2p_m.pdf",
          "specs": { "maxCurrent": "4A", "voltageRating": "250V" }
        },
        "MOLEX_2P_FEMALE": {
          "manufacturer": "Molex",
          "partNumber": "02-99-2021",
          "pins": 2,
          "gender": "female",
          "type": "Wire-to-Board",
          "datasheetUrl": "http://example.com/molex_2p_f.pdf",
          "specs": { "maxCurrent": "4A", "voltageRating": "250V" }
        },
        "GX12_2P_MALE": {
          "manufacturer": "Various",
          "partNumber": "GX12-2P-M",
          "pins": 2,
          "gender": "male",
          "type": "Circular",
          "datasheetUrl": null,
          "specs": {}
        },
        "GX12_2P_FEMALE": {
          "manufacturer": "Various",
          "partNumber": "GX12-2P-F",
          "pins": 2,
          "gender": "female",
          "type": "Circular",
          "datasheetUrl": null,
          "specs": {}
        },
        "GX12_4P_MALE": {
          "manufacturer": "Various",
          "partNumber": "GX12-4P-M",
          "pins": 4,
          "gender": "male",
          "type": "Circular",
          "datasheetUrl": null,
          "specs": {}
        },
        "GX12_4P_FEMALE": {
          "manufacturer": "Various",
          "partNumber": "GX12-4P-F",
          "pins": 4,
          "gender": "female",
          "type": "Circular",
          "datasheetUrl": null,
          "specs": {}
        },
        "MOLEX_5P_MALE": {
          "manufacturer": "Molex",
          "partNumber": "02-99-1051",
          "pins": 5,
          "gender": "male",
          "type": "Wire-to-Board",
          "datasheetUrl": null,
          "specs": {}
        },
        "MOLEX_5P_FEMALE": {
          "manufacturer": "Molex",
          "partNumber": "02-99-2051",
          "pins": 5,
          "gender": "female",
          "type": "Wire-to-Board",
          "datasheetUrl": null,
          "specs": {}
        }
      },
      "wireTypes": {
        "AWG22_SHIELDED": {
          "unit": "AWG",
          "shielded": true,
          "insulationType": "XLPO",
          "temperatureRating": "125°C",
          "standards": ["SAE J1128"]
        },
        "AWG22_UNSHIELDED": {
          "unit": "AWG",
          "shielded": false,
          "insulationType": "XLPO",
          "temperatureRating": "105°C",
          "standards": ["SAE J1128"]
        },
        "MM2_0.5_SHIELDED": {
          "unit": "mm2",
          "shielded": true,
          "insulationType": "XLPO",
          "temperatureRating": "125°C",
          "standards": ["ISO 6722"]
        }
      },
      "nets": {
        "GND": {
          "name": "System Ground",
          "signalType": "ground",
          "voltage": "0V",
          "colorCode": "black",
          "description": "Masa del sistema"
        }
      },
      "colorPalette": {
        "black": "#000000",
        "red": "#dc2626",
        "blue": "#2563eb",
        "green": "#16a34a",
        "yellow": "#eab308",
        "orange": "#ea580c",
        "white": "#f5f5f5",
        "gray": "#6b7280",
        "brown": "#92400e",
        "violet": "#7c3aed",
        "Green/White": "#22c55e",
        "White/Green": "#e5e7eb"
      }
    }
  },
  "data": {
    "containers": [
      {
        "id": "T100",
        "type": "system",
        "name": "Moto Eléctrica",
        "parent_id": null,
        "designator": "EMOTO-Z1",
        "position": { "x": 50, "y": 50 },
        "size": { "width": 2600, "height": 1200 },
        "owner": "leo",
        "sectionRef": null,
        "notes": []
      },
      {
        "id": "T200",
        "type": "enclosure",
        "name": "Caja 1",
        "parent_id": "T100",
        "designator": "BOX1",
        "position": { "offsetX": 90, "offsetY": 110 },
        "size": { "width": 950, "height": 1000 },
        "owner": "martin",
        "sectionRef": "ccu",
        "notes": []
      },
      {
        "id": "T201",
        "type": "enclosure",
        "name": "Caja 2",
        "parent_id": "T100",
        "designator": "BOX2",
        "position": { "offsetX": 1593, "offsetY": 122 },
        "size": { "width": 950, "height": 1000 },
        "owner": "nico",
        "sectionRef": "sensors",
        "notes": []
      },
      {
        "id": "T300",
        "type": "pcb",
        "name": "PCB 1",
        "parent_id": "T200",
        "designator": "PCB1",
        "position": { "offsetX": 80, "offsetY": 100 },
        "size": { "width": 368, "height": 837 },
        "owner": "martin",
        "sectionRef": "ccu",
        "notes": []
      },
      {
        "id": "T301",
        "type": "pcb",
        "name": "PCB 2",
        "parent_id": "T201",
        "designator": "PCB2",
        "position": { "offsetX": 427, "offsetY": 117 },
        "size": { "width": 472, "height": 817 },
        "owner": "nico",
        "sectionRef": "sensors",
        "notes": []
      }
    ],
    "connectors": [
      {
        "id": "C001",
        "type": "connector",
        "name": "Molex 2P",
        "parent_id": "T300",
        "designator": "J1",
        "pins": 2,
        "gender": "male",
        "mountType": "fixed",
        "edgeSide": "right",
        "offset": 100,
        "size": { "width": 180, "height": 115 },
        "matedId": "M001",
        "modelRef": "MOLEX_2P_MALE",
        "owner": "martin",
        "notes": []
      },
      {
        "id": "C002",
        "type": "connector",
        "name": "Molex 2P",
        "parent_id": "T200",
        "designator": "J2",
        "pins": 2,
        "gender": "female",
        "mountType": "flying",
        "edgeSide": "left",
        "offset": null,
        "size": { "width": 180, "height": 115 },
        "matedId": "M001",
        "modelRef": "MOLEX_2P_FEMALE",
        "owner": "martin",
        "notes": []
      },
      {
        "id": "C003",
        "type": "connector",
        "name": "GX12",
        "parent_id": "T200",
        "designator": "J3",
        "pins": 2,
        "gender": "female",
        "mountType": "fixed",
        "edgeSide": "right",
        "offset": 740,
        "size": { "width": 180, "height": 115 },
        "matedId": "M002",
        "modelRef": "GX12_2P_FEMALE",
        "owner": "martin",
        "notes": []
      },
      {
        "id": "C004",
        "type": "connector",
        "name": "GX12",
        "parent_id": "T100",
        "designator": "J4",
        "pins": 2,
        "gender": "male",
        "mountType": "flying",
        "edgeSide": "left",
        "offset": null,
        "size": { "width": 180, "height": 115 },
        "matedId": "M002",
        "modelRef": "GX12_2P_MALE",
        "owner": "leo",
        "notes": []
      },
      {
        "id": "C005",
        "type": "connector",
        "name": "GX12 4P",
        "parent_id": "T100",
        "designator": "J5",
        "pins": 4,
        "gender": "male",
        "mountType": "flying",
        "edgeSide": "right",
        "offset": null,
        "size": { "width": 180, "height": 115 },
        "matedId": "M003",
        "modelRef": "GX12_4P_MALE",
        "owner": "leo",
        "notes": []
      },
      {
        "id": "C006",
        "type": "connector",
        "name": "GX12 4P",
        "parent_id": "T201",
        "designator": "J6",
        "pins": 4,
        "gender": "female",
        "mountType": "fixed",
        "edgeSide": "left",
        "offset": 728,
        "size": { "width": 180, "height": 115 },
        "matedId": "M003",
        "modelRef": "GX12_4P_FEMALE",
        "owner": "nico",
        "notes": []
      },
      {
        "id": "C007",
        "type": "connector",
        "name": "Molex 5P",
        "parent_id": "T201",
        "designator": "J7",
        "pins": 5,
        "gender": "male",
        "mountType": "flying",
        "edgeSide": "left",
        "offset": null,
        "size": { "width": 180, "height": 115 },
        "matedId": "M004",
        "modelRef": "MOLEX_5P_MALE",
        "owner": "nico",
        "notes": []
      },
      {
        "id": "C008",
        "type": "connector",
        "name": "Molex 5P",
        "parent_id": "T301",
        "designator": "J8",
        "pins": 5,
        "gender": "female",
        "mountType": "fixed",
        "edgeSide": "right",
        "offset": 71,
        "size": { "width": 180, "height": 115 },
        "matedId": "M004",
        "modelRef": "MOLEX_5P_FEMALE",
        "owner": "nico",
        "notes": []
      },
      {
        "id": "C009",
        "type": "connector",
        "name": "Molex 2P",
        "parent_id": "T301",
        "designator": "J9",
        "pins": 2,
        "gender": "female",
        "mountType": "fixed",
        "edgeSide": "left",
        "offset": 251,
        "size": { "width": 180, "height": 115 },
        "matedId": null,
        "modelRef": "MOLEX_2P_FEMALE",
        "owner": "nico",
        "notes": [
          {
            "date": "2026-07-12",
            "user": "leo",
            "text": "Reserva para faro auxiliar"
          }
        ]
      }
    ],
    "wires": [
      {
        "id": "W001",
        "type": "wired",
        "from": { "connector": "C002", "pin": 1 },
        "to": { "connector": "C003", "pin": 1 },
        "net": "GND",
        "length": 150,
        "gauge": 22,
        "gaugeUnit": "AWG",
        "color": "black",
        "thickness": 2,
        "wireTypeRef": "AWG22_SHIELDED",
        "owner": "martin",
        "notes": []
      },
      {
        "id": "W002",
        "type": "wired",
        "from": { "connector": "C004", "pin": 1 },
        "to": { "connector": "C005", "pin": 1 },
        "net": "GND",
        "length": 400,
        "gauge": 22,
        "gaugeUnit": "AWG",
        "color": "black",
        "thickness": 2,
        "wireTypeRef": "AWG22_UNSHIELDED",
        "owner": "leo",
        "notes": []
      },
      {
        "id": "W003",
        "type": "wired",
        "from": { "connector": "C006", "pin": 1 },
        "to": { "connector": "C007", "pin": 2 },
        "net": "GND",
        "length": 180,
        "gauge": 22,
        "gaugeUnit": "AWG",
        "color": "black",
        "thickness": 2,
        "wireTypeRef": "AWG22_SHIELDED",
        "owner": "nico",
        "notes": []
      }
    ],
    "mates": [
      {
        "id": "M001",
        "type": "mated",
        "from": { "connector": "C001", "pin": 1 },
        "to": { "connector": "C002", "pin": 1 },
        "net": "GND",
        "pinMapping": "direct",
        "owner": "martin",
        "notes": []
      },
      {
        "id": "M002",
        "type": "mated",
        "from": { "connector": "C004", "pin": 1 },
        "to": { "connector": "C003", "pin": 1 },
        "net": "GND",
        "pinMapping": "direct",
        "owner": "leo",
        "notes": []
      },
      {
        "id": "M003",
        "type": "mated",
        "from": { "connector": "C005", "pin": 1 },
        "to": { "connector": "C006", "pin": 1 },
        "net": "GND",
        "pinMapping": "direct",
        "owner": "nico",
        "notes": []
      },
      {
        "id": "M004",
        "type": "mated",
        "from": { "connector": "C007", "pin": 2 },
        "to": { "connector": "C008", "pin": 2 },
        "net": "GND",
        "pinMapping": null,
        "owner": "nico",
        "notes": []
      }
    ]
  }
}
```

---

## APÉNDICE B – RESUMEN DE CAMBIOS V2.2 → V2.3

| Cambio | V2.2 | V2.3 |
|--------|------|------|
| **Principio de inferencia** | No existía | **Nueva sección 1.5**: autocompletado desde catálogos |
| **Catálogo `people`** | name, alias, role, notes | **Solo `name`** |
| **Catálogo `sections`** | name, owners | **Solo `name`** (owners eliminado) |
| **`signalType` en nets** | power, ground, data, communication, analog, shield | **shield eliminado**. Lista extensible |
| **Catálogo `colorPalette`** | No existía | **Nuevo**: mapeo de nombres a hexadecimales |
| **Schema** | Genérico (`required: ["id", "type"]`) | **Específico por entidad** (containers, connectors, wires, mates) |
| **`gaugeUnit` en wires** | Recomendado, default "mm2" | **Opcional**, inferido del catálogo si wireTypeRef existe, default "mm2" |
| **`color` en wires** | Defecto "blue" | **Defecto "black"**, autocompletado desde net |
| **`pinMapping` en mates** | null o no definido | **Autocompletado como "direct"** en UI (solo mates nuevos). Mates existentes mantienen su valor |
| **Limitación de M** | No documentada | **Documentada**: dos conectores por acople. Casos atípicos con dos M separados |
| **`offset` en volantes** | null (no especificado comportamiento con valor) | **Ignorado** sin advertencia |
| **Profundidad jerárquica** | Sin límite | **Máximo 4 niveles** + detección de ciclos |
| **Validación** | AJV mencionado | **100% custom** (sin librerías externas) |
| **Arquitectura** | No especificada | **Single file**: `index.html` + `db.json` |
| **Pan en SVG** | No especificado | **Botón medio del mouse** (o Shift + click izquierdo) |
| **Curvas Bézier** | No especificado | **Fórmula documentada** (puntos de control según edgeSide) |
| **Distribución de pines** | No especificado | **Documentada**: vertical/horizontal, espaciado, pin 1 arriba/izquierda |
| **Z-index** | No especificado | **Orden documentado**: fondo → contenedores → conectores → cables → pines → overlay |
| **Crear/Eliminar/Duplicar** | No especificado | **Documentado**: botones en tabla y panel details |
| **Tabla de datos** | No especificado | **HTML nativa**, edición inline, resaltado de errores |
| **Responsive** | No especificado | **Panel overlay**, 100% ancho en móvil |
| **Ejemplo base** | Nets variadas (GND, +12V, +5V) | **Restaurado**: todos comparten `GND` |
| **Alineación pin 1** | Sin nota sobre orientación | **Nota añadida**: asume misma orientación de numeración |
| **Referencias cruzadas** | Algunas imprecisas | **Corregidas**: 3.3.1.3, 4.1.1.b, 4.4.2, 10.13.3 |
| **Nota histórica version** | No existía | **Añadida** en 2.7.6: revision→version en V2.2 |

---

**FIN DE LA DOCUMENTACIÓN – VERSIÓN 2.3**
