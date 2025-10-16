# Guía Completa de Instancias AWS EC2: Tipos, Costos y Recomendaciones

## Resumen Ejecutivo

AWS EC2 ofrece **63 familias de instancias** distribuidas en cinco categorías principales, cada una optimizada para diferentes cargas de trabajo. Esta investigación proporciona una base de datos completa para tomar decisiones informadas sobre la selección de instancias, con información actualizada sobre costos, características y mejores prácticas.[1][2][3]

**Dato Clave**: Las instancias basadas en AWS Graviton pueden ofrecer **hasta 40% mejor precio-rendimiento** comparado con instancias x86 equivalentes.[4][5][6]

## Categorías de Instancias EC2

### 1. General Purpose (Propósito General)

Las instancias de propósito general ofrecen un balance entre CPU, memoria y red, ideales para cargas de trabajo diversas.[7][8][1]

**Familias Principales**:

- **T3/T3a/T4g** (Burstable): Diseñadas para cargas con uso moderado de CPU y picos ocasionales[9][10]
  - T3: Intel Xeon (3.1 GHz), $0.0832/hora (large)
  - T3a: AMD EPYC, $0.0752/hora (large) - **10% más económico que T3**[9]
  - T4g: AWS Graviton2, $0.0672/hora (large) - **40% mejor precio-rendimiento**[11][9]
  
- **M5/M6i/M7i/M7g** (Estables): Para aplicaciones que requieren recursos consistentes[12][13][14]
  - M5: Intel Xeon Platinum, $0.096/hora (large)
  - M6i: Intel Ice Lake con DDR5, $0.096/hora (large)[13]
  - M7i: Intel 4ª Gen, $0.1008/hora (large) - **15% mejor que M6i**[15]
  - M7g: AWS Graviton3, $0.0816/hora (large) - **25% mejor que M6g**[16][17]

**Casos de Uso**: Servidores web, repositorios de código, bases de datos pequeñas/medianas, entornos de desarrollo.[1][7][12]

### 2. Compute Optimized (Optimizado para Cómputo)

Instancias con alta relación CPU/memoria para cargas intensivas en procesamiento.[8][18][1]

**Familias Principales**:

- **C5/C6i/C7i**: Intel Xeon optimizado para cómputo[19][15]
  - C5: $0.085/hora (large)
  - C7i: Intel 4ª Gen, $0.089/hora (large)[15]
  
- **C6g/C7g**: AWS Graviton para cómputo eficiente[20][21]
  - C6g: Graviton2, $0.068/hora (large) - **40% mejor precio/rendimiento vs C5**[22]
  - C7g: Graviton3, $0.0725/hora (large) - **25% mejor vs C6g**[21][20]

**Casos de Uso**: Batch processing, HPC, modelado científico, transcodificación de medios, servidores de juegos, inferencia ML.[23][18][24]

### 3. Memory Optimized (Optimizado para Memoria)

Instancias con alta relación memoria/CPU, diseñadas para procesamiento de grandes datasets en memoria.[18][8][1]

**Familias Principales**:

- **R5/R6i/R7i/R7g**: Memoria optimizada general[25][17][19]
  - R5: Intel Xeon, $0.126/hora (large)
  - R6i: Intel Ice Lake, $0.126/hora (large)[26]
  - R7g: Graviton3, $0.108/hora (large) - **25% mejor rendimiento vs R6g**[17][25]
  
- **X1e/X2idn/High Memory**: Para cargas extremas de memoria[23][18]
  - X1e: Hasta 3.9 TB RAM, $2.001/hora (large)
  - High Memory: Hasta 24 TB RAM para SAP HANA producción[18]

**Casos de Uso**: Bases de datos en memoria (Redis, Memcached), SAP HANA, análisis de big data en tiempo real, procesamiento de datos de alta performance.[24][25][18]

### 4. Storage Optimized (Optimizado para Almacenamiento)

Instancias diseñadas para cargas que requieren alto IOPS y acceso secuencial rápido a grandes datasets.[27][8][1]

**Familias Principales**:

- **I3/I4i**: IOPS alto con almacenamiento NVMe local[28][29]
  - I3: Hasta 3,000 IOPS, $0.156/hora (large)[28]
  - I4i: Intel Ice Lake, hasta 30 TB NVMe, mejor IOPS[29]
  
- **D2/D3**: Almacenamiento HDD denso de bajo costo[27][23]
  - D2: HDD local denso, $0.690/hora (large)
  - D3: Hasta 336 TB almacenamiento HDD[27]

**Casos de Uso**: Data warehousing, NoSQL (Cassandra, MongoDB), Hadoop, Elasticsearch, procesamiento de logs.[23][28][27]

