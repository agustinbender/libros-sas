# Análisis de Gaps - Libros SAS

**Fecha de análisis:** 2025-11-19
**Documentos revisados:** REQUERIMIENTOS.MD, CLAUDE.MD

---

## Resumen Ejecutivo

Se identificaron gaps entre los requerimientos funcionales y la documentación técnica. Este documento detalla cada gap para su revisión y priorización.

---

## 1. Gaps Críticos (Bloqueantes para MVP)

### GAP-001: Proveedor de IA no definido

**Estado:** ✅ RESUELTO

**Decisión tomada:**
- **Proveedor:** OpenRouter (Vercel AI SDK)
- **Ventajas:**
  - Flexibilidad para cambiar entre modelos (GPT-4, Claude, Llama)
  - Fallback automático si un modelo está caído
  - Fácil comparación de costos
- **Modelos por caso de uso:** A definir en variables de entorno
- **Fallback:** Procesamiento manual cuando IA no está disponible

**Documentado en:** CLAUDE.MD - Sección "AI Integration Strategy"

---

### GAP-002: Estrategia de almacenamiento de PDFs

**Estado:** ✅ RESUELTO

**Decisión tomada:**
- **Almacenamiento:** Volumen local persistente `/data/files/`
- **Estructura:** `/data/files/{sociedad_id}/{libro_id}/{acto_id}/`
- **Migración futura:** S3/MinIO cuando se necesite escalar
- **Límites:** 50MB por archivo, 100 archivos por batch

**Documentado en:** CLAUDE.MD - Sección "File Storage Strategy"

---

### GAP-003: Procesamiento asíncrono de PDFs

**Estado:** ✅ RESUELTO

**Decisión tomada:** Custom con SQLite para MVP

**Implementación:**
- Tabla `job_queue` con campos: id, type, payload, status, attempts, error, created_at, updated_at
- Estados: pending, processing, completed, failed
- Worker simple con setInterval o cron job
- Reintentos con backoff exponencial (3 intentos máx)
- Migrar a BullMQ + Redis cuando se necesite escalar

**Documentado en:** CLAUDE.MD - Sección "Async Processing Strategy"

---

## 2. Gaps Importantes (Impactan funcionalidad core)

### GAP-011: Procesamiento de PDFs escaneados (OCR/Vision)

**Impacto:** Medio-Alto
**Requerimientos afectados:** FR-06, FR-08, FR-42

**Descripción:**
Muchos PDFs societarios son escaneados (sin texto seleccionable) y requieren OCR o Vision API para extraer contenido.

**Decisión tomada:**
- **Pipeline definido:**
  1. Detectar si PDF tiene texto seleccionable
  2. Si tiene texto → usar pdf-parse
  3. Si no tiene/mala calidad → convertir a imágenes → Vision API
- **Vision APIs** (vía OpenRouter):
  - GPT-4 Vision para documentos complejos
  - Claude 3 Vision como alternativa
  - Gemini Pro Vision para optimizar costos
- **Fallback:** Tesseract.js para OCR offline como opción económica

**Documentado en:** CLAUDE.MD - Sección "PDF Processing Strategy"

**Pendiente:** Definir umbrales de calidad de texto para decidir cuándo usar Vision.

---

### GAP-004: Generación de PDFs con encabezado de hash

**Estado:** ✅ RESUELTO

**Decisión tomada:** pdf-lib para manipulación, Puppeteer/React-PDF para generación

**Implementación:**
- **pdf-lib**: Para manipular PDFs existentes (insertar página de encabezado)
- **Puppeteer o React-PDF (@react-pdf/renderer)**: Para generar nuevos PDFs desde HTML/React
- **Encabezado hash**: Primera página insertada con:
  - Hash anterior declarado
  - Fecha del acto
  - Número de acto
  - QR code opcional con link a verificación

**Documentado en:** CLAUDE.MD - Sección "PDF Generation Strategy"

---

### GAP-005: Modelo de Contexto y versionado

**Estado:** ✅ RESUELTO

**Decisión tomada:** Markdown

**Implementación:**
- Contexto almacenado como texto Markdown en la BD
- Renderizado con `react-markdown` en frontend
- Comparación de versiones con `jsdiff` (diff semántico)
- UI con split view: versión anterior | versión actual
- Editor: textarea simple o editor Markdown básico

**Documentado en:** CLAUDE.MD - Sección "Context Document Strategy"

---

### GAP-006: Extracción de hash del comprobante TAD

**Estado:** ✅ RESUELTO

