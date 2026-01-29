# Guía de Configuración e Integración de Membox

**Propósito**: Guía completa para configurar e integrar membox (arquitectura de memoria de continuidad temática) en tu sistema mempheromone.

**Estado**: ✅ Todos los componentes listos para implementación

[🇺🇸 English Version](MEMBOX_SETUP.md)

---

## Qué Cambió (2026-01-29)

### 1. **Parámetros Ajustados** ✅

Actualizado `/home/ike/.claude/plugins/rlm-prototype/scripts/mempheromone_membox.py`:

```python
# Antes → Después
TOPIC_CONTINUATION_THRESHOLD = 0.6 → 0.5   # Mejor agrupación
TOPIC_WINDOW_SIZE = 5 → 10                  # Memoria más larga
EVENT_LINK_THRESHOLD = 0.7 → 0.5            # Más enlaces entre temas
```

**Mejora Esperada**:
- Más memorias por caja (actualmente promedio 1.4 → objetivo 3-5)
- Mejor detección de continuidad temática
- Más enlaces de rastreo (actualmente 11% → objetivo 30-50%)

### 2. **Automatización Creada** ✅

**Tres opciones de automatización:**

#### Opción A: Cron Job (Recomendado)
```bash
# Instalar
crontab -e

# Agregar línea (se ejecuta cada hora en :00)
0 * * * * /home/ike/mempheromone/scripts/membox_cron.sh

# Verificar
crontab -l
```

**Registros**: `/var/log/mempheromone/membox_worker.log`

#### Opción B: Temporizador Systemd
```bash
# Instalar
sudo cp /home/ike/mempheromone/systemd/membox-worker.* /etc/systemd/system/
sudo systemctl daemon-reload

# Habilitar e iniciar
sudo systemctl enable membox-worker.timer
sudo systemctl start membox-worker.timer

# Verificar estado
systemctl status membox-worker.timer
journalctl -u membox-worker -f
```

#### Opción C: Manual/Bajo Demanda
```bash
# Procesar última hora
python3 /home/ike/mempheromone/scripts/membox_worker.py --since 1h

# Procesar últimas 24 horas
python3 /home/ike/mempheromone/scripts/membox_worker.py --since 24h

# Ejecución de prueba (vista previa)
python3 /home/ike/mempheromone/scripts/membox_worker.py --since 1h --dry-run
```

### 3. **Herramienta MCP Creada** ✅

**Herramienta**: `process_membox`

**Uso** (vía PostgreSQL MCP o herramienta Python):
```python
process_membox(since='1h', limit=100, dry_run=False)
```

**Respuesta**:
```json
{
  "success": true,
  "stats": {
    "found": 42,
    "processed": 42,
    "boxes_created": 8,
    "boxes_updated": 12,
    "errors": 0
  },
  "message": "Procesadas 42/42 memorias (creadas 8 cajas, actualizadas 12 cajas)"
}
```

---

## Instalación

### Paso 1: Verificar Prerequisitos

```bash
# Verificar base de datos
psql -U ike -d mempheromone -c "SELECT COUNT(*) FROM memory_boxes;"

# Verificar que existe el script membox
ls -lh /home/ike/.claude/plugins/rlm-prototype/scripts/mempheromone_membox.py

# Verificar parámetros ajustados
grep TOPIC_CONTINUATION_THRESHOLD /home/ike/.claude/plugins/rlm-prototype/scripts/mempheromone_membox.py
# Debe mostrar: TOPIC_CONTINUATION_THRESHOLD = 0.5
```

### Paso 2: Probar Script Worker

```bash
# Probar con ejecución de prueba
python3 /home/ike/mempheromone/scripts/membox_worker.py --since 24h --dry-run

# Salida esperada:
# Found X unboxed memories
# DRY RUN - would process:
#   [debugging_fact] Fixed authentication bug...
#   [claude_memory] Remember to check logs...
#   ...
```

### Paso 3: Ejecutar Procesamiento Inicial

```bash
# Procesar últimas 24 horas realmente
python3 /home/ike/mempheromone/scripts/membox_worker.py --since 24h

# Verificar resultados
psql -U ike -d mempheromone -c "
SELECT
  COUNT(*) as boxes,
  AVG(memory_count) as avg_per_box,
  MAX(updated_at) as last_updated
FROM memory_boxes;
"
```

### Paso 4: Instalar Automatización

**Elige una:**

#### Cron (Recomendado por Simplicidad)
```bash
# Editar crontab
crontab -e

# Agregar línea
0 * * * * /home/ike/mempheromone/scripts/membox_cron.sh

# Esperar 1 hora, luego verificar registros
tail -f /var/log/mempheromone/membox_worker.log
```

#### Systemd (Recomendado para Producción)
```bash
# Instalar
sudo cp /home/ike/mempheromone/systemd/membox-worker.* /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable membox-worker.timer
sudo systemctl start membox-worker.timer

# Verificar
systemctl status membox-worker.timer
systemctl list-timers | grep membox
```

