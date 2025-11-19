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

**Estado:** 🟡 PENDIENTE DECISIÓN

**Opciones evaluadas:**

| Opción | Pros | Contras |
|--------|------|---------|
| **BullMQ + Redis** | Maduro, dashboard UI, reintentos, prioridades | Requiere Redis (más infra/costo) |
| **Custom con SQLite** | Sin dependencias extra, simple para MVP | Hay que implementar, sin dashboard |

**Recomendación:** Para MVP usar custom con SQLite (tabla `job_queue`). Migrar a BullMQ cuando escale.

**Decisión requerida:** ¿Custom SQLite para MVP o BullMQ desde el inicio?

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

**Impacto:** Medio-Alto
**Requerimientos afectados:** FR-20, FR-24

**Descripción:**
Se debe generar PDFs con encabezado que incluya hash anterior, pero no se definió:
- Librería de generación de PDF (puppeteer, pdf-lib, jsPDF)
- Template del encabezado
- Cómo insertar encabezado en PDFs existentes (IVA)

**Decisiones requeridas:**
1. ¿Qué librería usar para generar PDFs?
2. ¿Cómo insertar página de encabezado en PDF existente?
3. ¿Formato exacto del encabezado?

**Recomendación:**
Usar `pdf-lib` para manipulación de PDFs existentes y React-PDF o Puppeteer para generar nuevos.

---

### GAP-005: Modelo de Contexto y versionado

**Estado:** 🟡 PENDIENTE DECISIÓN

**Opciones evaluadas:**

| Opción | Pros | Contras |
|--------|------|---------|
| **Markdown** | Familiar, fácil renderizar, IA lo genera bien, versionable | Menos control estructura |
| **JSON estructurado** | Validación schema, consultas granulares | Necesita UI custom, menos flexible |

**Recomendación:** Markdown - Balance entre flexibilidad y estructura. Usar `diff` o `jsdiff` para comparar versiones.

**Decisión requerida:** ¿Markdown o JSON estructurado?

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

**Estado:** 🟡 PENDIENTE DECISIÓN

**Opciones evaluadas:**

| Opción | Pros | Contras |
|--------|------|---------|
| **Handlebars** | Condicionales `{{#if}}`, loops `{{#each}}`, helpers custom | Más complejo, ~50KB |
| **Mustache** | Muy simple, pequeño (~10KB), logic-less | Sin condicionales complejos |
| **Custom** | Sin dependencias, control total | Hay que implementar todo |

**Ejemplo de uso:**
```handlebars
En {{ciudad}}, a los {{dia}} días de {{mes}} de {{año}}...
{{#if multiples_socios}}
Los socios presentes son:
{{#each socios}}
- {{nombre}} (DNI {{dni}})
{{/each}}
{{else}}
Con la presencia del único socio {{socio_unico.nombre}}...
{{/if}}
```

**Recomendación:** Handlebars - Los templates de actas necesitan condicionales y loops. Guardar referencia a versión de template en cada acto.

**Decisión requerida:** ¿Handlebars, Mustache, o custom?

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
| GAP-003 | Procesamiento asíncrono | Alto | Alto | **P1** | 🟡 Pendiente decisión |
| GAP-011 | OCR/Vision para PDFs | Medio-Alto | Medio | **P1** | ✅ Resuelto |
| GAP-004 | Generación de PDFs | Medio-Alto | Medio | **P1** | Pendiente |
| GAP-005 | Modelo de Contexto | Medio | Medio | **P2** | 🟡 Pendiente decisión |
| GAP-006 | Extracción hash TAD | Medio | Bajo | **P2** | ✅ Resuelto |
| GAP-007 | Sistema de templates | Medio | Medio | **P2** | 🟡 Pendiente decisión |
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

2. **Decisiones pendientes (elegir una opción):**
   - [ ] GAP-003: ¿Custom SQLite o BullMQ+Redis para colas?
   - [ ] GAP-005: ¿Markdown o JSON para Contexto?
   - [ ] GAP-007: ¿Handlebars, Mustache, o custom para templates?
   - [ ] GAP-004: Definir librería de generación de PDFs

3. **Antes de iniciar desarrollo:**
   - [ ] Crear ADR (Architecture Decision Record) consolidando decisiones
   - [ ] Diseñar schema de base de datos completo
   - [ ] Definir estructura de carpetas del proyecto

4. **Investigación paralela:**
   - [ ] Revisar templates de actos existentes (para crear templates iniciales)
   - [ ] Testear calidad de Vision API con PDFs escaneados reales

---

_Documento generado automáticamente. Revisar y actualizar según avance del proyecto._
