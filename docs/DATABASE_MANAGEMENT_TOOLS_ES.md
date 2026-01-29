# Herramientas MCP de Gestión de Base de Datos

**Propósito**: Herramientas esenciales de gestión y monitoreo de base de datos para el sistema mempheromone.

**Ubicación**: Estas herramientas deben incluirse en el servidor MCP principal (`mcp-server/src/tools/db_management/`)

[🇺🇸 English Version](DATABASE_MANAGEMENT_TOOLS.md)

---

## Catálogo de Herramientas

### 1. Estadísticas de Base de Datos

**Herramienta**: `get_database_stats`

**Propósito**: Obtener visión general de la salud y tamaño de la base de datos

**Implementación**:

```typescript
import { z } from 'zod';
import { Pool } from 'pg';

const GetDatabaseStatsSchema = z.object({
  include_details: z.boolean().default(false).describe('Incluir estadísticas detalladas de tablas')
});

export async function get_database_stats(
  args: z.infer<typeof GetDatabaseStatsSchema>,
  pool: Pool
): Promise<string> {
  const { include_details } = args;

  // Estadísticas generales
  const statsQuery = `
    SELECT
      (SELECT COUNT(*) FROM debugging_facts) as debugging_facts,
      (SELECT COUNT(*) FROM claude_memories) as claude_memories,
      (SELECT COUNT(*) FROM crystallization_events) as crystallizations,
      (SELECT COUNT(*) FROM session_narratives) as narratives,
      (SELECT COUNT(*) FROM wisdom) as wisdom_entries,
      (SELECT COUNT(*) FROM embeddings) as embeddings,
      (SELECT COUNT(*) FROM memory_boxes) as memory_boxes,
      (SELECT COUNT(*) FROM trace_links) as trace_links,
      (SELECT AVG(pheromone_score) FROM debugging_facts) as avg_pheromone,
      (SELECT COUNT(*) FROM debugging_facts WHERE pheromone_score >= 15) as expert_facts,
      (SELECT COUNT(*) FROM debugging_facts WHERE pheromone_score >= 10) as solid_facts,
      (SELECT pg_database_size(current_database())) as db_size_bytes
  `;

  const result = await pool.query(statsQuery);
  const stats = result.rows[0];

  let output = `📊 Estadísticas de Base de Datos Mempheromone\n\n`;
  output += `Conteos de Memoria:\n`;
  output += `  Hechos de Depuración: ${stats.debugging_facts}\n`;
  output += `  Memorias de Claude: ${stats.claude_memories}\n`;
  output += `  Cristalizaciones: ${stats.crystallizations}\n`;
  output += `  Narrativas de Sesión: ${stats.narratives}\n`;
  output += `  Entradas de Sabiduría: ${stats.wisdom_entries}\n`;
  output += `  Incrustaciones: ${stats.embeddings}\n`;
  output += `  Cajas de Memoria: ${stats.memory_boxes}\n`;
  output += `  Enlaces de Rastreo: ${stats.trace_links}\n\n`;

  output += `Métricas de Calidad:\n`;
  output += `  Feromona Promedio: ${parseFloat(stats.avg_pheromone).toFixed(2)}\n`;
  output += `  Hechos Expertos (≥15): ${stats.expert_facts}\n`;
  output += `  Hechos Sólidos (≥10): ${stats.solid_facts}\n\n`;

  const dbSizeMB = (parseInt(stats.db_size_bytes) / 1024 / 1024).toFixed(2);
  output += `Tamaño de Base de Datos: ${dbSizeMB} MB\n`;

  if (include_details) {
    // Tamaños de tablas
    const tableSizeQuery = `
      SELECT
        schemaname,
        tablename,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
      FROM pg_tables
      WHERE schemaname = 'public'
      ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
      LIMIT 10
    `;

    const tableSizes = await pool.query(tableSizeQuery);

    output += `\nTop 10 Tablas por Tamaño:\n`;
    for (const row of tableSizes.rows) {
      output += `  ${row.tablename}: ${row.size}\n`;
    }
  }

  return output;
}

export const get_database_stats_tool = {
  name: 'get_database_stats',
  description: 'Obtener visión general de salud, conteos de memoria y métricas de calidad de la base de datos mempheromone',
  inputSchema: GetDatabaseStatsSchema
};
```

