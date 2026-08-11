# 📊 Proyecto: Negocio de Belleza - Arquitectura de Datos y Automatización Financiera

## 🏗️ Fundamento, Arquitectura y Entorno

Este proyecto documenta la transición de un sistema de registro no estructurado (hojas de cálculo) a un modelo relacional robusto. A diferencia de trabajar en un entorno donde el dato crudo, la fórmula matemática y la presentación visual coexisten en la misma celda, esta base de datos aplica una separación estricta: los datos puros se almacenan en **tablas normalizadas** y la matemática transaccional se procesa en **vistas lógicas**.

*   **Motor de Base de Datos (Servidor):** PostgreSQL.
*   **Interfaz de Gestión (Cliente):** pgAdmin.
*   **Diseño Principal:** Normalización (Data Diet) para eliminar redundancia mediante catálogos dimensionales, tablas de hechos y reglas estrictas de integridad referencial (Primary Keys y Foreign Keys).

### 🛠️ Tipos de Datos Estratégicos Implementados
En lugar de permitir texto libre, se estructuraron columnas con reglas de negocio estrictas para garantizar la integridad financiera y operativa:
*   `SERIAL`: Contadores automáticos irrepetibles para delegar al sistema la creación de identidades únicas.
*   `VARCHAR` / `TEXT`: Limitación inteligente de longitud de campos (ej. folio del ticket o teléfono) para optimizar memoria.
*   `NUMERIC(Precisión, Escala)`: El estándar de oro para cálculos exactos en costos e ingresos, evitando pérdidas por errores de redondeo.
*   `BOOLEAN`: Solución lógica para manejar gratuidades y excepciones de clientes sin ensuciar la captura de ingresos.
*   `DATE` / `TIME`: Estandarización cronológica para agrupar cortes de caja y medir tiempos operativos.

---

## 🗄️ Diccionario de Datos: Tablas Dimensionales

### 1. `catalogo_servicios`
Catálogo maestro que almacena la oferta del estudio. Define tiempos teóricos y parámetros de negocio (CRM) para retoques.

| Nombre del Campo | Tipo de Dato | Llave | Descripción / Regla de Negocio |
| :--- | :---: | :---: | :--- |
| `id_servicio` | `SERIAL (INT)` | **PK** | Identificador único autoincremental de cada servicio. |
| `nombre_servicio` | `VARCHAR` | - | Nombre comercial del procedimiento ofrecido. |
| `descripción` | `TEXT` | - | Descripción breve del servicio. |
| `categoría` | `VARCHAR` | - | Agrupación lógica (ej. Pestañas, Uñas) para análisis de ventas. |
| `duración` | `INT` | - | Tiempo teórico estándar (en minutos). Funciona como "salvavidas" si la cita real carece de tiempos de registro. |
| `retoque` | `BOOLEAN` | - | Bandera (TRUE/FALSE). Indica si el servicio permite mantenimiento a menor costo. |
| `dias_min_retoque` | `INT` | - | Umbral inferior de días para considerar un retoque válido. |
| `dias_max_retoque` | `INT` | - | Umbral superior. Si se excede, el sistema exige un servicio nuevo. |

> **Dependencias:** Consultada vía `INNER JOIN` por `registro_citas` y `vista_crm_retoques`.

### 2. `clientes`
Controla reglas de excepción de cobro y auditoría de tiempos operativos.

| Nombre de Columna | Tipo de Dato | Llave | Descripción / Regla de Negocio |
| :--- | :---: | :---: | :--- |
| `id_cliente` | `SERIAL (INT)` | **PK** | Identificador único del cliente. |
| `nombre` | `VARCHAR` | - | Nombre del cliente. |
| `apellido` | `VARCHAR` | - | Primer apellido del cliente. |
| `teléfono` | `VARCHAR` | - | Contacto primario para citas y seguimiento CRM. |
| `gratuidad` | `BOOLEAN` | - | Bandera de cortesía. Si es TRUE, justifica un cobro de $0.00 y fuerza el uso del tiempo teórico en reportes. |

### 3. `inventario_insumos`
Catálogo de control de almacén que estandariza las presentaciones comerciales para el cálculo automatizado de costos unitarios.

| Nombre de Columna | Tipo de Dato | Llave | Descripción / Regla de Negocio |
| :--- | :---: | :---: | :--- |
| `id_insumo` | `SERIAL (INT)` | **PK** | Identificador único del producto. |
| `nombre_insumo` | `VARCHAR` | - | Nombre descriptivo del material. |
| `tipo_insumo` | `VARCHAR` | - | Clasificación operativa (ej. 'Desechable', 'Herramienta'). Filtra activos reutilizables del costeo variable. |
| `cantidad_paquete` | `NUMERIC(10,2)` | - | Volumen por presentación para calcular el costo unitario real. |
| `unidad_medida` | `VARCHAR` | - | Unidad base de consumo (ml, gr, pieza). |

---

## 📊 Diccionario de Datos: Tablas de Hechos (Fact Tables)

### 1. `registro_citas`
Tabla transaccional principal. Centraliza la operación diaria y es la fuente de verdad (Source of Truth) para ingresos.

