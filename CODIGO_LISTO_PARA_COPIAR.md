# CÓDIGO LISTO PARA COPIAR - Fragmentos Exactos para Implementación

**Para usar:** Copiar/pegar directamente, adaptando nombres de variables según necesites

---

## 1️⃣ lib/tipos/estadisticas.ts

```typescript
// Crear archivo nuevo con este contenido exacto

export interface PrestamoConCliente {
  id: string;
  cliente_id: string;
  estado: "activo" | "pagado";
  fecha_corte: string;
  created_at: string;
}

export interface HistorialRegistro {
  id: string;
  cliente_id: string;
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

---

## 2️⃣ lib/contador-estadisticas.ts

```typescript
// Crear archivo nuevo con este contenido exacto

import {
  HistorialRegistro,
  PrestamoConCliente,
  EstadisticasCliente,
} from "./types/estadisticas";

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

  // 1. TOTAL DE PRÉSTAMOS
  const totalPrestamos = prestamos.length;

  // 2. PRÉSTAMOS ACTIVOS
  const prestamosActivos = prestamos.filter((p) => p.estado === "activo").length;

  // 3. PRÉSTAMOS PAGADOS
  const prestamosPagados = prestamos.filter((p) => p.estado === "pagado").length;

  // 4. DÍAS DE RETRASO ACTUAL
  const prestamoActivo = prestamos.find((p) => p.estado === "activo");
  let diasRetraso = 0;
  if (prestamoActivo) {
    const hoy = new Date();
    hoy.setHours(0, 0, 0, 0);
    const [year, month, day] = prestamoActivo.fecha_corte
      .split("-")
      .map(Number);
    const fechaCorte = new Date(year, month - 1, day);
    fechaCorte.setHours(0, 0, 0, 0);
    const diasRestantes = Math.round(
      (fechaCorte.getTime() - hoy.getTime()) / (1000 * 60 * 60 * 24)
    );
    diasRetraso = diasRestantes < 0 ? Math.abs(diasRestantes) : 0;
  }

  // 5. CONTADOR DE ARREGLOS DE PAGO
  const arreglosDePago = historial.filter(
    (h) => h.cliente_id === cliente.id && h.tipo === "refinanciamiento"
  ).length;

  // 6. ÚLTIMO CRÉDITO
  let ultimoPrestamo: EstadisticasCliente["ultimoPrestamo"] = null;
  const prestamosPagadosOrdenados = prestamos
    .filter((p) => p.estado === "pagado")
    .sort(
      (a, b) =>
        new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
    );

  if (prestamosPagadosOrdenados.length > 0) {
    const ultimoPrestamoData = prestamosPagadosOrdenados[0];

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

  // 7. ÚLTIMO PRÉSTAMO - ARREGLOS
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

## 3️⃣ Modificación: prestamos-pendientes.tsx (Insert de Arreglo)

```typescript
// BUSCAR ESTA SECCIÓN en prestamos-pendientes.tsx (aprox línea 834)
// REEMPLAZAR con el código de abajo:

// VALIDACIÓN CRÍTICA: Asegurar que cliente_id existe
if (!prestamoSeleccionado?.cliente_id) {
  console.error("[v0] ERROR CRÍTICO: Préstamo sin cliente_id", prestamoSeleccionado);
  alert("Error: No se puede procesar el arreglo. Préstamo incompleto.");
  return;
}

// Insertar en historial con cliente_id garantizado
const { error: errorHistorial } = await supabase
  .from("historial_prestamos")
  .insert([
    {
      cliente_id: prestamoSeleccionado.cliente_id, // ← CRÍTICO: siempre incluir
      prestamo_id: prestamoSeleccionado.id,
      tipo: "refinanciamiento",
      descripcion: `Arreglo de pago: Nuevo capital C$${capitalNuevoArreglo.toLocaleString(
        "es-NI"
      )}, Plazo ${numeroDias} días, Total a pagar C$${totalPagarNuevo.toLocaleString(
        "es-NI"
      )}, Prenda: ${formArregloRefinanciamiento.descripcion_prenda}`,
      monto: capitalNuevoArreglo,
    },
  ]);

if (errorHistorial) {
  console.error("[v0] Error insertando en historial:", errorHistorial);
  alert("Error al guardar el arreglo: " + errorHistorial.message);
  return;
}

console.log(
  "[v0] Arreglo registrado exitosamente. cliente_id:",
  prestamoSeleccionado.cliente_id
);
```

---

## 4️⃣ Modificación: informes-estadisticas.tsx

```typescript
// AGREGAR IMPORT al inicio del archivo:
import {
  calcularEstadisticasCliente,
  validarDatos,
} from "@/lib/contador-estadisticas";

// REEMPLAZAR la función cargarDatos() con esto:
const cargarDatos = useCallback(async () => {
  try {
    setCargando(true);

    // Obtener clientes
    const { data: clientesData } = await supabase
      .from("clientes")
      .select("*")
      .order("nombre");

    if (!clientesData) {
      setClientes([]);
      return;
    }

    // Obtener préstamos
    const { data: prestamosData } = await supabase
      .from("prestamos")
      .select("*");

    // Obtener historial
    const { data: historialData } = await supabase
      .from("historial_prestamos")
      .select("*");

    // Calcular estadísticas para cada cliente
    const clientesConEstadisticas = clientesData.map((cliente) => {
      const prestamosCliente = (prestamosData || []).filter(
        (p) => p.cliente_id === cliente.id
      );

      try {
        // Validar datos
        validarDatos(cliente, prestamosCliente, historialData);

        // Usar función centralizada
        const stats = calcularEstadisticasCliente(
          cliente,
          prestamosCliente,
          historialData
        );

        return {
          ...cliente,
          ...stats,
        };
      } catch (error) {
        console.error("[v0] Error calculando estadísticas para cliente:", error);
        // Retornar con valores por defecto seguros
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
    });

    setClientes(clientesConEstadisticas);
  } catch (err) {
    console.error("Error cargando datos:", err);
  } finally {
    setCargando(false);
  }
}, []);
```

---

## 5️⃣ Modificación: central.tsx

```typescript
// AGREGAR IMPORT al inicio del archivo:
import {
  calcularEstadisticasCliente,
  validarDatos,
} from "@/lib/contador-estadisticas";

// BUSCAR el useEffect que carga datos del cliente y REEMPLAZAR con:
useEffect(() => {
  if (!clienteSeleccionado?.id) return;

  const cargarDatos = async () => {
    try {
      // Obtener préstamos del cliente
      const { data: prestamosData } = await supabase
        .from("prestamos")
        .select("*")
        .eq("cliente_id", clienteSeleccionado.id);

      // Obtener historial completo
      const { data: historialData } = await supabase
        .from("historial_prestamos")
        .select("*");

      try {
        // Validar datos
        validarDatos(
          clienteSeleccionado,
          prestamosData || [],
          historialData || []
        );

        // Usar EXACTAMENTE la misma función que informes
        const stats = calcularEstadisticasCliente(
          clienteSeleccionado,
          prestamosData || [],
          historialData || []
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
        console.error("[v0] Error calculando estadísticas en Central:", error);
        setCreditInfo({
          totalPrestamos: 0,
          prestamosActivos: 0,
          prestamosPagados: 0,
          diasRetraso: 0,
          arreglosDePago: 0,
          ultimoPrestamo: null,
          ultimoPrestamoArreglos: 0,
        });
      }
    } catch (err) {
      console.error("Error cargando datos del cliente:", err);
    }
  };

  cargarDatos();
}, [clienteSeleccionado?.id]);
```

---

## 6️⃣ lib/supabase-subscriptions.ts

```typescript
// Crear archivo nuevo con este contenido exacto

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
        event: "*",
        schema: "public",
        table: "historial_prestamos",
        filter: `cliente_id=eq.${clienteId}`,
      },
      (payload) => {
        console.log("[v0] Cambio en historial detectado:", payload.eventType);
        onchange();
      }
    )
    .subscribe((status) => {
      console.log("[v0] Subscription historial status:", status);
    });

  return channel;
}

/**
 * Suscribirse a cambios en prestamos
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
        console.log("[v0] Cambio en prestamos detectado:", payload.eventType);
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

---

## 7️⃣ Script SQL: Backfill

```sql
-- Ejecutar SOLO si SELECT COUNT(*) FROM historial_prestamos WHERE cliente_id IS NULL devuelve > 0

-- 1. Backfill cliente_id desde prestamos table
UPDATE historial_prestamos h
SET cliente_id = p.cliente_id
WHERE h.cliente_id IS NULL
  AND h.prestamo_id = p.id
  AND p.cliente_id IS NOT NULL;

-- 2. Verificar cuántos quedan sin cliente_id
SELECT COUNT(*) as registros_sin_cliente_id
FROM historial_prestamos
WHERE cliente_id IS NULL;
-- Esperado resultado: 0

-- 3. Agregar constraint para prevenir futuros NULL (opcional pero recomendado)
ALTER TABLE historial_prestamos
ADD CONSTRAINT check_cliente_id_not_null
CHECK (cliente_id IS NOT NULL);
```

---

## 8️⃣ Test: Validación Crítica

```typescript
// __tests__/contador-estadisticas.test.ts

import { calcularEstadisticasCliente } from "@/lib/contador-estadisticas";

describe("Contador de Estadísticas", () => {
  it("Debe contar arreglos de pago correctamente", () => {
    const cliente = { id: "cliente-1" };
    const prestamos = [
      {
        id: "p1",
        cliente_id: "cliente-1",
        estado: "pagado",
        fecha_corte: "2026-05-15",
        created_at: "2026-04-01",
      },
    ];
    const historial = [
      {
        id: "h1",
        cliente_id: "cliente-1",
        prestamo_id: "p1",
        tipo: "refinanciamiento",
        created_at: "2026-04-10",
      },
      {
        id: "h2",
        cliente_id: "cliente-1",
        prestamo_id: "p1",
        tipo: "refinanciamiento",
        created_at: "2026-04-15",
      },
    ];

    const stats = calcularEstadisticasCliente(cliente, prestamos, historial);

    expect(stats.arreglosDePago).toBe(2);
  });

  it("Debe detectar último crédito con mora", () => {
    const cliente = { id: "cliente-1" };
    const prestamos = [
      {
        id: "p1",
        cliente_id: "cliente-1",
        estado: "pagado",
        fecha_corte: "2026-05-15",
        created_at: "2026-04-01",
      },
    ];
    const historial = [
      {
        id: "h1",
        cliente_id: "cliente-1",
        prestamo_id: "p1",
        tipo: "mora",
        dias_extension: 5,
        created_at: "2026-04-10",
      },
    ];

    const stats = calcularEstadisticasCliente(cliente, prestamos, historial);

    expect(stats.ultimoPrestamo?.puntual).toBe(false);
    expect(stats.ultimoPrestamo?.diasMora).toBe(5);
  });
});
```

---

## 🎯 USO RÁPIDO

1. **lib/types/estadisticas.ts:** Copiar completo → crear archivo nuevo
2. **lib/contador-estadisticas.ts:** Copiar completo → crear archivo nuevo
3. **prestamos-pendientes.tsx:** Buscar línea ~834, reemplazar bloque completo
4. **informes-estadisticas.tsx:** Agregar import + reemplazar cargarDatos()
5. **central.tsx:** Agregar import + reemplazar useEffect
6. **lib/supabase-subscriptions.ts:** Copiar completo → crear archivo nuevo
7. **Script SQL:** Ejecutar solo si hay cliente_id NULL
8. **Tests:** Crear archivo nuevo y ejecutar

---

**Todos los códigos están listos para copiar/pegar. Adaptadores según nombres locales.**