---

### 2. Análisis de Distribución de Feromonas

**Herramienta**: `get_pheromone_distribution`

**Propósito**: Analizar la distribución de puntuación de feromonas para entender la calidad de memoria

**Implementación**:

```typescript
const GetPheromoneDistributionSchema = z.object({
  memory_type: z.enum(['debugging_facts', 'memory_boxes', 'all']).default('all')
});

export async function get_pheromone_distribution(
  args: z.infer<typeof GetPheromoneDistributionSchema>,
  pool: Pool
): Promise<string> {
  const { memory_type } = args;

  let output = `📈 Distribución de Puntuación de Feromonas\n\n`;

  if (memory_type === 'debugging_facts' || memory_type === 'all') {
    const debuggingQuery = `
      SELECT
        CASE
          WHEN pheromone_score >= 20 THEN 'Experto+ (20)'
          WHEN pheromone_score >= 15 THEN 'Experto (15-19)'
          WHEN pheromone_score >= 12 THEN 'Sólido+ (12-14)'
          WHEN pheromone_score >= 10 THEN 'Sólido (10-11)'
          WHEN pheromone_score >= 5 THEN 'No Probado (5-9)'
          ELSE 'Bajo (<5)'
        END as tier,
        COUNT(*) as count,
        AVG(pheromone_score) as avg_score,
        MIN(pheromone_score) as min_score,
        MAX(pheromone_score) as max_score
      FROM debugging_facts
      GROUP BY tier
      ORDER BY MIN(pheromone_score) DESC
    `;

    const result = await pool.query(debuggingQuery);

    output += `Hechos de Depuración:\n`;
    for (const row of result.rows) {
      const pct = (row.count / result.rows.reduce((sum, r) => sum + parseInt(r.count), 0) * 100).toFixed(1);
      output += `  ${row.tier}: ${row.count} (${pct}%) - prom: ${parseFloat(row.avg_score).toFixed(2)}\n`;
    }
    output += '\n';
  }

  if (memory_type === 'memory_boxes' || memory_type === 'all') {
    const boxesQuery = `
      SELECT
        CASE
          WHEN pheromone_score >= 20 THEN 'Experto+ (20)'
          WHEN pheromone_score >= 15 THEN 'Experto (15-19)'
          WHEN pheromone_score >= 12 THEN 'Sólido+ (12-14)'
          WHEN pheromone_score >= 10 THEN 'Sólido (10-11)'
          WHEN pheromone_score >= 5 THEN 'No Probado (5-9)'
          ELSE 'Bajo (<5)'
        END as tier,
        COUNT(*) as count,
        AVG(pheromone_score) as avg_score,
        AVG(memory_count) as avg_memories_per_box
      FROM memory_boxes
      WHERE is_active = TRUE
      GROUP BY tier
      ORDER BY MIN(pheromone_score) DESC
    `;

    const result = await pool.query(boxesQuery);

    output += `Cajas de Memoria:\n`;
    for (const row of result.rows) {
      output += `  ${row.tier}: ${row.count} cajas - prom: ${parseFloat(row.avg_score).toFixed(2)} - prom memorias/caja: ${parseFloat(row.avg_memories_per_box).toFixed(1)}\n`;
    }
  }

  return output;
}

export const get_pheromone_distribution_tool = {
  name: 'get_pheromone_distribution',
  description: 'Analizar distribución de puntuación de feromonas entre tipos de memoria',
  inputSchema: GetPheromoneDistributionSchema
};
```

---

