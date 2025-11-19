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

**Impacto:** Alto
**Requerimientos afectados:** FR-05, FR-20, FR-22, FR-23, FR-24

**Descripción:**
Se cargarán potencialmente cientos de PDFs por sociedad, pero no se definió:
- Dónde almacenar (filesystem local, S3, otro)
- Estructura de directorios
- Límites de almacenamiento
- Backup de archivos
- Retención de archivos borrados

**Decisiones requeridas:**
1. ¿Usar S3/MinIO o volumen persistente local?
2. ¿Cuánto almacenamiento asignar por sociedad?
3. ¿Cómo manejar archivos de sociedades borradas?

**Recomendación:**
Para MVP con SQLite, usar volumen persistente local (`/data/files/`). Migrar a S3 cuando escale.

---

### GAP-003: Procesamiento asíncrono de PDFs

**Impacto:** Alto
**Requerimientos afectados:** FR-05, FR-06, FR-07, FR-08

**Descripción:**
La carga masiva de PDFs requiere procesamiento asíncrono (clasificación, extracción de IA), pero no hay:
- Sistema de colas definido
- Workers de procesamiento
- Estrategia de reintentos
- Notificación de progreso al usuario

**Decisiones requeridas:**
1. ¿Usar Bull/BullMQ, Agenda, o implementación custom?
2. ¿Cuántos workers paralelos?
3. ¿Websockets o polling para progreso?

**Recomendación:**
Usar BullMQ con Redis para colas. Websockets para progreso en tiempo real.

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

**Impacto:** Medio
**Requerimientos afectados:** FR-10, FR-11, FR-32

**Descripción:**
El Contexto es un documento vivo con versionado, pero no se definió:
- Formato del contenido (Markdown, HTML, JSON estructurado)
- Cómo calcular y mostrar diffs entre versiones
- Cómo la IA propone cambios (patch, texto completo)

**Decisiones requeridas:**
1. ¿Formato del Contexto?
2. ¿Cómo mostrar comparación de versiones?
3. ¿Cómo estructurar propuestas de cambio de IA?

**Recomendación:**
Usar Markdown para el Contexto. Guardar versiones completas. Mostrar diff con librería como `diff`.

---

### GAP-006: Extracción de hash del comprobante TAD

**Impacto:** Medio
**Requerimientos afectados:** FR-22

**Descripción:**
Se debe extraer el hash del comprobante TAD para validarlo, pero:
- No se conoce el formato exacto del comprobante TAD
- No se sabe si es PDF, imagen, u otro
- No se sabe dónde aparece el hash en el comprobante

**Decisiones requeridas:**
1. ¿Formato del comprobante TAD?
2. ¿Usar OCR o el hash está en texto seleccionable?
3. ¿Qué hacer si no se puede extraer automáticamente?

**Recomendación:**
Investigar formato real de comprobantes TAD. Permitir ingreso manual del hash como fallback.

---

### GAP-007: Sistema de templates estructurado

**Impacto:** Medio
**Requerimientos afectados:** FR-36, FR-37, FR-38, FR-39

**Descripción:**
Los templates son centrales para generación de actos, pero falta definir:
- Sintaxis de variables (`{{var}}`, `${var}`, otro)
- Cómo validar que todas las variables están completas
- Cómo versionar templates sin romper actos existentes
- Herencia o composición de templates

**Decisiones requeridas:**
1. ¿Qué motor de templates usar? (Handlebars, Mustache, custom)
2. ¿Cómo manejar variables opcionales vs requeridas?
3. ¿Qué pasa con actos que usaron versión anterior del template?

**Recomendación:**
Usar Handlebars por su soporte de condicionales y loops. Mantener referencia a versión de template usada en cada acto.

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

## 5. Preguntas Abiertas para el Negocio

### Q-001: Integración futura con TAD
¿Hay planes de integrar directamente con la API de TAD en el futuro? Esto afecta el diseño de la arquitectura.

### Q-002: Multi-tenancy
¿Cada sociedad es completamente aislada o hay escenarios de grupos empresarios que quieran ver múltiples sociedades consolidadas?

### Q-003: Soporte offline
¿Los usuarios necesitan trabajar sin conexión? Esto afecta significativamente la arquitectura.

### Q-004: Firma digital
El MVP excluye firma digital, pero ¿está planificada para futuro? ¿Qué tipo de firma (simple, avanzada, calificada)?

### Q-005: Retención legal
¿Hay requisitos legales sobre cuánto tiempo retener los documentos? ¿Se pueden eliminar después de X años?

### Q-006: Auditorías externas
¿Los auditores externos necesitarán acceso al sistema? ¿Con qué rol?

---

## 6. Matriz de Priorización

| Gap ID | Descripción | Impacto | Esfuerzo | Prioridad | Estado |
|--------|-------------|---------|----------|-----------|--------|
| GAP-001 | Proveedor de IA | Alto | Bajo | **P0** | ✅ Resuelto |
| GAP-002 | Almacenamiento PDFs | Alto | Medio | **P0** | Pendiente |
| GAP-003 | Procesamiento asíncrono | Alto | Alto | **P1** | Pendiente |
| GAP-011 | OCR/Vision para PDFs | Medio-Alto | Medio | **P1** | ✅ Resuelto |
| GAP-004 | Generación de PDFs | Medio-Alto | Medio | **P1** | Pendiente |
| GAP-005 | Modelo de Contexto | Medio | Medio | **P2** | Pendiente |
| GAP-006 | Extracción hash TAD | Medio | Bajo | **P2** | Pendiente |
| GAP-007 | Sistema de templates | Medio | Medio | **P2** | Pendiente |
| GAP-008 | Invitación usuarios | Bajo | Bajo | **P3** | Pendiente |
| GAP-009 | Formato exportación | Bajo | Bajo | **P3** | Pendiente |
| GAP-010 | Detalle auditoría | Bajo | Bajo | **P3** | Pendiente |

---

## 7. Próximos Pasos Recomendados

1. **Inmediato (esta semana):**
   - [x] Decidir proveedor de IA (GAP-001) → **OpenRouter**
   - [x] Definir estrategia de procesamiento PDFs (GAP-011) → **Vision API fallback**
   - [ ] Decidir estrategia de almacenamiento (GAP-002)
   - [ ] Responder preguntas Q-001 a Q-006

2. **Antes de iniciar desarrollo:**
   - [ ] Crear ADR para decisiones técnicas clave
   - [ ] Definir librería de generación de PDFs (GAP-004)
   - [ ] Definir sistema de colas (GAP-003)

3. **Durante diseño de BD:**
   - [ ] Definir formato de Contexto (GAP-005)
   - [ ] Definir sintaxis de templates (GAP-007)

4. **Investigación paralela:**
   - [ ] Obtener ejemplos reales de comprobantes TAD (GAP-006)
   - [ ] Revisar templates de actos existentes
   - [ ] Testear calidad de OCR con diferentes tipos de PDFs societarios

---

_Documento generado automáticamente. Revisar y actualizar según avance del proyecto._