| Nombre de Columna | Tipo de Dato | Llave | Descripción / Regla de Negocio |
| :--- | :---: | :---: | :--- |
| `id_cita` | `SERIAL (INT)` | **PK** | Identificador de la transacción. |
| `folio_ticket` | `VARCHAR` | - | Identificador de la venta. Debe ser `NULL` si el servicio no se concretó. |
| `id_cliente` | `INT` | **FK** | Vinculado a `clientes`. |
| `id_servicio` | `INT` | **FK** | Vinculado a `catalogo_servicios`. |
| `fecha` | `DATE` | - | Fecha calendarizada (YYYY-MM-DD). |
| `hora_cita` | `TIME` | - | Hora agendada originalmente. |
| `hora_llegada` / `salida` | `TIME` | - | Horas reales. La diferencia determina el tiempo invertido. |
| `estatus` | `VARCHAR` | - | Restringido vía `CHECK` a: 'Asistencia', 'Cancelada', 'Pospuesta'. |
| `precio_cobrado` | `NUMERIC(10,2)` | - | Monto bruto. $0.00 si aplica `gratuidad`. |
| `metodo_pago` | `VARCHAR` | - | Efectivo, Transferencia, Terminal, etc. |

### 2. `registro_costos_fijos`
Bitácora financiera de egresos operativos exógenos. Su diseño mensualizado permite flexibilidad ante la inflación.

| Nombre de Columna | Tipo de Dato | Llave | Descripción / Regla de Negocio |
| :--- | :---: | :---: | :--- |
| `id_costo` | `SERIAL (INT)` | **PK** | Identificador del registro. |
| `año` / `mes` | `INT` | - | Año y mes fiscal del gasto. |
| `concepto` | `VARCHAR` | - | Descripción del egreso (renta, luz). |
| `importe` | `NUMERIC(10,2)` | - | Valor desembolsado. |

### 3. `compras_insumos`
Historial de abastecimiento. Habilita el modelo de "Costo de Reposición" (rentabilidad basada en precios de mercado recientes).

| Nombre de Columna | Tipo de Dato | Llave | Descripción / Regla de Negocio |
| :--- | :---: | :---: | :--- |
| `id_compra` | `SERIAL (INT)` | **PK** | Transacción de compra. |
| `id_insumo` | `INT` | **FK** | Referencia al material adquirido. |
| `fecha` | `DATE` | - | El motor busca el `MAX(fecha)` para aplicar el costo actualizado. |
| `costo` | `NUMERIC(10,4)` | - | Costo por paquete. |

### 4. `receta_servicios` (Junction Table)
Resuelve la relación muchos-a-muchos, definiendo el *Bill of Materials* (BOM) por servicio.

| Nombre de Columna | Tipo de Dato | Llave | Descripción / Regla de Negocio |
| :--- | :---: | :---: | :--- |
| `id_receta` | `SERIAL (INT)` | **PK** | Asignación de receta. |
| `id_servicio` | `INT` | **FK** | Servicio a costear. |
| `id_insumo` | `INT` | **FK** | Material consumido. |
| `consumo` | `NUMERIC(10,4)` | - | Porción teórica estandarizada. |
| `merma` | `NUMERIC(5,2)` | - | Desperdicio (ej. 10%) aumentado al insumo. |

---

## 🧠 Motor Analítico (Vistas Lógicas)

Las Vistas (Views) funcionan como consultas pre-guardadas para la **Automatización Financiera** y el enmascaramiento de datos (Data Masking) para proteger la privacidad del cliente.

### `vista_costos_servicios` (Motor de Costeo Dinámico)
Calcula el "Costo de Reposición" asegurando que la rentabilidad se mida con precios actuales, no históricos.
*   **Filtro:** Excluye herramientas.
*   **Lógica CTE (`ultimas_compras`):** Usa `MAX(fecha)` para extraer el precio unitario de la compra más reciente.
*   **Matemática:** `SUM(consumo * precio_unitario_actualizado)` agrupado por servicio.

### `vista_financiera_ticket` (Procesador Transaccional Diario)
Transforma citas en crudo a tickets financieros auditados y calcula rentabilidad neta.
*   **Regla de Tiempo Real:** Usa `hora_salida - hora_llegada`. Si la bandera `gratuidad = TRUE` o hay campos nulos/cero, el sistema inyecta el tiempo teórico del catálogo.
*   **Desglose Fiscal:** Calcula `Ingreso Bruto / 1.16` y determina la utilidad neta restando los costos variables.

### `vista_resumen_mensual` (Estado de Resultados P&L Automatizado)
Agrupa la operación en renglones mensuales cruzando utilidad operativa con gastos fijos. Preparada para ingesta en Power BI.
*   **Agrupación:** Uso de `EXTRACT(YEAR)` y `EXTRACT(MONTH)`.
*   **Protección Nulos:** Uso de `COALESCE()` para evitar colapsos matemáticos si un mes no tiene registro de costos fijos.
*   **Métricas Calculadas:** % de Ocupación, Ticket Promedio e EBITDA.

### `vista_crm_retoques` (Modelo de Predicción de Mantenimiento)
Monitorea el ciclo de vida del cliente para detectar ventas cruzadas.
*   **Lógica:** Extrae `MAX(fecha)` por cliente y resta `CURRENT_DATE` para obtener los días transcurridos.
*   **Dictamen Técnico (`CASE`):** Clasifica automáticamente en *"Muy pronto"*, *"En tiempo ideal"* o *"Excedido (Requiere set nuevo)"*.