### 3. Análisis de Calidad de Memoria

**Herramienta**: `analyze_memory_quality`

**Propósito**: Identificar problemas de calidad y sugerir mejoras

**Implementación**:

```typescript
export async function analyze_memory_quality(pool: Pool): Promise<string> {
  let output = `🔍 Análisis de Calidad de Memoria\n\n`;

  // 1. Memorias obsoletas (no accedidas recientemente)
  const staleQuery = `
    SELECT COUNT(*) as stale_count
    FROM debugging_facts
    WHERE last_accessed < NOW() - INTERVAL '90 days'
      AND pheromone_score < 10
  `;
  const stale = await pool.query(staleQuery);
  output += `Memorias Obsoletas (>90 días, feromona baja): ${stale.rows[0].stale_count}\n`;

  // 2. Incrustaciones faltantes
  const missingEmbeddingsQuery = `
    SELECT COUNT(*) as missing
    FROM debugging_facts df
    LEFT JOIN embeddings e ON df.fact_id = e.memory_id
    WHERE e.memory_id IS NULL
  `;
  const missing = await pool.query(missingEmbeddingsQuery);
  output += `Incrustaciones Faltantes: ${missing.rows[0].missing}\n`;

  // 3. Memorias nunca accedidas
  const neverAccessedQuery = `
    SELECT COUNT(*) as never_accessed
    FROM debugging_facts
    WHERE search_count = 0 OR search_count IS NULL
  `;
  const neverAccessed = await pool.query(neverAccessedQuery);
  output += `Nunca Accedidas: ${neverAccessed.rows[0].never_accessed}\n`;

  // 4. Alta feromona pero nunca recuperadas
  const highButUnusedQuery = `
    SELECT COUNT(*) as count
    FROM debugging_facts
    WHERE pheromone_score >= 12
      AND (retrieval_count = 0 OR retrieval_count IS NULL)
  `;
  const highUnused = await pool.query(highButUnusedQuery);
  output += `Alta Feromona pero Nunca Recuperadas: ${highUnused.rows[0].count}\n`;

  // 5. Cobertura de membox
  const memboxCoverageQuery = `
    SELECT
      COUNT(DISTINCT df.fact_id) as total_facts,
      COUNT(DISTINCT mbi.memory_id) as facts_in_boxes,
      (COUNT(DISTINCT mbi.memory_id)::float / COUNT(DISTINCT df.fact_id) * 100) as coverage_pct
    FROM debugging_facts df
    LEFT JOIN memory_box_items mbi ON df.fact_id = mbi.memory_id AND mbi.memory_type = 'debugging_fact'
  `;
  const coverage = await pool.query(memboxCoverageQuery);
  output += `\nCobertura de Membox:\n`;
  output += `  Total de Hechos: ${coverage.rows[0].total_facts}\n`;
  output += `  En Cajas: ${coverage.rows[0].facts_in_boxes}\n`;
  output += `  Cobertura: ${parseFloat(coverage.rows[0].coverage_pct).toFixed(1)}%\n`;

  // 6. Recomendaciones
  output += `\n💡 Recomendaciones:\n`;

  if (parseInt(stale.rows[0].stale_count) > 100) {
    output += `  ⚠️  Considerar limpiar ${stale.rows[0].stale_count} memorias obsoletas de baja calidad\n`;
  }

  if (parseInt(missing.rows[0].missing) > 0) {
    output += `  ⚠️  Regenerar ${missing.rows[0].missing} incrustaciones faltantes\n`;
  }

  if (parseFloat(coverage.rows[0].coverage_pct) < 50) {
    output += `  ⚠️  Baja cobertura de membox (${parseFloat(coverage.rows[0].coverage_pct).toFixed(1)}%) - considerar ejecutar bootstrap de membox\n`;
  }

  return output;
}

export const analyze_memory_quality_tool = {
  name: 'analyze_memory_quality',
  description: 'Identificar problemas de calidad y proporcionar recomendaciones de mejora',
  inputSchema: z.object({})
};
```

