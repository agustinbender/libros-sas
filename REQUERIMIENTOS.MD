# REQUERIMIENTO.MD  
**Producto:** Gestor de libros societarios con IA para SAS (Argentina)  
**Versión:** 0.1 (MVP)  
**Autor:** _(por definir)_  
**Fecha:** _(por definir)_

---

## 1. Objetivo del sistema

Desarrollar una aplicación que permita gestionar los **libros societarios digitales de una SAS en Argentina**, reduciendo al mínimo la carga manual de datos mediante el uso de IA, y manteniendo:

- La **trazabilidad** de los actos societarios.
- La **cadena de hashes** exigida para libros digitales.
- La **validez formal** frente al esquema de registro de hash en Trámites a Distancia (TAD).

---

## 2. Alcance

Incluye:

- Gestión de múltiples **sociedades** (SAS) en una misma cuenta.
- Ingesta masiva de **PDFs existentes** (estatuto, modificaciones, actas, libros IVA, balances, etc.).
- Clasificación y extracción automática de datos societarios vía IA.
- Generación y mantenimiento de:
  - **Libros digitales**.
  - **Actos** (actas, registros, balances).
  - **Contexto de la sociedad** (documento textual vivo).
- Generación asistida de nuevos actos usando IA y un **repositorio de templates**.
- Cálculo de **hash** y control de **cadena de hashes**.
- Flujo de **registro de hash en TAD** y validación mediante comprobante.
- Gestión de **libros IVA compras / IVA ventas** (hash + encabezado).
- Roles y permisos, borrado lógico, auditoría y trazabilidad.

No incluye (MVP):

- Firma digital de documentos dentro de la app.
- Integración directa con TAD (carga/verificación automática).
- Control de legalidad profundo (solo controles formales y de consistencia básica).

---

## 3. Actores y roles

### 3.1 Superadmin

- Acceso a **todas** las sociedades del sistema.
- Gestiona:
  - Usuarios de nivel superadmin.
  - Alta/baja de sociedades (a nivel global).
  - **Listado de templates** (creación, edición, activación/desactivación).
- Acceso total a todos los módulos y datos (lectura y escritura).

### 3.2 Owner de Sociedad

- “Dueño” de una sociedad específica.
- Puede:
  - Configurar la sociedad.
  - Cargar PDFs para onboarding.
  - Crear, editar y cerrar actos de esa sociedad.
  - Gestionar usuarios asociados a esa sociedad (invitar, asignar roles RW/RO).
  - Editar el **Contexto de la sociedad**.

### 3.3 Usuario Read-Write (RW)

- Asociado a una o más sociedades.
- Puede:
  - Ver todos los libros y actos de la(s) sociedad(es) a la(s) que tenga acceso.
  - Crear/editar actos y disparar generación de PDFs.
  - Subir comprobantes TAD.
  - Proponer/editar cambios en el Contexto (según permisos definidos).

### 3.4 Usuario Read-Only (RO)

- Asociado a una o más sociedades.
- Solo puede:
  - Ver libros, actos, documentos PDF, hashes y contexto.
  - No puede modificar ni crear.

---

## 4. Conceptos clave

- **Sociedad**: entidad jurídica (SAS) con CUIT, denominación, etc.
- **Libro digital**: conjunto de documentos (PDF) correlacionados a un tipo de libro (actas, registro de acciones, diario, inventarios y balances, IVA compras, IVA ventas).
- **Acto / documento de libro**: acta, registro, balance u otro documento que pertenece a un libro y tiene hash propio.
- **Contexto de la sociedad**: documento textual vivo que resume la situación societaria (datos básicos, estructura accionaria, autoridades, hitos).
- **Hash**: valor de hash (ej. SHA-256) del PDF final, utilizado para registro en TAD y para encadenar documentos.
- **TAD**: Trámites a Distancia, plataforma donde se registra el hash de los documentos.
- **Template**: plantilla parametrizable de acto (ej. “Acta de asamblea – aprobación de estados contables”), gestionada por superadmin, utilizada por la IA para generar textos.

