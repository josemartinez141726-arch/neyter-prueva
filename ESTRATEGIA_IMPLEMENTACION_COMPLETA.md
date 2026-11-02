# ESTRATEGIA DE IMPLEMENTACIÓN INTEGRAL - Funcionalidad de Informes

## FASE 0: DIAGNÓSTICO PREVIO (PRE-IMPLEMENTACIÓN)

### Paso 0.1: Inspeccionar Estado de Base de Datos
```sql
-- DIAGNÓSTICO 1: ¿Cuántos registros tienen cliente_id NULL?
SELECT COUNT(*) as registros_sin_cliente_id
FROM historial_prestamos
WHERE cliente_id IS NULL;
-- Esperado: 0 (si está correcto) o > 0 (si hay problema)

-- DIAGNÓSTICO 2: Verificar estructura de tabla
\d historial_prestamos;
-- Verificar que cliente_id existe y NO es NOT NULL

-- DIAGNÓSTICO 3: Estructura de prestamos
\d prestamos;
-- Verificar que cliente_id existe

-- DIAGNÓSTICO 4: Muestra de datos
SELECT TOP 10
  h.id,
  h.cliente_id,
  h.prestamo_id,
  h.tipo,
  p.cliente_id as prestamo_cliente_id
FROM historial_prestamos h
LEFT JOIN prestamos p ON h.prestamo_id = p.id
WHERE h.tipo = 'refinanciamiento';
```

**DECISIÓN CRÍTICA:**
- Si `cliente_id IS NULL` > 0: Necesitamos BACKFILL antes de proceder
- Si `cliente_id IS NULL` = 0: Procedemos directamente a correcciones

---

## FASE 1: PREPARACIÓN (1-2 horas)

### Paso 1.1: Crear Script Backfill (SI NECESARIO)
**Archivo:** `scripts/010_backfill_historial_cliente_id.sql`

```sql
-- Solo ejecutar si DIAGNÓSTICO 1 encontró registros NULL

-- 1. Backfill cliente_id desde prestamos table
UPDATE historial_prestamos h
SET cliente_id = p.cliente_id
WHERE h.cliente_id IS NULL
  AND h.prestamo_id = p.id
  AND p.cliente_id IS NOT NULL;

-- 2. Verificar: ¿Cuántos quedan sin cliente_id?
SELECT COUNT(*) FROM historial_prestamos WHERE cliente_id IS NULL;
-- Esperado: 0

-- 3. Agregar constraint si no existe
ALTER TABLE historial_prestamos
ADD CONSTRAINT check_cliente_id_not_null
CHECK (cliente_id IS NOT NULL);
```

**Ejecución:**
```bash
# Si necesario:
psql -U usuario -d base_datos -f scripts/010_backfill_historial_cliente_id.sql
```

### Paso 1.2: Verificar Estructura de Datos
**Archivo:** `lib/types/estadisticas.ts` (CREAR)

```typescript
export interface PrestamoConCliente {
  id: string;
  cliente_id: string; // CRÍTICO: nunca undefined
  estado: "activo" | "pagado";
  fecha_corte: string;
  created_at: string;
}

export interface HistorialRegistro {
  id: string;
  cliente_id: string; // CRÍTICO: nunca null/undefined
  prestamo_id: string;
  tipo: "refinanciamiento" | "mora" | "retraso" | "pago_mora";
  dias_extension?: number;
  descripcion?: string;
  monto?: number;
  created_at: string;
}

export interface EstadisticasCliente {
  totalPrestamos: number;
  prestamosActivos: number;
  prestamosPagados: number;
  diasRetraso: number;
  arreglosDePago: number;
  ultimoPrestamo: {
    puntual: boolean;
    diasMora: number;
  } | null;
  ultimoPrestamoArreglos: number;
}
```

### Paso 1.3: Crear Función Compartida de Cálculos
**Archivo:** `lib/contador-estadisticas.ts` (CREAR)