---

### 4. Limpiar Memorias de Baja Calidad

**Herramienta**: `cleanup_low_quality_memories`

**Propósito**: Eliminar o archivar memorias debajo del umbral de calidad

**Implementación**:

```typescript
const CleanupLowQualityMemoriesSchema = z.object({
  min_pheromone: z.number().default(3.0).describe('Eliminar memorias debajo de esta puntuación'),
  min_age_days: z.number().default(90).describe('Solo eliminar memorias más antiguas que esto'),
  dry_run: z.boolean().default(true).describe('Vista previa de eliminaciones sin eliminar realmente')
});

export async function cleanup_low_quality_memories(
  args: z.infer<typeof CleanupLowQualityMemoriesSchema>,
  pool: Pool
): Promise<string> {
  const { min_pheromone, min_age_days, dry_run } = args;

  // Encontrar candidatos para eliminación
  const findQuery = `
    SELECT fact_id, symptom, pheromone_score, last_accessed, search_count
    FROM debugging_facts
    WHERE pheromone_score < $1
      AND last_accessed < NOW() - INTERVAL '${min_age_days} days'
      AND (search_count = 0 OR search_count IS NULL)
    ORDER BY pheromone_score ASC
    LIMIT 100
  `;

  const candidates = await pool.query(findQuery, [min_pheromone]);

  let output = `🧹 Limpiar Memorias de Baja Calidad\n\n`;
  output += `Criterios:\n`;
  output += `  Feromona < ${min_pheromone}\n`;
  output += `  Edad > ${min_age_days} días\n`;
  output += `  Nunca buscadas\n\n`;

  output += `Encontrados ${candidates.rows.length} candidatos para eliminación:\n\n`;

  if (candidates.rows.length === 0) {
    return output + `No hay memorias que coincidan con los criterios de limpieza. ¡La base de datos está limpia! ✨\n`;
  }

  for (const row of candidates.rows.slice(0, 10)) {
    output += `  [ψ=${row.pheromone_score.toFixed(1)}] ${row.symptom.substring(0, 60)}...\n`;
  }

  if (candidates.rows.length > 10) {
    output += `  ... y ${candidates.rows.length - 10} más\n`;
  }

  if (dry_run) {
    output += `\n⚠️  EJECUCIÓN DE PRUEBA - No se realizaron eliminaciones.\n`;
    output += `Establecer dry_run=false para eliminar realmente estas memorias.\n`;
  } else {
    // Eliminar realmente
    const deleteQuery = `
      DELETE FROM debugging_facts
      WHERE pheromone_score < $1
        AND last_accessed < NOW() - INTERVAL '${min_age_days} days'
        AND (search_count = 0 OR search_count IS NULL)
    `;

    const deleteResult = await pool.query(deleteQuery, [min_pheromone]);

    output += `\n✅ Eliminadas ${deleteResult.rowCount} memorias de baja calidad.\n`;
  }

  return output;
}

export const cleanup_low_quality_memories_tool = {
  name: 'cleanup_low_quality_memories',
  description: 'Eliminar o archivar memorias debajo del umbral de calidad (soporta dry-run)',
  inputSchema: CleanupLowQualityMemoriesSchema
};
```

---

### 5. Reconstruir Incrustaciones

**Herramienta**: `rebuild_embeddings`

**Propósito**: Regenerar incrustaciones para memorias que las tienen faltantes

**Implementación**:

```typescript
const RebuildEmbeddingsSchema = z.object({
  memory_type: z.enum(['debugging_facts', 'claude_memories', 'all']).default('all'),
  limit: z.number().default(100).describe('Máximo de incrustaciones a generar por ejecución'),
  force: z.boolean().default(false).describe('Regenerar incluso si la incrustación existe')
});

export async function rebuild_embeddings(
  args: z.infer<typeof RebuildEmbeddingsSchema>,
  pool: Pool
): Promise<string> {
  const { memory_type, limit, force } = args;

  let output = `🔄 Reconstruir Incrustaciones\n\n`;

  // Encontrar memorias faltantes incrustaciones
  let findQuery: string;

  if (memory_type === 'debugging_facts' || memory_type === 'all') {
    findQuery = `
      SELECT df.fact_id as memory_id, df.symptom || ': ' || df.solution as content
      FROM debugging_facts df
      LEFT JOIN embeddings e ON df.fact_id = e.memory_id
      WHERE ${force ? 'TRUE' : 'e.memory_id IS NULL'}
      LIMIT $1
    `;

    const missing = await pool.query(findQuery, [limit]);

    output += `Hechos de Depuración: ${missing.rows.length} incrustaciones a generar\n`;

    // Nota: La generación real de incrustaciones requeriría llamar al servicio de incrustaciones
    // Esto es un marcador de posición mostrando la estructura
    output += `⚠️  La generación de incrustaciones requiere servicio externo (sentence-transformers)\n`;
    output += `   Usar: python3 scripts/generate_embeddings.py --limit ${limit}\n`;
  }

  return output;
}

export const rebuild_embeddings_tool = {
  name: 'rebuild_embeddings',
  description: 'Regenerar incrustaciones para memorias que las tienen faltantes',
  inputSchema: RebuildEmbeddingsSchema
};
```

---

### 6. Verificar Estado de Membox

**Herramienta**: `check_membox_status`

**Propósito**: Verificar si el sistema membox funciona correctamente

**Implementación**:

```typescript
export async function check_membox_status(pool: Pool): Promise<string> {
  let output = `📦 Estado del Sistema Membox\n\n`;

  // 1. Verificar si existen tablas
  const tablesQuery = `
    SELECT tablename
    FROM pg_tables
    WHERE schemaname = 'public'
      AND tablename IN ('memory_boxes', 'memory_box_items', 'trace_links')
    ORDER BY tablename
  `;

  const tables = await pool.query(tablesQuery);
  const tableNames = tables.rows.map(r => r.tablename);

  output += `Tablas Requeridas:\n`;
  output += `  memory_boxes: ${tableNames.includes('memory_boxes') ? '✅' : '❌ FALTANTE'}\n`;
  output += `  memory_box_items: ${tableNames.includes('memory_box_items') ? '✅' : '❌ FALTANTE'}\n`;
  output += `  trace_links: ${tableNames.includes('trace_links') ? '✅' : '❌ FALTANTE'}\n\n`;

  if (tableNames.length < 3) {
    output += `⚠️  ¡Tablas de membox faltantes! Ejecutar migración de esquema:\n`;
    output += `   psql -d mempheromone -f schema/membox_tables.sql\n`;
    return output;
  }

  // 2. Verificar si existen cajas
  const boxCountQuery = `
    SELECT
      COUNT(*) as total_boxes,
      COUNT(*) FILTER (WHERE is_active = TRUE) as active_boxes,
      AVG(memory_count) as avg_memories_per_box,
      MAX(memory_count) as max_memories_in_box,
      AVG(pheromone_score) as avg_pheromone
    FROM memory_boxes
  `;

  const boxStats = await pool.query(boxCountQuery);
  const stats = boxStats.rows[0];

  output += `Cajas de Memoria:\n`;
  output += `  Total: ${stats.total_boxes}\n`;
  output += `  Activas: ${stats.active_boxes}\n`;
  output += `  Prom Memorias/Caja: ${parseFloat(stats.avg_memories_per_box || 0).toFixed(1)}\n`;
  output += `  Máx Memorias en Caja: ${stats.max_memories_in_box || 0}\n`;
  output += `  Prom Feromona: ${parseFloat(stats.avg_pheromone || 0).toFixed(2)}\n\n`;

  if (parseInt(stats.total_boxes) === 0) {
    output += `⚠️  ¡No existen cajas de memoria! Sistema membox no inicializado.\n`;
    output += `   Ejecutar: python3 scripts/mempheromone_membox.py bootstrap\n\n`;
  }

  // 3. Verificar enlaces de rastreo
  const linksQuery = `
    SELECT COUNT(*) as total_links
    FROM trace_links
  `;

  const links = await pool.query(linksQuery);
  output += `Enlaces de Rastreo: ${links.rows[0].total_links}\n\n`;

  if (parseInt(links.rows[0].total_links) === 0 && parseInt(stats.total_boxes) > 0) {
    output += `⚠️  No existen enlaces de rastreo pero sí cajas. El tejedor de rastreo puede no estar ejecutándose.\n\n`;
  }

  // 4. Actividad reciente
  const recentActivityQuery = `
    SELECT
      MAX(created_at) as last_box_created,
      MAX(updated_at) as last_box_updated
    FROM memory_boxes
  `;

  const activity = await pool.query(recentActivityQuery);

  if (activity.rows[0].last_box_created) {
    const lastCreated = new Date(activity.rows[0].last_box_created);
    const lastUpdated = new Date(activity.rows[0].last_box_updated);
    const daysSinceCreated = Math.floor((Date.now() - lastCreated.getTime()) / (1000 * 60 * 60 * 24));
    const daysSinceUpdated = Math.floor((Date.now() - lastUpdated.getTime()) / (1000 * 60 * 60 * 24));

    output += `Actividad Reciente:\n`;
    output += `  Última Caja Creada: hace ${daysSinceCreated} días\n`;
    output += `  Última Caja Actualizada: hace ${daysSinceUpdated} días\n\n`;

    if (daysSinceUpdated > 7) {
      output += `⚠️  Sin actualizaciones recientes a cajas (${daysSinceUpdated} días). Membox puede no estar integrado en el flujo de trabajo.\n`;
      output += `   Verificar si membox se llama cuando se almacenan nuevas memorias.\n\n`;
    } else {
      output += `✅ ¡El sistema membox se está usando activamente!\n\n`;
    }
  }

  // 5. Estado general
  const isWorking =
    tableNames.length === 3 &&
    parseInt(stats.total_boxes) > 0 &&
    parseInt(links.rows[0].total_links) > 0 &&
    activity.rows[0].last_box_updated;

  output += `Estado General: ${isWorking ? '✅ FUNCIONANDO' : '⚠️  NECESITA ATENCIÓN'}\n`;

  return output;
}

export const check_membox_status_tool = {
  name: 'check_membox_status',
  description: 'Verificar si el sistema membox (memoria de continuidad temática) funciona correctamente',
  inputSchema: z.object({})
};
```

---

### 7. Optimizar Base de Datos

**Herramienta**: `optimize_database`

**Propósito**: Ejecutar VACUUM ANALYZE y optimizar el rendimiento de consultas

**Implementación**:

```typescript
const OptimizeDatabaseSchema = z.object({
  full_vacuum: z.boolean().default(false).describe('Ejecutar VACUUM FULL (requiere bloqueo exclusivo)'),
  reindex: z.boolean().default(false).describe('Reconstruir todos los índices')
});

export async function optimize_database(
  args: z.infer<typeof OptimizeDatabaseSchema>,
  pool: Pool
): Promise<string> {
  const { full_vacuum, reindex } = args;

  let output = `⚙️  Optimización de Base de Datos\n\n`;

  try {
    // 1. VACUUM ANALYZE
    output += `Ejecutando VACUUM ANALYZE...\n`;
    if (full_vacuum) {
      await pool.query('VACUUM FULL ANALYZE');
      output += `  ✅ VACUUM FULL ANALYZE completo (espacio en disco recuperado)\n`;
    } else {
      await pool.query('VACUUM ANALYZE');
      output += `  ✅ VACUUM ANALYZE completo (estadísticas actualizadas)\n`;
    }

    // 2. Reindexar si se solicitó
    if (reindex) {
      output += `\nReconstruyendo índices...\n`;

      const indexes = [
        'idx_debugging_facts_pheromone',
        'idx_embeddings_hnsw',
        'idx_memory_boxes_pheromone',
        'idx_legal_cases_pheromone'
      ];

      for (const idx of indexes) {
        try {
          await pool.query(`REINDEX INDEX CONCURRENTLY ${idx}`);
          output += `  ✅ ${idx}\n`;
        } catch (err) {
          output += `  ⚠️  ${idx} - ${err.message}\n`;
        }
      }
    }

    // 3. Actualizar estadísticas de tabla
    output += `\nActualizando estadísticas...\n`;
    await pool.query('ANALYZE');
    output += `  ✅ Estadísticas actualizadas\n`;

    output += `\n✅ ¡Optimización de base de datos completa!\n`;
  } catch (err) {
    output += `\n❌ Error durante optimización: ${err.message}\n`;
  }

  return output;
}

export const optimize_database_tool = {
  name: 'optimize_database',
  description: 'Ejecutar VACUUM ANALYZE y optimizar el rendimiento de consultas',
  inputSchema: OptimizeDatabaseSchema
};
```

---

### 8. Respaldar Memorias

**Herramienta**: `backup_memories`

**Propósito**: Exportar memorias a JSON para respaldo

**Implementación**:

```typescript
const BackupMemoriesSchema = z.object({
  output_path: z.string().default('/tmp/mempheromone_backup.json'),
  include_types: z.array(z.string()).default(['debugging_facts', 'claude_memories', 'crystallizations'])
});

export async function backup_memories(
  args: z.infer<typeof BackupMemoriesSchema>,
  pool: Pool
): Promise<string> {
  const { output_path, include_types } = args;

  let output = `💾 Respaldar Memorias\n\n`;

  // Nota: La escritura real de archivos requeriría acceso al sistema de archivos
  // Esto muestra la estructura de consulta

  for (const type of include_types) {
    let query: string;
    let count = 0;

    if (type === 'debugging_facts') {
      const result = await pool.query(`SELECT COUNT(*) FROM debugging_facts`);
      count = parseInt(result.rows[0].count);
      output += `  Hechos de Depuración: ${count} memorias\n`;
    } else if (type === 'claude_memories') {
      const result = await pool.query(`SELECT COUNT(*) FROM claude_memories`);
      count = parseInt(result.rows[0].count);
      output += `  Memorias de Claude: ${count} memorias\n`;
    } else if (type === 'crystallizations') {
      const result = await pool.query(`SELECT COUNT(*) FROM crystallization_events`);
      count = parseInt(result.rows[0].count);
      output += `  Cristalizaciones: ${count} memorias\n`;
    }
  }

  output += `\n⚠️  La exportación de archivos requiere acceso al sistema de archivos.\n`;
  output += `   Usar: python3 scripts/backup_memories.py --output ${output_path}\n`;

  return output;
}

export const backup_memories_tool = {
  name: 'backup_memories',
  description: 'Exportar memorias a JSON para respaldo (devuelve comando de respaldo)',
  inputSchema: BackupMemoriesSchema
};
```

---

## Instalación

### 1. Agregar al Servidor MCP

Agregar estas herramientas al índice principal de tu servidor MCP:

```typescript
// mcp-server/src/index.ts
import { get_database_stats_tool, get_database_stats } from './tools/db_management/get_database_stats';
import { get_pheromone_distribution_tool, get_pheromone_distribution } from './tools/db_management/get_pheromone_distribution';
import { analyze_memory_quality_tool, analyze_memory_quality } from './tools/db_management/analyze_memory_quality';
import { cleanup_low_quality_memories_tool, cleanup_low_quality_memories } from './tools/db_management/cleanup_low_quality_memories';
import { check_membox_status_tool, check_membox_status } from './tools/db_management/check_membox_status';
import { optimize_database_tool, optimize_database } from './tools/db_management/optimize_database';

// Registrar herramientas
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    get_database_stats_tool,
    get_pheromone_distribution_tool,
    analyze_memory_quality_tool,
    cleanup_low_quality_memories_tool,
    rebuild_embeddings_tool,
    check_membox_status_tool,
    optimize_database_tool,
    backup_memories_tool,
    // ... otras herramientas
  ]
}));

// Manejar llamadas de herramientas
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case 'get_database_stats':
      return { content: [{ type: 'text', text: await get_database_stats(args, pool) }] };
    case 'get_pheromone_distribution':
      return { content: [{ type: 'text', text: await get_pheromone_distribution(args, pool) }] };
    case 'analyze_memory_quality':
      return { content: [{ type: 'text', text: await analyze_memory_quality(pool) }] };
    case 'cleanup_low_quality_memories':
      return { content: [{ type: 'text', text: await cleanup_low_quality_memories(args, pool) }] };
    case 'check_membox_status':
      return { content: [{ type: 'text', text: await check_membox_status(pool) }] };
    case 'optimize_database':
      return { content: [{ type: 'text', text: await optimize_database(args, pool) }] };
    // ... otras herramientas
  }
});
```

### 2. Crear Archivos de Herramientas

Crear estructura de directorio:

```bash
mcp-server/
├── src/
│   ├── tools/
│   │   ├── db_management/
│   │   │   ├── get_database_stats.ts
│   │   │   ├── get_pheromone_distribution.ts
│   │   │   ├── analyze_memory_quality.ts
│   │   │   ├── cleanup_low_quality_memories.ts
│   │   │   ├── check_membox_status.ts
│   │   │   ├── optimize_database.ts
│   │   │   └── backup_memories.ts
```

---

## Ejemplos de Uso

### Verificación de Salud Diaria

```typescript
// Verificar salud general
get_database_stats(include_details=true)

// Verificar distribución de feromonas
get_pheromone_distribution(memory_type='all')

// Analizar calidad
analyze_memory_quality()

// Verificar si membox funciona
check_membox_status()
```

### Mantenimiento Semanal

```typescript
// Limpiar memorias de baja calidad (primero ejecución de prueba)
cleanup_low_quality_memories(min_pheromone=3.0, min_age_days=90, dry_run=true)

// Si está satisfecho, eliminar realmente
cleanup_low_quality_memories(min_pheromone=3.0, min_age_days=90, dry_run=false)

// Optimizar base de datos
optimize_database(full_vacuum=false, reindex=false)
```

### Limpieza Profunda Mensual

```typescript
// Reconstruir incrustaciones faltantes
rebuild_embeddings(memory_type='all', limit=500)

// Vacuum completo (requiere bloqueo exclusivo - ejecutar en horas no pico)
optimize_database(full_vacuum=true, reindex=true)

// Respaldar todo
backup_memories(output_path='/backups/mempheromone_monthly.json')
```

---

## Resultados Esperados

Después de implementar estas herramientas, deberías tener:

✅ **Monitoreo**: Métricas de salud de base de datos en tiempo real
✅ **Control de Calidad**: Identificar y corregir problemas de calidad
✅ **Mantenimiento**: Limpieza y optimización automatizadas
✅ **Validación de Membox**: Verificar que el sistema membox funciona
✅ **Respaldo/Restauración**: Protección de datos

---

## Próximos Pasos

1. Implementar herramientas en TypeScript
2. Agregar al registro de herramientas del servidor MCP
3. Probar cada herramienta individualmente
4. Crear panel de monitoreo automatizado
5. Configurar cron jobs para mantenimiento semanal

---

**Esto es parte del conjunto de herramientas principales de mempheromone y debe incluirse en el repositorio público.**
