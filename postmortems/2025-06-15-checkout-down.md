# Post-Mortem: Caída del checkout en producción

**Fecha:** 15/06/2025
**Severidad:** P1 (crítico)
**Duración del incidente:** 52 minutos
**Estado:** Resuelto

## Descripción objetiva del incidente

El deploy de la versión 2.4.1 incluyó una migración que renombró la
columna `order_status` a `status`. El código de la versión 2.4.0,
todavía corriendo en algunos servidores durante el despliegue gradual,
seguía consultando `order_status`, causando errores 500 en el checkout
durante 52 minutos.

## Análisis de causas (5 porqués)

| Pregunta | Respuesta |
|----------|-----------|
| ¿Por qué falló el checkout? | El código buscaba una columna renombrada |
| ¿Por qué buscaba una columna inexistente? | El renombrado no fue retrocompatible |
| ¿Por qué no fue retrocompatible? | No había política de migraciones en dos fases |
| ¿Por qué no había política? | Nunca habíamos tenido un incidente así |
| ¿Por qué nunca lo previmos? | No existía checklist de revisión de migraciones |

**Causa raíz:** ausencia de un proceso estandarizado para migraciones de
base de datos no retrocompatibles.

## Acciones correctivas y preventivas

- [ ] Implementar política de migraciones en dos fases (expand/contract).
  Responsable: equipo backend · Plazo: 1 semana
- [ ] Agregar checklist obligatorio en el template de PR para migraciones.
  Responsable: tech lead · Plazo: 3 días
- [ ] Configurar alerta temprana de error rate (primer minuto).
  Responsable: DevOps · Plazo: 1 semana

## Lecciones aprendidas

El error no fue de una persona, fue de un proceso de review que no incluía
verificar la retrocompatibilidad de las migraciones. La cultura blameless
permitió que el equipo se enfocara en mejorar el sistema en lugar de buscar
culpables.