### 5. Accelerated Computing (Cómputo Acelerado)

Instancias con GPUs o aceleradores especializados para cargas de trabajo intensivas.[30][8][1]

**Familias Principales con GPU NVIDIA**:

- **P4d/P4de**: NVIDIA A100 (80GB) para entrenamiento ML
  - P4d: $32.77/hora - **Reducción de 33% en precio (junio 2025)**[31][30]
  
- **P5/P5en**: NVIDIA H100/H200 para LLMs
  - P5: H100 (80GB), $98.32/hora - **Reducción de 44% en precio**[30][31]
  - P5en: H200 (141GB), $72.50/hora - **Reducción de 25%**[31][30]
  
- **G5/G4dn**: Para gráficos y ML inference
  - G4dn: NVIDIA T4, $0.526/hora (base)[32]
  - G5: NVIDIA A10G, $1.006/hora (base)[32]

**Aceleradores AWS Personalizados**:

- **Inf2**: AWS Inferentia2 para inferencia ML, $0.76/hora - **2.3x mejor vs Inf1**[32]
- **Trn1/Trn1n**: AWS Trainium para entrenamiento ML con PyTorch, **3x mejor que P4**[32]

**Casos de Uso**: Entrenamiento de ML/DL, LLMs, inferencia ML, renderizado de gráficos, transcodificación de video.[33][30][32]

## Modelos de Precios AWS EC2

### Comparación de Modelos

| Modelo | Ahorro vs On-Demand | Compromiso | Flexibilidad | Mejor Para |
|--------|---------------------|------------|--------------|------------|
| **On-Demand** | 0% | Ninguno | Máxima | Cargas impredecibles, dev/test[2][34] |
| **Reserved Instances** | Hasta 72% | 1-3 años | Baja | Cargas estables 24/7[2][35][36] |
| **Savings Plans** | Hasta 72% | 1-3 años | Alta | Cargas estables con flexibilidad[35][37][38] |
| **Spot Instances** | Hasta 90% | Ninguno | Máxima | Tolerante a interrupciones[2][39][40] |
| **Dedicated Hosts** | Hasta 70% | 1-3 años | Baja | Cumplimiento, licencias BYOL[41][42] |

### Reserved Instances vs Savings Plans

**Cuándo usar Reserved Instances**:[35][36][38]
- Cargas de trabajo predecibles y estables
- Necesidad de reservar capacidad en AZ específica
- Workloads que no cambiarán de instancia/región
- Servicios como RDS, Redshift (solo RI disponible)

**Cuándo usar Savings Plans**:[36][38][35]
- Cargas de trabajo que pueden cambiar de instancia
- Necesidad de flexibilidad entre regiones
- Uso de múltiples servicios (EC2, Lambda, Fargate)
- Gestión simplificada sin configuración manual

**Diferencia Clave**: Savings Plans se basan en compromiso de gasto ($/hora), mientras que Reserved Instances se basan en tipos de instancia específicos.[38][35]

### Spot Instances: Mejores Prácticas

**Recomendaciones Clave**:[39][43][40][44]

1. **Diversificar instancias**: Usar múltiples tipos y AZs para maximizar disponibilidad
2. **Usar Auto Scaling**: Combinar instancias Spot con On-Demand como respaldo
3. **Implementar manejo de interrupciones**: Prepararse para avisos de 2 minutos
4. **Selección basada en atributos**: Usar attribute-based instance type selection para flexibilidad
5. **Estrategia capacity-optimized**: Priorizar pools con mayor capacidad disponible

**Casos de Uso Ideales**: Batch processing, CI/CD, big data, contenedores, HPC, análisis de datos.[40][44][39]

## Comparación de Procesadores

### Intel vs AMD vs Graviton

| Arquitectura | Ventajas | Precio/Rendimiento | Cuándo Usar |
|--------------|----------|-------------------|-------------|
| **Intel x86** | Amplio soporte software, alto rendimiento single-thread | Base (100%) | Cuando se requiere máxima compatibilidad[13][19] |
| **AMD x86** | 10-20% mejor costo, excelente multi-thread | 10-20% mejor | Balance costo/rendimiento en x86[19][45][46] |
| **Graviton2 (ARM)** | Eficiencia energética, amplia adopción | 40% mejor | Cargas compatibles ARM, ahorro significativo[4][5][6] |
| **Graviton3 (ARM)** | DDR5, 25% mejor vs G2, criptografía 2x más rápida | 25% mejor vs G2 | Última gen ARM, mejor rendimiento[25][17][21] |
| **Graviton4 (ARM)** | Mejor rendimiento ARM hasta la fecha | 30% mejor vs G3 | Máximo rendimiento ARM, última generación[33][24] |