### Paso 5: Verificar Que Funciona

```bash
# Esperar 1 hora después de la instalación, luego verificar:

# 1. Verificar actualizaciones recientes
psql -U ike -d mempheromone -c "
SELECT topic, memory_count, updated_at
FROM memory_boxes
ORDER BY updated_at DESC
LIMIT 10;
"

# 2. Verificar registros
tail -20 /var/log/mempheromone/membox_worker.log

# 3. Verificar que las cajas están creciendo
psql -U ike -d mempheromone -c "
SELECT
  COUNT(*) FILTER (WHERE memory_count > 1) as multi_memory_boxes,
  COUNT(*) FILTER (WHERE memory_count = 1) as single_memory_boxes,
  AVG(memory_count) as avg_memories_per_box
FROM memory_boxes;
"
```

**Criterios de Éxito**:
- `updated_at` debe estar dentro de la última hora
- `avg_memories_per_box` debe aumentar con el tiempo (objetivo: 3-5)
- `multi_memory_boxes` debe crecer (mejor agrupación)

---

## Monitoreo

### Verificación de Salud Diaria

```bash
# Verificar si membox está ejecutándose
systemctl status membox-worker.timer  # Si usas systemd

# O
tail -50 /var/log/mempheromone/membox_worker.log  # Si usas cron

# Verificar estadísticas de base de datos
psql -U ike -d mempheromone -c "
SELECT
  COUNT(*) as total_boxes,
  COUNT(*) FILTER (WHERE is_active = TRUE) as active_boxes,
  AVG(memory_count) as avg_memories_per_box,
  MAX(memory_count) as max_in_box,
  AVG(pheromone_score) as avg_pheromone,
  MAX(updated_at) as last_updated,
  EXTRACT(EPOCH FROM (NOW() - MAX(updated_at)))/3600 as hours_since_update
FROM memory_boxes;
"
```

### Revisión Semanal

```bash
# Obtener estadísticas detalladas
python3 /home/ike/mempheromone/scripts/membox_worker.py --since 7d --dry-run

# Verificar crecimiento de enlaces de rastreo
psql -U ike -d mempheromone -c "
SELECT COUNT(*) as total_links FROM trace_links;

SELECT
  mb1.topic as source_topic,
  mb2.topic as target_topic,
  tl.similarity_score,
  tl.linking_events
FROM trace_links tl
JOIN memory_boxes mb1 ON tl.source_box_id = mb1.id
JOIN memory_boxes mb2 ON tl.target_box_id = mb2.id
ORDER BY tl.similarity_score DESC
LIMIT 10;
"
```

---

## Solución de Problemas

### Problema 1: No Se Crean Cajas Nuevas

**Síntomas**:
- `boxes_created: 0` en salida del worker
- `avg_memories_per_box` no aumenta

**Diagnóstico**:
```bash
# Verificar si hay memorias sin caja
psql -U ike -d mempheromone -c "
SELECT COUNT(*) as unboxed_count
FROM debugging_facts df
LEFT JOIN memory_box_items mbi ON df.fact_id = mbi.memory_id
WHERE mbi.box_id IS NULL
  AND df.first_seen > NOW() - INTERVAL '24 hours';
"
```

**Soluciones**:
- Si el conteo es 0: No hay nuevas memorias para procesar (esperado)
- Si el conteo > 0 pero no se crean cajas: El umbral de tema puede ser muy estricto
  - Bajar `TOPIC_CONTINUATION_THRESHOLD` a 0.4
  - Aumentar `TOPIC_WINDOW_SIZE` a 15

### Problema 2: Cron Job No Se Ejecuta

**Síntomas**:
- No hay entradas de registro en `/var/log/mempheromone/membox_worker.log`
- `last_updated` está obsoleto

**Diagnóstico**:
```bash
# Verificar crontab
crontab -l | grep membox

# Verificar servicio cron
systemctl status cron

# Verificar errores
grep membox /var/log/syslog
```

**Soluciones**:
- Asegurar que crontab tiene la ruta correcta
- Verificar permisos de script: `chmod +x /home/ike/mempheromone/scripts/membox_cron.sh`
- Verificar que existe el directorio de registros: `mkdir -p /var/log/mempheromone`

### Problema 3: Temporizador Systemd No Se Activa

**Diagnóstico**:
```bash
# Verificar estado del temporizador
systemctl status membox-worker.timer

# Verificar estado del servicio
systemctl status membox-worker.service

# Ver registros
journalctl -u membox-worker -n 50
```

**Soluciones**:
- Asegurar que el temporizador está habilitado: `sudo systemctl enable membox-worker.timer`
- Verificar horario del temporizador: `systemctl list-timers | grep membox`
- Reiniciar temporizador: `sudo systemctl restart membox-worker.timer`