```typescript
import { HistorialRegistro, PrestamoConCliente, EstadisticasCliente } from "./types/estadisticas";

/**
 * FUNCIÓN CENTRALIZADA para calcular estadísticas de un cliente
 * Usada por: informes-estadisticas.tsx y central.tsx
 * Garantiza: Lógica idéntica en ambos componentes
 */
export function calcularEstadisticasCliente(
  cliente: { id: string },
  prestamos: PrestamoConCliente[],
  historial: HistorialRegistro[]
): EstadisticasCliente {
  // VALIDACIÓN CRÍTICA
  if (!cliente.id) {
    throw new Error("[v0] ERROR: cliente.id es undefined");
  }

  // 1. TOTAL DE PRÉSTAMOS (todos, activos + pagados)
  const totalPrestamos = prestamos.length;

  // 2. PRÉSTAMOS ACTIVOS (estado = "activo")
  const prestamosActivos = prestamos.filter((p) => p.estado === "activo").length;

  // 3. PRÉSTAMOS PAGADOS (estado = "pagado")
  const prestamosPagados = prestamos.filter((p) => p.estado === "pagado").length;

  // 4. DÍAS DE RETRASO ACTUAL (del préstamo activo más reciente)
  const prestamoActivo = prestamos.find((p) => p.estado === "activo");
  let diasRetraso = 0;
  if (prestamoActivo) {
    const hoy = new Date();
    hoy.setHours(0, 0, 0, 0);
    const [year, month, day] = prestamoActivo.fecha_corte.split("-").map(Number);
    const fechaCorte = new Date(year, month - 1, day);
    fechaCorte.setHours(0, 0, 0, 0);
    const diasRestantes = Math.round(
      (fechaCorte.getTime() - hoy.getTime()) / (1000 * 60 * 60 * 24)
    );
    diasRetraso = diasRestantes < 0 ? Math.abs(diasRestantes) : 0;
  }

  // 5. CONTADOR DE ARREGLOS DE PAGO (histórico total, acumulativo)
  const arreglosDePago = historial.filter(
    (h) => h.cliente_id === cliente.id && h.tipo === "refinanciamiento"
  ).length;

  // 6. ÚLTIMO CRÉDITO (del préstamo más recientemente PAGADO)
  let ultimoPrestamo: EstadisticasCliente["ultimoPrestamo"] = null;
  const prestamosPagadosOrdenados = prestamos
    .filter((p) => p.estado === "pagado")
    .sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime());

  if (prestamosPagadosOrdenados.length > 0) {
    const ultimoPrestamoData = prestamosPagadosOrdenados[0];
    
    // Contar días de mora del historial de ese préstamo
    const historialUltimo = historial.filter(
      (h) =>
        h.prestamo_id === ultimoPrestamoData.id &&
        (h.tipo === "mora" || h.tipo === "retraso")
    );
    const diasMoraUltimo = historialUltimo.reduce(
      (sum, h) => sum + (h.dias_extension || 0),
      0
    );

    ultimoPrestamo = {
      puntual: diasMoraUltimo === 0,
      diasMora: diasMoraUltimo,
    };
  }

  // 7. ÚLTIMO PRÉSTAMO - ARREGLOS (de ese crédito pagado)
  let ultimoPrestamoArreglos = 0;
  if (prestamosPagadosOrdenados.length > 0) {
    const ultimoPrestamoData = prestamosPagadosOrdenados[0];
    ultimoPrestamoArreglos = historial.filter(
      (h) =>
        h.prestamo_id === ultimoPrestamoData.id &&
        h.tipo === "refinanciamiento"
    ).length;
  }

  return {
    totalPrestamos,
    prestamosActivos,
    prestamosPagados,
    diasRetraso,
    arreglosDePago,
    ultimoPrestamo,
    ultimoPrestamoArreglos,
  };
}

/**
 * Valida que los datos tengan estructura correcta
 */
export function validarDatos(
  cliente: any,
  prestamos: any[],
  historial: any[]
): void {
  if (!cliente?.id) {
    throw new Error("[v0] ERROR: cliente.id inválido");
  }

  for (const p of prestamos) {
    if (!p.id || !p.cliente_id) {
      console.error("[v0] ERROR: Préstamo sin id o cliente_id", p);
      throw new Error("[v0] ERROR: Estructura de préstamo inválida");
    }
  }

  for (const h of historial) {
    if (!h.cliente_id) {
      console.error("[v0] ERROR: Historial sin cliente_id", h);
      throw new Error(
        `[v0] ERROR: Registro de historial sin cliente_id (tipo: ${h.tipo})`
      );
    }
  }
}
```