---

## 5. Requerimientos funcionales

### 5.1 Gestión de usuarios y roles

**FR-01 – Gestión de roles**

- El sistema debe soportar los siguientes roles:
  - **SuperAdmin**: Acceso global a todas las sociedades.
  - **Owner**: Dueño de una sociedad específica.
  - **Read-Write (RW)**: Puede ver y modificar.
  - **Read-Only (RO)**: Solo puede ver.
- Cada usuario debe tener un rol global (SuperAdmin o no) y permisos por sociedad (Owner / RW / RO).
- Nota: Profesionales externos (contadores, abogados) se agregan como usuarios normales con el rol que corresponda.

**FR-02 – Asociación usuario–sociedad**

- El Superadmin puede:
  - Crear sociedades (vacías inicialmente).
  - Asignar un Owner para cada sociedad.
- El Owner puede:
  - Invitar usuarios por correo.
  - Asignarles rol RW o RO sobre su sociedad.
  - Revocar accesos.

**FR-03 – Borrado lógico de usuarios**

- La baja de un usuario debe ser lógica:
  - No se eliminan sus acciones históricas ni registros de auditoría.
  - Se marca como inactivo y no puede volver a iniciar sesión.

---

### 5.2 Onboarding y carga masiva de documentos (mínimo input)

**FR-04 – Creación de sociedad con mínimo input**

- Para crear una nueva sociedad, el usuario solo debe proporcionar:
  - Un nombre identificatorio (que puede ser provisorio).
- Opcionalmente puede ingresar CUIT, pero no es obligatorio (se intentará extraer de documentos).

**FR-05 – Carga masiva de PDFs**

- El Owner debe poder cargar una **carpeta completa de PDFs** (drag & drop) que contenga:
  - Estatuto / contrato constitutivo.
  - Modificaciones de estatuto.
  - Actas de socios / asamblea / directorio / gerencia.
  - Registro de acciones.
  - Balances / inventarios.
  - Libros IVA, otros.
- La app solo debe pedir confirmación de que esos son los documentos societarios a procesar.

**FR-06 – Clasificación automática de documentos**

- El sistema, utilizando IA, debe clasificar cada PDF en una de las categorías:
  - Estatuto / Modificación de estatuto.
  - Constancia de registración IGJ (plancha de certificación).
  - Acta de socios/asamblea.
  - Acta de administración (directorio/gerencia).
  - Registro de acciones / movimiento accionario.
  - Balance / inventario.
  - Libro IVA compras.
  - Libro IVA ventas.
  - Comprobante TAD (registro de hash).
  - Otro / sin clasificar (si no hay suficientes señales).
- **Detección de PDFs embebidos:**
  - Los documentos de IGJ frecuentemente contienen el documento original como PDF adjunto/embebido.
  - El sistema debe detectar y extraer automáticamente estos PDFs embebidos.
  - Procesar tanto el documento principal como los adjuntos.

**FR-07 – Creación automática de libros digitales**

- A partir de la clasificación, el sistema debe:
  - Crear automáticamente los libros digitales necesarios para la sociedad (si no existen).
  - Asociar cada documento a su libro correspondiente.

---

### 5.3 Extracción de datos y Contexto de la sociedad

**FR-08 – Extracción automática de datos de la sociedad**