### Problema 4: Baja Calidad de Agrupación

**Síntomas**:
- La mayoría de las cajas tienen solo 1 memoria
- `avg_memories_per_box` se mantiene alrededor de 1.0

**Soluciones**:
1. **Bajar umbral de tema**:
   ```bash
   # Editar script membox
   vim /home/ike/.claude/plugins/rlm-prototype/scripts/mempheromone_membox.py

   # Cambiar:
   TOPIC_CONTINUATION_THRESHOLD = 0.4  # De 0.5
   ```

2. **Aumentar tamaño de ventana**:
   ```python
   TOPIC_WINDOW_SIZE = 15  # De 10
   ```

3. **Verificar extracción de palabras clave**:
   ```sql
   -- Ver qué palabras clave se están extrayendo
   SELECT keyword, COUNT(*) as frequency
   FROM (
     SELECT unnest(keywords) as keyword
     FROM memory_boxes
   ) kw
   GROUP BY keyword
   ORDER BY frequency DESC
   LIMIT 30;
   ```

---

## Ajuste de Rendimiento

### Para Sistemas de Alto Volumen (>1000 memorias/día)

```bash
# Ejecutar worker con más frecuencia
# En lugar de cada hora, ejecutar cada 15 minutos
*/15 * * * * /home/ike/mempheromone/scripts/membox_cron.sh

# Procesar lotes más pequeños
python3 /home/ike/mempheromone/scripts/membox_worker.py --since 15m --limit 50
```

### Para Sistemas de Bajo Volumen (<100 memorias/día)

```bash
# Ejecutar con menos frecuencia para mejores lotes
# Ejecutar cada 6 horas
0 */6 * * * /home/ike/mempheromone/scripts/membox_cron.sh

# Procesar ventanas más grandes
python3 /home/ike/mempheromone/scripts/membox_worker.py --since 6h --limit 500
```

### Optimización de Memoria

```sql
-- Si la tabla memory_boxes se vuelve muy grande (>10,000 cajas)

-- Archivar cajas inactivas antiguas
UPDATE memory_boxes
SET is_active = FALSE
WHERE updated_at < NOW() - INTERVAL '90 days'
  AND pheromone_score < 5.0;

-- VACUUM para recuperar espacio
VACUUM ANALYZE memory_boxes;
```

---

## Resultados Esperados Después de 1 Semana

**Antes de la Integración**:
- Total de cajas: 749
- Promedio memorias/caja: 1.4
- Enlaces de rastreo: 84 (11%)
- Última actualización: hace 3 días

**Después de la Integración (1 semana)**:
- Total de cajas: 900-1200 (creciendo con nuevas memorias)
- Promedio memorias/caja: 3-5 (mejor agrupación)
- Enlaces de rastreo: 400-800 (40-60% de vinculación)
- Última actualización: <1 hora (ejecutándose activamente)

**Después de la Integración (1 mes)**:
- Total de cajas: 1500-2500
- Promedio memorias/caja: 5-8 (fuerte continuidad temática)
- Enlaces de rastreo: 1500+ (navegación rica entre temas)
- Puntuaciones de feromonas evolucionando naturalmente basado en uso

---

## Integración con Otras Herramientas

### Exportación RLM

Membox ya está integrado con la exportación RLM si estás usando la última versión:

```bash
# La exportación incluye cajas de memoria
python3 ~/.claude/plugins/rlm-prototype/scripts/mempheromone_export.py \
  --output /tmp/context.txt

# Verificar si las cajas están incluidas
grep -A5 "Memory Boxes" /tmp/context.txt
```

### Herramientas MCP

Agrega a tu configuración de servidor MCP para habilitar la herramienta `process_membox`:

```json
{
  "mcpServers": {
    "mempheromone": {
      "command": "python3",
      "args": ["/home/ike/mempheromone/mcp_tools/process_membox.py"],
      "env": {
        "PGHOST": "/var/run/postgresql",
        "PGDATABASE": "mempheromone"
      }
    }
  }
}
```

---

## Próximos Pasos

1. ✅ Instalar automatización (cron o systemd)
2. ✅ Ejecutar procesamiento inicial
3. ⏳ Esperar 24 horas
4. ⏳ Verificar que funciona (verificar registros + base de datos)
5. ⏳ Monitorear por 1 semana
6. ⏳ Ajustar parámetros si es necesario

---

## Soporte

Para problemas o preguntas:
- Verificar registros: `/var/log/mempheromone/membox_worker.log`
- Ejecutar diagnóstico: `python3 scripts/membox_worker.py --since 24h --dry-run --verbose`
- Usar herramienta MCP: `check_membox_status` (ver DATABASE_MANAGEMENT_TOOLS.md)

---

**¡Tu sistema membox está listo para implementar! 🚀**