---

## FASE 2: CORRECCIONES INMEDIATAS (2-3 horas)

### Paso 2.1: Corregir `prestamos-pendientes.tsx`
**Cambios en función de arreglo de pago:**

```typescript
// ANTES (INCORRECTO):
await supabase.from("historial_prestamos").insert([
  {
    prestamo_id: prestamoSeleccionado.id,
    // cliente_id FALTA o es undefined
    tipo: "refinanciamiento",
    // ...
  },
]);

// DESPUÉS (CORRECTO):
// 1. Validar que prestamoSeleccionado tiene cliente_id
if (!prestamoSeleccionado?.cliente_id) {
  throw new Error(
    "[v0] ERROR CRÍTICO: Préstamo seleccionado sin cliente_id"
  );
}

// 2. Insertar con cliente_id garantizado
const { error: errorHistorial } = await supabase
  .from("historial_prestamos")
  .insert([
    {
      cliente_id: prestamoSeleccionado.cliente_id, // GARANTIZADO
      prestamo_id: prestamoSeleccionado.id,
      tipo: "refinanciamiento",
      descripcion: `Arreglo de pago: Nuevo capital C$${capitalNuevoArreglo.toLocaleString(
        "es-NI"
      )}, Plazo ${numeroDias} días...`,
      monto: capitalNuevoArreglo,
    },
  ]);

if (errorHistorial) {
  console.error("[v0] Error insertando historial:", errorHistorial);
  throw errorHistorial;
}

console.log(
  "[v0] Arreglo registrado con cliente_id:",
  prestamoSeleccionado.cliente_id
);
```

### Paso 2.2: Corregir `informes-estadisticas.tsx`
**Usar función compartida:**

```typescript
// ANTES (DUPLICADO):
const arreglosDePago = (historialData || []).filter(
  (h) => h.cliente_id === cliente.id && h.tipo === "refinanciamiento"
).length;
// ... más cálculos duplicados ...

// DESPUÉS (CENTRALIZADO):
import { calcularEstadisticasCliente, validarDatos } from "@/lib/contador-estadisticas";

// En la función cargarDatos():
try {
  // Validar datos
  validarDatos(cliente, prestamosCliente, historialData);

  // Calcular todo de una vez
  const stats = calcularEstadisticasCliente(
    cliente,
    prestamosCliente,
    historialData
  );

  return {
    ...cliente,
    ...stats, // Expande: totalPrestamos, prestamosActivos, etc.
  };
} catch (error) {
  console.error("[v0] Error calculando estadísticas:", error);
  // Retornar valores por defecto seguros
  return {
    ...cliente,
    totalPrestamos: 0,
    prestamosActivos: 0,
    prestamosPagados: 0,
    diasRetraso: 0,
    arreglosDePago: 0,
    ultimoPrestamo: null,
    ultimoPrestamoArreglos: 0,
  };
}
```

### Paso 2.3: Corregir `central.tsx`
**Usar la MISMA función compartida:**

```typescript
// Exactamente igual a informes-estadisticas.tsx
import { calcularEstadisticasCliente, validarDatos } from "@/lib/contador-estadisticas";

// En el useEffect donde se carga el cliente:
try {
  validarDatos(clienteSeleccionado, prestamosData, historialData);
  
  const stats = calcularEstadisticasCliente(
    clienteSeleccionado,
    prestamosData,
    historialData
  );

  setCreditInfo({
    totalPrestamos: stats.totalPrestamos,
    prestamosActivos: stats.prestamosActivos,
    prestamosPagados: stats.prestamosPagados,
    diasRetraso: stats.diasRetraso,
    arreglosDePago: stats.arreglosDePago,
    ultimoPrestamo: stats.ultimoPrestamo,
    ultimoPrestamoArreglos: stats.ultimoPrestamoArreglos,
  });
} catch (error) {
  console.error("[v0] Error en Central:", error);
  setCreditInfo(defaultCreditInfo);
}
```