**Datos de Rendimiento Real**:[47][45][46]
- **Intel** mantiene ventaja en aplicaciones x86 legacy
- **AMD** ofrece mejor relación precio/rendimiento en x86, ganando cuota de mercado (+88% YoY)[45]
- **Graviton** creciendo rápidamente (+66% YoY) pero requiere recompilación para ARM[6][4][45]

## Guía de Selección por Caso de Uso

### Casos de Uso Comunes

| Caso de Uso | Familia Recomendada | Modelo de Precio | Consideraciones |
|-------------|---------------------|------------------|-----------------|
| **Sitio web pequeño/mediano** | T3, T3a, T4g | On-Demand o Savings Plan | Verificar créditos CPU suficientes[48][49] |
| **Servidor web alto tráfico** | C5, C6i, C7g | Reserved o Savings Plan | Balanceo de carga, Auto Scaling[48][49] |
| **Base de datos relacional** | R5, R6i, R7g | Reserved o Savings Plan | Alta memoria, IOPS EBS optimizado[18][49] |
| **Base de datos en memoria** | R5, R6g, X2idn | Reserved o Savings Plan | Relación memoria/vCPU 8:1[18][19] |
| **Data warehouse** | I3, I4i, R5 | Reserved o Spot | Alto IOPS, NVMe local[27][29] |
| **Big Data (Hadoop/Spark)** | I3, D2, D3, R5 | Spot o Reserved | Almacenamiento denso, red alta[39][27] |
| **ML Training** | P4, P5, Trn1 | Reserved o Savings Plan | Múltiples GPUs, ancho de banda[30][31] |
| **ML Inference** | Inf2, G5, G4dn | On-Demand o Savings Plan | Costo/inferencia optimizado[32] |
| **CI/CD pipelines** | T3, C5 (Spot) | Spot | Tolerante a interrupciones[39][40] |
| **SAP HANA** | X1e, High Memory | Reserved | Memoria extrema, certificación SAP[18] |

## Optimización de Costos: Mejores Prácticas

### Estrategias de Alto Impacto

**1. Right-Sizing (Ajuste Correcto)**:[50][51][52]
- Usar AWS Compute Optimizer para recomendaciones basadas en uso real
- Revisar métricas de CloudWatch (CPU, memoria, red, disco)
- Downsizing puede ahorrar **30%** sin impacto en rendimiento
- Revisar y ajustar trimestralmente

**2. Adopción de Generaciones Recientes**:[51][53]
- M7 ofrece **15% mejor rendimiento** vs M6 al mismo precio
- Graviton3 ofrece **25% mejor** vs Graviton2
- AWS no actualiza automáticamente, requiere migración manual

**3. Estrategia de Pricing Mixta**:[54][55][51]
- Base estable: Reserved Instances o Savings Plans (60-72% ahorro)
- Picos predecibles: Auto Scaling con On-Demand
- Cargas tolerantes: Spot Instances (hasta 90% ahorro)
- Ejemplo: 50% RI + 30% On-Demand + 20% Spot

**4. Gestión de Recursos No Utilizados**:[56][54]
- Eliminar volúmenes EBS no adjuntos
- Liberar Elastic IPs no utilizados ($0.005/hora desde feb 2024)[57]
- Apagar instancias dev/test fuera de horario laboral
- Ahorro potencial: **10-20%** del gasto total

**5. Optimización Regional**:[58]
- us-east-1 (N. Virginia) típicamente más económico
- São Paulo (sa-east-1) puede ser **50% más caro**
- Mumbai (ap-south-1) aproximadamente **12.5% más caro**
- Considerar data residency vs costo

### Herramientas de Optimización

**AWS Native**:[48][59][51]
- **AWS Compute Optimizer**: Recomendaciones ML basadas en uso
- **AWS Cost Explorer**: Análisis de gastos y tendencias
- **AWS Trusted Advisor**: Mejores prácticas y oportunidades de ahorro
- **AWS Pricing Calculator**: Estimación de costos antes de desplegar

**Consideraciones de Auto Scaling**:[43][60][39]
- Definir políticas basadas en métricas reales (CPU, memoria, latencia)
- Implementar cooldown periods apropiados
- Usar target tracking para simplicidad
- Combinar con Spot para máximo ahorro

## Mejores Prácticas de Rendimiento

### Networking

**Enhanced Networking**:[61][62]
- Habilitar en todas las instancias modernas (gratis)
- Hasta **25 Gbps** de ancho de banda en instancias grandes
- Menor latencia y mayor PPS (packets per second)