- A partir del estatuto, constancia de registración IGJ y sus modificaciones, el sistema debe extraer y completar automáticamente:
  - **Datos básicos:**
    - Denominación social completa.
    - CUIT de la sociedad (ej: 30716713357).
    - Número de inscripción IGJ (ej: RL-2019-110008094-APN-DA#IGJ).
    - Domicilio legal (jurisdicción).
    - Sede social (dirección específica).
    - Fecha de constitución.
    - Duración de la sociedad (años).
    - Fecha de cierre de ejercicio.
  - **Capital y acciones:**
    - Capital social inicial.
    - Capital social vigente (si hubo aumentos).
    - Cantidad y tipo de acciones.
    - Valor nominal por acción.
  - **Socios/Accionistas:**
    - Nombre completo de cada socio.
    - DNI/CUIT de cada socio.
    - Porcentaje de participación.
    - Cantidad de acciones por socio.
  - **Autoridades:**
    - Administradores titulares (nombre, DNI, cargo).
    - Administradores suplentes (nombre, DNI, cargo).
    - Órgano de fiscalización (si aplica).
  - **Reglas de funcionamiento:**
    - Estructura básica de órganos.
    - Reglas de quórum y mayorías.
    - Reglas de transferencia de acciones.
  - **Objeto social:** Resumen de las actividades principales.

**FR-09 – Campos editables con trazabilidad**

- Todos los campos extraídos (como los de FR-08) deben ser:
  - Editables manualmente por Owner y RW.
  - Con registro de:
    - Valor anterior,
    - Valor nuevo,
    - Usuario y fecha de modificación.

**FR-10 – Generación inicial del Contexto de la sociedad**

- El sistema debe generar automáticamente un documento textual de Contexto que incluya:
  - Datos generales (denominación, CUIT, domicilio, constitución, duración, cierre de ejercicio).
  - Estructura accionaria vigente estimada.
  - Autoridades vigentes estimadas (órganos, personas, períodos).
  - Lista resumida de hitos (constitución, aumentos de capital, reformas de estatuto, cambios de autoridades, etc.).

**FR-11 – Actualización continua del Contexto**

- Cada vez que un acto llegue a estado **Final**, la IA debe:
  - Analizar el acto.
  - Proponer cambios al Contexto (p.ej. actualización de autoridades, capital).
- El usuario (Owner/RW) debe poder:
  - Aceptar los cambios propuestos tal cual.
  - Editarlos antes de guardar.
- Debe quedar un historial de versiones del Contexto (fecha, usuario, resumen de cambios).

---

### 5.4 Libros, actos y estados

**FR-12 – Tipos de libros soportados**

- Para cada sociedad el sistema debe soportar al menos los siguientes libros digitales:
  - Libro de Actas de Socios / Asamblea.
  - Libro de Actas de Administración (directorio/gerencia).
  - Libro de Registro de Acciones.
  - Libro Diario.
  - Libro de Inventarios y Balances.
  - Libro IVA Compras.
  - Libro IVA Ventas.

**FR-13 – Entidad Acto / Documento de libro**

Cada acto/documento debe almacenar como mínimo:

- Libro al que pertenece.
- Tipo de acto (ej. acta de socios, acta de directorio, registro accionarios, balance, IVA compras/ventas).
- Fecha del acto (o período, en el caso de diario/IVA).
- Número de acta (cuando corresponda).
- Hash del PDF final.
- Hash anterior declarado en el encabezado (si existe).
- Estado frente a TAD:
  - Draft (sin PDF).
  - Pendiente de registro TAD.
  - Final.
  - Error de TAD (hash no coincide).
- Archivos asociados:
  - PDF del acto.
  - Comprobante TAD (cuando se cargue).
- Metadatos extraídos (participantes, resoluciones, etc.), todos editables con trazabilidad.

**FR-14 – Ciclo de vida de un acto**

- Estados permitidos:
  1. **Draft**: solo texto editable; sin PDF ni hash.
  2. **Pendiente de registro TAD**: PDF generado con hash y encabezado; hash calculado.
  3. **Final**: comprobante TAD cargado y validado.
  4. **Error TAD**: comprobante TAD no coincide con el hash del PDF.

- Transiciones:
  - Draft → Pendiente de registro TAD (al generar PDF).
  - Pendiente de registro TAD → Final (al validar comprobante).
  - Pendiente de registro TAD → Error TAD (si hash no coincide).
  - No se debe poder volver de Final a Draft; cambios futuros requieren un nuevo acto o procedimiento específico.

---

### 5.5 Creación asistida de actos (prompt + templates)

**FR-15 – Creación de acto por texto libre**

- El usuario (Owner/RW) debe poder crear un nuevo acto eligiendo:
  - Sociedad.
  - Libro objetivo.
- El sistema mostrará un campo de texto libre:
  - El usuario describe qué quiere hacer (ej.: “aprobación de balance 31/12/2024 y distribución de utilidades”).

**FR-16 – Uso del índice de templates**

- El sistema debe contar con un **listado indexado de templates** de actos, gestionado por Superadmin, que incluya:
  - ID de template.
  - Nombre descriptivo.
  - Tipo de libro al que aplica.
  - Tipo de acto (ej. designación de apoderado, aprobación de balance, aumento de capital).
  - Tags/palabras clave.
  - Variables parametrizables (ej.: {fecha_ejercicio, monto_dividendos, nombre_apoderado, etc.}).
  - Estado (activo/archivado).
- Ante una descripción del usuario, la IA debe:
  - Consultar el índice de templates.
  - Seleccionar uno o varios templates relevantes.
  - Incorporarlos a su contexto para generar la propuesta de acto.

**FR-17 – Generación de propuestas de texto**

- A partir del input del usuario + contexto de la sociedad + template(s) elegido(s), la IA debe:
  - Proponer al menos una redacción completa de acta.
  - Completar variables usando:
    - Datos extraídos del Contexto de la sociedad.
    - Supuestos razonables (ej. fechas), que el usuario puede modificar.

**FR-18 – Iteración con IA**

- Mientras el acto esté en Draft:
  - El usuario puede solicitar cambios (“agregá una cláusula…”, “simplificá el texto…”).
  - La IA genera nuevas versiones.
- El sistema debe guardar histórico de versiones de borrador:
  - Texto anterior / texto nuevo.
  - Usuario que aprobó el cambio.

**FR-19 – Guardar borrador**

- El usuario puede guardar el borrador en cualquier momento, sin generar PDF.
- Borradores deben aparecer listados y ser retomables.

---

### 5.6 Generación de PDF, hash y flujo TAD

**FR-20 – Generación de PDF con hash anterior**

- Al “cerrar” un acto (pasar de Draft a Pendiente de registro TAD), el sistema debe:
  1. Determinar el **último acto con PDF** del mismo libro.
  2. Obtener su hash real.
  3. Incluir en el encabezado del nuevo documento un texto estándar del tipo:  
     “Hash del documento anterior de este libro: [HASH_ANTERIOR]”.
  4. Generar el PDF final basado en el texto del borrador.
  5. Calcular el hash (ej. SHA-256) del PDF.
  6. Guardar hash real y hash anterior declarado.

**FR-21 – Pantalla de hash para TAD**

- Tras la generación del PDF, el sistema debe:
  - Mostrar el hash calculado.
  - Mostrar los datos necesarios para registrar el documento en TAD según el tipo de libro (fecha de acta, número de acta, etc.).
  - Permitir:
    - Copiar el hash al portapapeles.
    - Descargar un archivo (TXT/CSV) con hash + metadatos.

**FR-22 – Carga y validación de comprobante TAD**

- El usuario debe poder subir el comprobante TAD (PDF u otro formato convertidos a PDF).
- El sistema debe:
  - Extraer el hash registrado en el comprobante.
  - Compararlo con el hash del PDF.
  - Si coinciden → marcar acto como Final.
  - Si no coinciden → marcar acto como Error TAD y mostrar alerta.

---

### 5.7 Flujo específico IVA Compras / IVA Ventas

**FR-23 – Carga simplificada de libros IVA**

- Para libros IVA Compras e IVA Ventas:
  - El usuario (ej. contador) sube un PDF ya generado por su sistema.
  - El sistema **no está obligado** a interpretar el contenido fiscal, solo a:
    - Calcular el hash.
    - Mantener la cadena de hashes.

**FR-24 – Encabezado con hash anterior para IVA**

- El sistema debe:
  - Obtener el hash del documento anterior del mismo libro.
  - Generar un PDF resultante que:
    - Contenga el contenido original.
    - Incluya en la **primera hoja** el encabezado con el hash anterior en formato estándar.
  - Permitir descargar este PDF listo para registrar en TAD.
  - Registrar el hash del nuevo documento y el hash anterior declarado.

**FR-25 – Validación TAD para IVA**

- El flujo de carga y validación del comprobante TAD para IVA debe ser idéntico al de los demás libros (FR-22).

---

### 5.8 Validaciones y controles básicos

**FR-26 – Control de secuencia temporal y numérica**

- Para cada libro de actas:
  - Verificar que los números de acta sigan una secuencia lógica (creciente).
  - Verificar que las fechas de las actas no presenten anomalías evidentes (ej. un acta 2020 entre dos de 2023).
- Actos con anomalías deben marcarse como “posible acto fuera de lugar”.

**FR-27 – Control de mandatos de autoridades**

- A partir de las actas de designación/remoción y del estatuto:
  - Reconstruir, en la medida de lo posible, períodos de mandato de autoridades.
- Marcar actas donde una persona actúa como autoridad fuera de su mandato estimado como:
  - “Posible inconsistencia de autoridades” (sin juicio de validez legal).

**FR-28 – Consistencia tipo de acto vs libro**

- El sistema debe marcar como sospechoso cuando:
  - Un tipo de acto típico de un órgano (ej. asamblea) aparece en un libro que no corresponde.
  - Exista un movimiento accionario sin acta asociada, etc.

**FR-29 – Cadena de hashes**

- Para cada libro:
  - `hash_anterior` declarado en el encabezado del documento N debe coincidir con el `hash_real` del documento N-1.
  - Deben marcarse cortes de cadena:
    - Falta de hash anterior (salvo en el primer documento).
    - Hash anterior que no corresponde a ningún documento existente.
    - Hash TAD que no coincide con el hash del PDF.

---

### 5.9 Borrado, auditoría y trazabilidad

**FR-30 – Borrado lógico de entidades**

- El borrado de:
  - Sociedades,
  - Libros,
  - Actos,
  - Documentos,
  debe ser lógico:
  - Se marca como eliminado,
  - No se elimina información ni archivos físicamente (al menos en MVP).

**FR-31 – Log de auditoría**

- El sistema debe registrar:
  - Creación, modificación y borrado lógico de:
    - Sociedades, libros, actos, contexto, templates.
  - Carga/reemplazo de PDFs y comprobantes.
  - Edición de metadatos extraídos por IA.
- Cada entrada de log debe incluir:
  - Usuario,
  - Fecha y hora,
  - Acción,
  - Entidad afectada,
  - Descripción legible del cambio.

**FR-32 – Versionado de metadatos y contexto**

- Para metadatos de actos y datos de sociedad:
  - Guardar versiones con antes/después.
- Para el Contexto:
  - Mantener un historial de versiones, con posibilidad de visualizar y, eventualmente, restaurar.

---

### 5.10 Navegación, búsqueda e índice

**FR-33 – Índice de libros y actos**

- Para cada sociedad, el sistema debe mostrar:
  - Lista de libros.
  - Dentro de cada libro, lista paginada de actos:
    - Número de acta (si aplica),
    - Fecha,
    - Tipo de acto,
    - Estado (Draft, Pendiente TAD, Final, Error TAD),
    - Resumen corto generado por IA.

**FR-34 – Resúmenes de actos**

- Para cada acto, la IA debe generar:
  - Un resumen corto (para el índice).
  - Un resumen extendido (1–3 párrafos) que incluya:
    - Quiénes participaron,
    - Temas tratados,
    - Decisiones clave.

**FR-35 – Buscador**

- Debe existir un buscador que permita:
  - Texto libre (sobre contenido de actos y contexto).
  - Filtros por:
    - Tipo de acto,
    - Persona involucrada,
    - Rango de fechas,
    - Estado (Draft/Final, etc.).

---

### 5.11 Gestión de templates por Superadmin

**FR-36 – CRUD de templates**

- El Superadmin debe poder:
  - Crear nuevos templates.
  - Editar templates existentes.
  - Marcar templates como activos/inactivos.
  - Versionar templates (manteniendo histórico de versiones).

**FR-37 – Estructura de un template**

Cada template debe contemplar al menos:

- ID.
- Nombre descriptivo.
- Descripción corta.
- Tipo de libro objetivo.
- Tipo de acto.
- Tags / palabras clave.
- Variables parametrizables (con nombre y breve descripción).
- Texto base con placeholders (ej.: `{{nombre_director}}`, `{{fecha_asamblea}}`, etc.).
- Estado (activo / archivado).

**FR-38 – Indexación y búsqueda de templates**

- El sistema debe mantener un índice que permita:
  - Buscar templates por nombre, tags, tipo de libro, tipo de acto.
  - Consultar rápidamente templates relevantes ante:
    - Un input de texto libre del usuario para crear un acto.

**FR-39 – Uso de templates en generación de actos**

- La IA debe:
  - Utilizar el índice de templates como fuente principal de estructuras de texto.
  - Rellenar variables con:
    - Datos del Contexto de la sociedad (si existen).
    - Preguntas al usuario si algún dato clave está ausente o ambiguo.
  - Permitir que el usuario vea y entienda qué template se usó como base (para transparencia).

---

## 6. Requerimientos no funcionales (alto nivel)

**NFR-01 – Mínimo input manual**

- El diseño del sistema debe priorizar que:
  - El owner, para iniciar, solo tenga que:
    - Crear la sociedad con un nombre.
    - Subir sus PDFs.
  - Toda la extracción de datos y armado de libros/actos/contexto sea automática, sujeta a revisión pero no obligatoria.

**NFR-02 – Seguridad y confidencialidad**

- Acceso autenticado y con controles de autorización por rol y sociedad.
- Los datos y documentos de una sociedad no deben ser accesibles por usuarios no autorizados.

**NFR-03 – Trazabilidad**

- Todas las operaciones relevantes (creación, cambios, borrados lógicos) deben quedar registradas en el log de auditoría.

**NFR-04 – Extensibilidad**

- El modelo de datos y la estructura de templates deben permitir agregar:
  - Nuevos tipos de actos,
  - Nuevos tipos de libros,
  sin romper el diseño actual.

---

## 7. Requerimientos adicionales (identificados en revisión)

### 7.1 Manejo de errores y recuperación

**FR-40 – Manejo de errores de IA**

- El sistema debe contemplar escenarios donde la IA no pueda clasificar o extraer datos:
  - Si la confianza de clasificación es baja (< umbral configurable), marcar documento como "Requiere revisión manual".
  - Permitir al usuario reclasificar manualmente documentos.
  - Registrar feedback del usuario para mejora continua del modelo.
- Debe mostrarse claramente qué datos fueron extraídos automáticamente vs ingresados manualmente.

**FR-41 – Recuperación de Error TAD**

- Cuando un acto está en estado "Error TAD":
  - Mostrar claramente la discrepancia (hash esperado vs hash del comprobante).
  - Permitir al usuario:
    - Subir un nuevo comprobante TAD.
    - Regenerar el PDF (lo cual invalida el hash anterior y requiere nuevo registro).
  - Si se regenera el PDF, el sistema debe:
    - Crear una nueva versión del acto.
    - Actualizar la cadena de hashes de documentos posteriores (si existen).
    - Registrar el motivo de la regeneración.

**FR-42 – Manejo de PDFs ilegibles o con OCR fallido**

- Si el sistema no puede procesar un PDF (calidad muy baja, encriptado, corrupto):
  - Notificar al usuario con descripción del problema.
  - Marcar el documento como "Procesamiento fallido".
  - Permitir:
    - Reintentar procesamiento.
    - Cargar versión alternativa del archivo.
    - Ingresar datos manualmente sin extracción automática.

---

### 7.2 Dashboard y notificaciones

**FR-43 – Dashboard de estado de la sociedad**

- Para cada sociedad, mostrar un dashboard que incluya:
  - Cantidad de actos por estado (Draft, Pendiente TAD, Final, Error).
  - Alertas activas (errores de cadena de hash, inconsistencias detectadas).
  - Próximos vencimientos (mandatos de autoridades que están por vencer).
  - Últimos actos creados/modificados.
  - Progreso del onboarding (si hay documentos en procesamiento).

**FR-44 – Sistema de notificaciones**

- El sistema debe enviar notificaciones (email y/o in-app) para:
  - Actos pendientes de registro TAD por más de X días (configurable).
  - Vencimiento próximo de mandatos de autoridades (30, 15, 7 días antes).
  - Errores detectados (Error TAD, corte de cadena de hash).
  - Invitaciones a sociedad (nuevo usuario invitado).
  - Finalización de procesamiento de carga masiva.
- El usuario debe poder configurar qué notificaciones recibir.

**FR-45 – Proceso de onboarding detallado con feedback**

- Durante la carga masiva de PDFs:
  - Mostrar cola de procesamiento con estado de cada archivo.
  - Estados posibles: En cola, Procesando, Clasificado, Error.
  - Barra de progreso general.
  - Tiempo estimado de finalización.
  - Permitir cancelar procesamiento de archivos en cola.
- Al finalizar:
  - Mostrar resumen: X documentos procesados, Y clasificados automáticamente, Z requieren revisión.
  - Link directo a documentos que requieren atención.

---

### 7.3 Exportación y respaldo

**FR-46 – Exportación de datos de la sociedad**

- Permitir exportar:
  - Libro completo: PDF compilado con todos los actos en orden cronológico.
  - Backup completo: ZIP con todos los PDFs originales + metadatos en JSON/CSV.
  - Contexto de la sociedad: PDF o Markdown.
  - Índice de actos: CSV con metadatos de todos los actos.
- Solo Owner y SuperAdmin pueden exportar backup completo.

---

### 7.4 Control y seguridad adicional

**FR-47 – Límites y cuotas**

- El sistema debe implementar límites configurables:
  - Máximo de sociedades por cuenta de usuario (para planes futuros).
  - Máximo de archivos por carga masiva: 100 archivos.
  - Tamaño máximo por archivo: 50 MB.
  - Llamadas a IA por sociedad/mes (para control de costos).
- Mostrar uso actual vs límites al usuario.

**FR-48 – Manejo de conflictos de edición concurrente**

- Si dos usuarios intentan editar el mismo borrador simultáneamente:
  - Usar bloqueo optimista basado en versión.
  - Al guardar, verificar que la versión no haya cambiado.
  - Si hay conflicto, mostrar ambas versiones y permitir al usuario resolver.
  - Registrar conflictos en auditoría.

**FR-49 – Permisos granulares por libro (futuro)**

- Evaluar para versiones futuras la posibilidad de permisos específicos por libro.
- Para MVP: los permisos se aplican a nivel de sociedad completa.
- Si se implementa, sería adicional al permiso general de la sociedad.

---

### 7.5 Flujos específicos del dominio

**FR-50 – Regeneración de PDF antes de registro TAD**

- Mientras un acto esté en estado "Pendiente de registro TAD" y no se haya subido comprobante:
  - Permitir volver a estado Draft para hacer correcciones.
  - Al regenerar PDF:
    - Recalcular hash.
    - Mantener el mismo hash_anterior (no cambia la posición en la cadena).
    - Registrar en auditoría el motivo de la regeneración.

**FR-51 – Primer documento de un libro (inicio de cadena)**

- Para el primer documento de cada libro:
  - El campo hash_anterior debe contener un valor especial: "PRIMER_DOCUMENTO_DEL_LIBRO".
  - El encabezado del PDF debe indicar: "Este es el primer documento de este libro digital. No existe hash anterior."
  - Esto aplica tanto para libros nuevos como para libros que inician su digitalización.

**FR-52 – Organización por ejercicio contable**

- Para libros que se organizan por ejercicio (Diario, Inventarios y Balances, IVA):
  - Permitir crear "ejercicios" con fecha de inicio y cierre.
  - Los actos se asocian a un ejercicio.
  - La numeración puede reiniciar por ejercicio o ser continua (configurable).
  - La cadena de hashes es continua independientemente del ejercicio.

**FR-53 – Transferencia de ownership de sociedad**

- El Owner actual debe poder transferir ownership a otro usuario:
  - El nuevo Owner debe tener cuenta activa en el sistema.
  - Requiere confirmación de ambas partes (email de confirmación).
  - El Owner anterior pasa a ser RW automáticamente.
  - Registrar transferencia en auditoría con detalle completo.

**FR-54 – Actos que afectan múltiples áreas del contexto**

- Cuando un acto afecta múltiples aspectos (ej: asamblea que aprueba balance Y cambia autoridades):
  - La IA debe identificar todos los cambios relevantes.
  - Proponer actualizaciones separadas al Contexto para cada aspecto.
  - El usuario puede aceptar/rechazar cada propuesta individualmente.
  - Mantener relación entre el acto y todas las actualizaciones de contexto que generó.

---

## 8. Requerimientos no funcionales adicionales

**NFR-05 – Estrategia de integración con IA**

- El sistema debe definir:
  - Proveedor de IA a utilizar (OpenAI GPT-4, Claude, etc.).
  - Estrategia de manejo de prompts (templates centralizados).
  - Manejo de errores y timeouts del servicio de IA.
  - Rate limiting y control de costos.
  - Fallback a procesamiento manual cuando IA no está disponible.

**NFR-06 – Almacenamiento de archivos**

- Los archivos PDF deben almacenarse de forma segura:
  - Ubicación: Volumen persistente separado o mismo volumen que BD.
  - Estructura organizada por sociedad/libro/acto.
  - Backup incluido en estrategia general de respaldo.
  - Retención de archivos de entidades borradas lógicamente.
  - Encriptación at-rest recomendada para datos sensibles.

**NFR-07 – Performance de procesamiento masivo**

- El procesamiento de carga masiva de PDFs debe:
  - Ejecutarse de forma asíncrona (no bloquear UI).
  - Usar cola de procesamiento con workers.
  - Proveer feedback de progreso en tiempo real.
  - Permitir procesamiento en paralelo de múltiples archivos.
  - Completar carga de 50 PDFs en menos de 10 minutos (objetivo).

**NFR-08 – Accesibilidad**

- La interfaz debe cumplir con estándares de accesibilidad:
  - WCAG 2.1 nivel AA como mínimo.
  - Navegación por teclado completa.
  - Compatibilidad con lectores de pantalla.
  - Contraste adecuado y textos legibles.

**NFR-09 – Internacionalización**

- Para MVP: Solo español (Argentina).
- El sistema debe estar preparado para internacionalización futura:
  - Textos externalizados en archivos de traducción.
  - Formatos de fecha/número configurables por locale.
  - Posible expansión a otros países con estructuras societarias similares.

---

_Fin de REQUERIMIENTO.MD_