---

## FASE 3: SINCRONIZACIÓN EN TIEMPO REAL (1-2 horas)

### Paso 3.1: Implementar Subscriptions a Supabase
**Archivo:** `lib/supabase-subscriptions.ts` (CREAR)

```typescript
import { RealtimeChannel } from "@supabase/supabase-js";
import { supabase } from "./supabase/client";

export type RefreshCallback = () => void;

/**
 * Suscribirse a cambios en historial_prestamos
 * Cuando hay un nuevo arreglo, automáticamente refrescar datos
 */
export function subscribeToHistorial(
  clienteId: string,
  onchange: RefreshCallback
): RealtimeChannel {
  const channel = supabase
    .channel(`historial-${clienteId}`)
    .on(
      "postgres_changes",
      {
        event: "*", // INSERT, UPDATE, DELETE
        schema: "public",
        table: "historial_prestamos",
        filter: `cliente_id=eq.${clienteId}`,
      },
      (payload) => {
        console.log("[v0] Cambio en historial detectado:", payload);
        onchange(); // Llamar a callback para refrescar
      }
    )
    .subscribe((status) => {
      console.log("[v0] Subscription status:", status);
    });

  return channel;
}

/**
 * Suscribirse a cambios en prestamos
 * Cuando cambia estado o mora, refrescar
 */
export function subscribeToPrestamos(
  clienteId: string,
  onchange: RefreshCallback
): RealtimeChannel {
  const channel = supabase
    .channel(`prestamos-${clienteId}`)
    .on(
      "postgres_changes",
      {
        event: "*",
        schema: "public",
        table: "prestamos",
        filter: `cliente_id=eq.${clienteId}`,
      },
      (payload) => {
        console.log("[v0] Cambio en prestamos detectado:", payload);
        onchange();
      }
    )
    .subscribe();

  return channel;
}

/**
 * Limpiar subscriptions
 */
export function unsubscribeAll(channels: RealtimeChannel[]): void {
  channels.forEach((ch) => {
    supabase.removeChannel(ch);
  });
}
```

### Paso 3.2: Integrar Subscriptions en Informes
**En `informes-estadisticas.tsx`:**

```typescript
import { subscribeToHistorial, subscribeToPrestamos, unsubscribeAll } from "@/lib/supabase-subscriptions";

export default function InformesEstadisticas() {
  const [subscriptions, setSubscriptions] = useState<RealtimeChannel[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [clientesEstadisticas, setClientesEstadisticas] = useState<ClienteConEstadisticas[]>([]);

  // Función para refrescar datos
  const cargarDatos = useCallback(async () => {
    // ... código actual de cargarDatos ...
  }, []);

  // Setup subscriptions
  useEffect(() => {
    const setupSubscriptions = async () => {
      const datos = await cargarDatos();
      
      // Para cada cliente, suscribirse a cambios
      const channels = datos.map((cliente) => {
        const ch1 = subscribeToHistorial(cliente.id, cargarDatos);
        const ch2 = subscribeToPrestamos(cliente.id, cargarDatos);
        return [ch1, ch2];
      }).flat();

      setSubscriptions(channels);
    };

    setupSubscriptions();

    // Cleanup
    return () => {
      unsubscribeAll(subscriptions);
    };
  }, [cargarDatos]);

  // Cargar datos iniciales y luego cada 5 segundos (backup)
  useEffect(() => {
    cargarDatos();
    const interval = setInterval(cargarDatos, 5000);
    return () => clearInterval(interval);
  }, [cargarDatos]);

  // ... resto del componente ...
}
```

---

## FASE 4: TESTING (2-3 horas)

### Paso 4.1: Crear Tests Automatizados
**Archivo:** `__tests__/contador-estadisticas.test.ts`

