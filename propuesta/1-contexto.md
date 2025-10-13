## **1.1 Contexto del Negocio**

BuildPeer es una plataforma de gestión de proyectos de construcción que planea una expansión que demanda evolución técnica:

**Métricas Actuales vs Proyección Q4 2026:**

- Clientes: 150 → 500+ +233%
- Usuarios activos: 10,000 → 50,000+ +400%
- Volumen diario: 1,000 uploads → 15,000+ uploads
- Expansión geográfica: México → 5 países LATAM

**Requisitos Críticos del Sector:**
- Archivos CAD/BIM de hasta **2GB** (planos de construcción)
- Trazabilidad completa para auditorías de obra

## **1.2 Problema Actual (Asunciones)**

El sistema actual enfrenta limitaciones críticas que impactan directamente la operación y crecimiento del negocio:

- **Escalabilidad limitada**: Servidor único EC2 soporta máximo **500 uploads concurrentes**, mientras la demanda proyectada es **2,000-3,000 concurrentes** para Q3 2026.
- **Fallos en carga masiva**: **25% de fallas** en uploads de archivos >500MB, requiriendo hasta 3 reintentos manuales por archivo.
- **Degradación del rendimiento**: Tiempo de respuesta promedio de **20-30 segundos** para uploads múltiples, con picos de **65 segundos** durante horas pico.
- **Impacto financiero**: Pérdida estimada de **$45K USD/mes** por retrasos en aprobaciones de documentos y **12% de churn** atribuible a problemas de performance.

## **1.3 Solución Propuesta**

Migración estratégica hacia una arquitectura distribuida y escalable que resolverá las limitaciones actuales:

**Beneficios:**

- **Performance 10x**: Reducción de latencia a **<3 segundos** para 95% de operaciones mediante upload directo a S3 y procesamiento paralelo.
- **Disponibilidad 99.9%**: De 87% actual a 99.9% mediante arquitectura resiliente con auto-scaling y multi-AZ deployment.
- **Capacidad ilimitada**: Escalamiento horizontal automático soportando **50,000+ uploads/día** sin degradación
- **ROI en 6 meses**: Reducción de 70% en costos operacionales por eficiencia y prevención de **$540K USD/año** en pérdidas por churn

**Timeline de Implementación:** 10 semanas con rollout incremental minimizando riesgo

## **1.4 Approach Técnico**

Estrategia de migración progresiva que balancea rendimiento con estabilidad operacional:

**Evolución Arquitectónica de Monolito a Monolito Modular por capas:**
```
Fase 1 (Semanas 1-3):  Monolito + Upload Service (Modular) → 10% tráfico
Fase 2 (Semanas 4-5):  Validación y optimización → 50% tráfico  
Fase 3 (Semanas 6-10): Migración completa → 100% tráfico
```

**Principios de Diseño:**

- **Strangler Fig Pattern**: Migración incremental sin disrupciones
- **Monolito Modular**: Preparación para microservicios futuros manteniendo simplicidad operacional actual dado el tamaño del equipo actual.
- **Cloud-Native**: Aprovechamiento total de servicios administrados AWS para escalabilidad.

**Mitigación de Riesgos:**

- Rollback automático en <10 segundos
- Shadow mode para validación sin impacto a usuarios
- Feature flags para control granular por cliente/proyecto