**Placement Groups**:[3][61]
- **Cluster**: Baja latencia (microsegundos) dentro de AZ única
- **Spread**: Alta disponibilidad, máximo 7 instancias por AZ
- **Partition**: Para aplicaciones distribuidas (Hadoop, Cassandra)

### Almacenamiento

**EBS Optimization**:[63][62]
- Habilitado por defecto en generaciones actuales
- Throughput dedicado entre EC2 y EBS
- Critical para workloads I/O intensivos

**Selección de Tipo EBS**:[64]
- **gp3**: General purpose, $0.08/GB-mes (mejor para mayoría)
- **io2**: IOPS provisionado, $0.125/GB-mes (bases de datos críticas)
- **st1**: Throughput optimizado, $0.045/GB-mes (big data)
- **sc1**: Cold HDD, $0.015/GB-mes (archivos)

### Seguridad

**Mejores Prácticas Críticas**:[62][61]
- Usar IAM roles en lugar de credenciales hardcoded
- Habilitar encriptación EBS por defecto
- Implementar Security Groups restrictivos
- Usar AWS Systems Manager para gestión sin SSH
- Habilitar CloudTrail para auditoría

## Consideraciones de Disponibilidad

**Multi-AZ Deployment**:[58][48]
- Distribuir instancias en múltiples Availability Zones
- Usar Application/Network Load Balancers
- Auto Scaling con múltiples AZs
- RDS Multi-AZ para bases de datos

**Auto Recovery**:[60][65]
- Configurar CloudWatch alarmas para recuperación automática
- Usar Auto Scaling para reemplazo automático
- Implementar health checks apropiados

## Recursos de Datos Generados

Esta investigación ha generado **8 archivos CSV** descargables con información detallada:



**Base de Datos Principal**: 47 familias de instancias con especificaciones completas, procesadores, rangos de vCPU/memoria, casos de uso y precios estimados.



**Instancias Aceleradas**: 16 familias con GPUs/aceleradores, incluyendo P4, P5, G5, Inf2, Trn1 y sus especificaciones.



**Modelos de Precios**: Comparación detallada de On-Demand, Reserved Instances, Savings Plans, Spot y Dedicated Hosts.



**Guía de Casos de Uso**: 17 casos de uso comunes con familias recomendadas, modelos de precio y consideraciones clave.



**Comparación de Procesadores**: Análisis Intel vs AMD vs Graviton con ventajas, desventajas y cuándo usar cada uno.



**Matriz de Decisión**: 10 patrones de carga de trabajo con instancias recomendadas y ahorros estimados.



**Mejores Prácticas**: 18 prácticas categorizadas por área (selección, costos, rendimiento, seguridad) con beneficios y prioridades.



**Precios Regionales**: Comparación de costos en 8 regiones principales para familias T3, M5, C5 y R5.

## Conclusiones y Recomendaciones

### Recomendaciones Principales

1. **Para Ahorro Inmediato**: Migrar a instancias Graviton (T4g, M7g, C7g, R7g) puede generar **20-40% ahorro** sin sacrificar rendimiento[5][4][9]

2. **Para Cargas 24/7**: Implementar Reserved Instances o Savings Plans para lograr **60-72% ahorro** vs On-Demand[2][35][38]

3. **Para Batch Processing**: Usar Spot Instances con Auto Scaling puede generar **70-90% ahorro**[55][39][40]

4. **Para Alto Rendimiento**: Usar generaciones más recientes (M7, C7, R7) para **15-25% mejor rendimiento** al mismo costo[17][15]

5. **Para Balance Precio/Rendimiento**: Considerar AMD (M6a, C6a, R6a) o Graviton3 para **mejor eficiencia**[53][19]

### Estrategia de Implementación

**Fase 1 - Quick Wins (0-30 días)**:
- Implementar Compute Optimizer y revisar recomendaciones
- Eliminar recursos no utilizados (EBS, EIPs)
- Apagar instancias dev/test fuera de horario

**Fase 2 - Optimización Táctica (1-3 meses)**:
- Migrar cargas estables a Reserved Instances/Savings Plans
- Implementar Auto Scaling con Spot Instances
- Right-sizing basado en métricas reales

**Fase 3 - Optimización Estratégica (3-6 meses)**:
- Migrar aplicaciones compatibles a Graviton
- Actualizar a generaciones más recientes
- Implementar multi-región si es cost-effective

### Mantener la Base de Datos Actualizada

AWS actualiza constantemente sus ofertas de instancias. Para mantener esta base de datos relevante:

- Revisar AWS Pricing API trimestralmente
- Monitorear anuncios de nuevas generaciones (M8, C8, R8 próximamente)
- Actualizar con nuevas familias y características
- Revisar cambios en modelos de pricing
- Ajustar recomendaciones basadas en evolución del mercado