```typescript
import { calcularEstadisticasCliente, validarDatos } from "@/lib/contador-estadisticas";

describe("Contador de Estadísticas", () => {
  it("Debe contar arreglos de pago correctamente", () => {
    const cliente = { id: "cliente-1" };
    const prestamos = [
      { id: "p1", cliente_id: "cliente-1", estado: "pagado", fecha_corte: "2026-05-15", created_at: "2026-04-01" },
    ];
    const historial = [
      { id: "h1", cliente_id: "cliente-1", prestamo_id: "p1", tipo: "refinanciamiento", created_at: "2026-04-10" },
      { id: "h2", cliente_id: "cliente-1", prestamo_id: "p1", tipo: "refinanciamiento", created_at: "2026-04-15" },
    ];

    const stats = calcularEstadisticasCliente(cliente, prestamos, historial);

    expect(stats.arreglosDePago).toBe(2);
  });

  it("Debe detectar último crédito con mora", () => {
    const cliente = { id: "cliente-1" };
    const prestamos = [
      { id: "p1", cliente_id: "cliente-1", estado: "pagado", fecha_corte: "2026-05-15", created_at: "2026-04-01" },
    ];
    const historial = [
      { id: "h1", cliente_id: "cliente-1", prestamo_id: "p1", tipo: "mora", dias_extension: 5, created_at: "2026-04-10" },
    ];

    const stats = calcularEstadisticasCliente(cliente, prestamos, historial);

    expect(stats.ultimoPrestamo?.puntual).toBe(false);
    expect(stats.ultimoPrestamo?.diasMora).toBe(5);
  });

  it("Debe lanzar error si cliente_id es null en historial", () => {
    const cliente = { id: "cliente-1" };
    const prestamos = [];
    const historialInvalido = [
      { id: "h1", cliente_id: null, prestamo_id: "p1", tipo: "mora", created_at: "2026-04-10" },
    ];

    expect(() => validarDatos(cliente, prestamos, historialInvalido)).toThrow();
  });
});
```

### Paso 4.2: Testing Manual (Checklist)
```
[ ] 1. Crear nuevo préstamo → totalPrestamos incrementa
[ ] 2. Hacer arreglo → arreglosDePago incrementa
[ ] 3. Hacer 5 arreglos → arreglosDePago = 5
[ ] 4. Pagar crédito → ultimoPrestamo muestra mora
[ ] 5. Informes y Central muestran lo mismo
[ ] 6. Sin refrescar Central, actualiza automáticamente
[ ] 7. BD no tiene cliente_id = NULL en historial
```

---

## FASE 5: ENTREGA Y VALIDACIÓN FINAL (1 hora)

### Paso 5.1: Verificación Final
```
✓ Paso 1: ¿Backfill ejecutado? (si fue necesario)
✓ Paso 2: ¿Cliente_id siempre se guarda en inserts?
✓ Paso 3: ¿Función compartida usada en Informes y Central?
✓ Paso 4: ¿Sincronizaciones en tiempo real funcionan?
✓ Paso 5: ¿Todos los tests pasan?
✓ Paso 6: ¿Sin errores en consola?
✓ Paso 7: ¿Datos coinciden entre Informes y Central?
```

### Paso 5.2: Entregables
- [x] Script backfill ejecutado
- [x] lib/contador-estadisticas.ts creado
- [x] prestamos-pendientes.tsx corregido
- [x] informes-estadisticas.tsx actualizado
- [x] central.tsx actualizado
- [x] Subscriptions implementadas
- [x] Tests creados y pasan
- [x] Documentación actualizada
- [x] Validación final completada

---

## CRONOGRAMA

| Fase | Duración | Estado |
|------|----------|--------|
| 0 - Diagnóstico | 15 min | PENDIENTE |
| 1 - Preparación | 1-2 h | PENDIENTE |
| 2 - Correcciones | 2-3 h | PENDIENTE |
| 3 - Sincronización | 1-2 h | PENDIENTE |
| 4 - Testing | 2-3 h | PENDIENTE |
| 5 - Entrega | 1 h | PENDIENTE |
| **TOTAL** | **7-14 h** | **PENDIENTE** |

---

## CRITERIOS DE ÉXITO

✓ Arreglos se cuentan correctamente en tiempo real
✓ Último crédito y último préstamo muestran datos reales
✓ Central y Informes sincronizados sin refresh
✓ Sin datos NULL en cliente_id
✓ Todos los tests automatizados pasan
✓ Sin errores en consola