**Análisis del comprobante real (IF-2023-05069991-APN-DA#IGJ):**

El comprobante TAD es un PDF generado por la IGJ con:
- **Formato:** PDF con texto seleccionable (NO requiere OCR)
- **Hash:** En línea `Hash: a53ab3dade11bd9036938f1ba67158210e723f9027759ee639e2d853fb022545`
- **Metadatos incluidos:**
  - Tipo de libro
  - Tipo de acta
  - Fecha del acta
  - N° acta
- **Firma digital:** "Gestion Documental Electronica" con timestamp

**Implementación:**
- Usar `pdf-parse` para extraer texto
- Regex: `/Hash:\s*([a-f0-9]{64})/i` para obtener el hash
- Fallback: Input manual si la extracción falla
- Validar que los metadatos coincidan con el acto

**Documentado en:** Ejemplo en `/Libro de actas - Socios 2022-11-24 - 5 - Hash*.pdf`

---

### GAP-007: Sistema de templates estructurado

**Estado:** ✅ RESUELTO

**Decisión tomada:** Markdown con placeholders simples, AI hace las sustituciones

**Implementación:**
- Templates en formato Markdown con placeholders tipo `{{variable}}`
- **Sin motor de templates** - la AI interpreta y adapta el contenido
- La AI es responsable de:
  - Reemplazar variables con valores del contexto
  - Adaptar singular/plural según cantidad de socios
  - Agregar/quitar secciones según el caso
  - Ajustar redacción al contexto específico
- Los templates son "guías/ejemplos" no templates mecánicos
- Más flexible para casos edge y variaciones

**Ejemplo de template:**
```markdown
En {{ciudad}}, a los {{dia}} días de {{mes}} de {{año}}, siendo las {{hora}} horas,
se constituye en el domicilio de la sociedad la reunión de socios de {{nombre_sociedad}}
con la presencia del Administrador titular y único accionista, tenedor de la totalidad
del Capital Social, Sr. {{nombre_socio1}}.

El Sr. {{apellido_socio1}} resuelve:
1. Que el acta sea firmada por él.
2. {{resolucion_principal}}
```

**Nota:** La AI adaptará automáticamente si hay múltiples socios, cambiando "único accionista" por la lista correspondiente.

**Documentado en:** CLAUDE.MD - Sección "Template Strategy"

---

## 3. Gaps Menores (Mejoras de UX/DX)

### GAP-008: Flujo de invitación de usuarios

**Impacto:** Bajo
**Requerimientos afectados:** FR-02

**Descripción:**
El Owner puede invitar usuarios, pero no se definió:
- Email de invitación (contenido, branding)
- Flujo si el usuario ya tiene cuenta vs no tiene
- Expiración de invitación

**Recomendación:**
Definir en fase de diseño de UI. No bloqueante para backend.

---

### GAP-009: Formato de exportación

**Impacto:** Bajo
**Requerimientos afectados:** FR-46

**Descripción:**
Se puede exportar datos, pero no se definió formato exacto de:
- Metadatos en JSON/CSV
- Estructura del ZIP de backup
- PDF compilado (portada, índice, separadores)

**Recomendación:**
Definir durante implementación. Empezar simple y mejorar.

---

### GAP-010: Nivel de detalle de auditoría

**Impacto:** Bajo
**Requerimientos afectados:** FR-31

**Descripción:**
Se debe registrar auditoría, pero no se definió:
- Qué campos exactos guardar para cada acción
- Formato de "descripción legible del cambio"
- Retención de logs

**Recomendación:**
Definir estructura base e ir expandiendo según necesidad.

---

## 4. Inconsistencias Detectadas

### INC-001: Nomenclatura de roles

**Documentos afectados:** REQUERIMIENTOS.MD, CLAUDE.MD

**Inconsistencia:**
- REQUERIMIENTOS.MD: Superadmin, Owner, RW, RO, Contador
- CLAUDE.MD (anterior): SuperAdmin, Institution Admin, Regular User

**Estado:** ✅ RESUELTO - CLAUDE.MD actualizado

---

### INC-002: Modelo "Books" vs "Libro"

**Documentos afectados:** CLAUDE.MD

**Inconsistencia:**
CLAUDE.MD original usaba "Books" (inglés), pero REQUERIMIENTOS.MD usa "Libro" (español).

**Estado:** ✅ RESUELTO - CLAUDE.MD actualizado a "Libro"

---

### INC-003: Sistema de notificaciones mencionado pero no requerido

**Documentos afectados:** REQUERIMIENTOS.MD (original), CLAUDE.MD

**Inconsistencia:**
CLAUDE.MD mencionaba "Notification System" pero REQUERIMIENTOS.MD original no tenía requerimientos de notificaciones.

**Estado:** ✅ RESUELTO - FR-44 agregado a REQUERIMIENTOS.MD

---

## 5. Preguntas Abiertas - RESPONDIDAS

### Q-001: Integración futura con TAD
**Respuesta:** No se sabe si TAD tiene API. Por ahora, flujo manual (copiar hash, subir comprobante).

**Impacto:** No afecta diseño actual. Si existe API en el futuro, se puede agregar como feature.

---

### Q-002: Multi-tenancy
**Respuesta:** Sí, un usuario puede ver más de una sociedad.

**Impacto:** Ya contemplado en el diseño con `UsuarioXSociedad`. El JWT incluye lista de sociedades con permisos.

---

### Q-003: Soporte offline
**Respuesta:** Sin soporte offline.

**Impacto:** Simplifica arquitectura. No se necesita service worker ni sincronización.

---

### Q-004: Firma digital
**Respuesta:** Si se necesita firma digital, se gestiona afuera del sistema.

**Impacto:** El sistema solo maneja PDFs ya firmados. No se necesita integración con proveedores de firma.

---

### Q-005: Retención legal
**Respuesta:**
- Documentos finales solo se borran lógicamente
- Plazo legal: 10 años
- Es responsabilidad del usuario mantener la suscripción
- Si deja de pagar, se pueden eliminar físicamente

**Impacto:**
- Implementar soft delete para todo
- Agregar política de retención configurable
- Job de limpieza para cuentas inactivas (futuro)

---

### Q-006: Auditorías externas
**Respuesta:** Sí, sería bueno tener log de eventos y versiones anteriores.

**Impacto:** Ya contemplado con:
- AuditLog para eventos
- Versionado de Contexto y metadatos
- Los auditores pueden ser usuarios RO de la sociedad

---

## 6. Matriz de Priorización

| Gap ID | Descripción | Impacto | Esfuerzo | Prioridad | Estado |
|--------|-------------|---------|----------|-----------|--------|
| GAP-001 | Proveedor de IA | Alto | Bajo | **P0** | ✅ Resuelto |
| GAP-002 | Almacenamiento PDFs | Alto | Medio | **P0** | ✅ Resuelto |
| GAP-003 | Procesamiento asíncrono | Alto | Alto | **P1** | ✅ Resuelto |
| GAP-011 | OCR/Vision para PDFs | Medio-Alto | Medio | **P1** | ✅ Resuelto |
| GAP-004 | Generación de PDFs | Medio-Alto | Medio | **P1** | ✅ Resuelto |
| GAP-005 | Modelo de Contexto | Medio | Medio | **P2** | ✅ Resuelto |
| GAP-006 | Extracción hash TAD | Medio | Bajo | **P2** | ✅ Resuelto |
| GAP-007 | Sistema de templates | Medio | Medio | **P2** | ✅ Resuelto |
| GAP-008 | Invitación usuarios | Bajo | Bajo | **P3** | Pendiente |
| GAP-009 | Formato exportación | Bajo | Bajo | **P3** | Pendiente |
| GAP-010 | Detalle auditoría | Bajo | Bajo | **P3** | Pendiente |

---

## 7. Próximos Pasos Recomendados

1. **Completado:**
   - [x] Decidir proveedor de IA (GAP-001) → **OpenRouter**
   - [x] Definir estrategia de procesamiento PDFs (GAP-011) → **Vision API fallback**
   - [x] Decidir estrategia de almacenamiento (GAP-002) → **Volumen local, migrar a S3**
   - [x] Responder preguntas Q-001 a Q-006 → **Todas respondidas**
   - [x] Obtener ejemplos reales de comprobantes TAD (GAP-006) → **Formato analizado**
   - [x] GAP-003: Procesamiento asíncrono → **Custom SQLite para MVP**
   - [x] GAP-005: Formato del Contexto → **Markdown**
   - [x] GAP-007: Sistema de templates → **Markdown + AI (sin motor)**
   - [x] GAP-004: Generación de PDFs → **pdf-lib**

2. **Antes de iniciar desarrollo:**
   - [ ] Diseñar schema de base de datos completo (Prisma)
   - [ ] Definir estructura de carpetas del proyecto
   - [ ] Configurar OpenRouter y probar conexión

3. **Investigación paralela:**
   - [ ] Revisar templates de actos existentes (para crear templates iniciales)
   - [ ] Testear calidad de Vision API con PDFs escaneados reales
   - [ ] Definir prompts iniciales para cada caso de uso de IA

---

_Documento generado automáticamente. Revisar y actualizar según avance del proyecto._
