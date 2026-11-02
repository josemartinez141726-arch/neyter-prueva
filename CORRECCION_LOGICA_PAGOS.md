# Corrección de Lógica - Pago Total del Ciclo y Total Pagado en Refinanciamientos

## Problema Identificado
La lógica anterior intentaba usar timestamps (`created_at`) para delimitar qué movimientos pertenecían a cada refinanciamiento, lo que causaba cálculos incorrectos porque:
- Los movimientos se registran en cualquier momento
- No hay garantía de orden cronológico
- Los refinanciamientos se crean después de los movimientos

## Solución Implementada

### 1. Nueva Columna en Base de Datos
**Script: `/scripts/013_add_numero_arreglo_to_movimientos.sql`**

Agrega columna `numero_arreglo` a `movimientos_historico`:
- Permite enlazar movimientos específicamente a refinanciamientos
- `null` = pertenece al crédito original
- `1, 2, 3...` = pertenece a ese refinanciamiento específico

### 2. Actualización de `registrarMovimiento`
**Archivo: `/lib/historial-sync.ts` línea 256**

Cambios:
- Acepta parámetro opcional `numeroArreglo`
- Si no se proporciona, auto-detecta el refinanciamiento activo más reciente
- Almacena `numero_arreglo` en cada movimiento registrado

```typescript
export const registrarMovimiento = async (
  clienteId: string,
  numeroCredito: number,
  tipoMovimiento: string,
  monto: number,
  descripcion?: string,
  numeroArreglo?: number  // NUEVO
) => { ... }
```

### 3. Función para Calcular Total del Refinanciamiento
**Archivo: `/lib/historial-sync.ts` línea 37**

`calcularTotalPagadoDesdeRefinanciamiento`:
- Filtra movimientos por `numero_arreglo` exacto
- Suma TODOS los movimientos de ese refinanciamiento específico
- Ejemplo: Refinanciamiento #2 con Cuota C$100 + Cuota C$100 + Pago único C$1,000 = C$1,200

### 4. Función para Calcular Pago Total del Ciclo
**Archivo: `/lib/historial-sync.ts` línea 8**

`calcularTotalPagado`:
- NO filtra por `numero_arreglo`
- Suma TODOS los movimientos del crédito completo
- Incluye movimientos de crédito original + todos los refinanciamientos
- Ejemplo: C$200 (original) + C$200 (Refin #1) + C$1,200 (Refin #2) = C$1,600

## Flujo de Datos

### Cuando se registra una cuota:
```
1. Admin paga cuota en Préstamos pendientes
2. registrarMovimiento se llama
3. Si hay Refinanciamiento activo → numero_arreglo = numero del refinanciamiento
4. Si no hay → numero_arreglo = null (crédito original)
5. Movimiento se almacena con numero_arreglo
```

### Cuando se actualiza un refinanciamiento como pagado:
```
1. actualizarCreditoPagado se llama
2. Calcula calcularTotalPagadoDesdeRefinanciamiento
3. Filtra movimientos por numero_arreglo del refinanciamiento
4. Suma todos esos movimientos
5. Actualiza refinanciamiento.total_pagado con el total
```

### Cuando se visualiza el crédito:
```
- "Pago total del ciclo" = suma de TODOS los movimientos (sin filtrar numero_arreglo)
- "Total pagado" (en Refinanciamientos) = suma de movimientos con numero_arreglo específico
```

## Ejemplo Práctico

Cliente recibe: C$1,200
- Crédito original: paga C$200
- Refinanciamiento #1: paga C$200
- Refinanciamiento #2: paga C$1,200 (C$100 + C$100 + C$1,000)

**Resultados:**
- Pago total del ciclo: C$1,600 ✓
- Total pagado Refin #2: C$1,200 ✓
- Total pagado Refin #1: C$200 ✓
- Total pagado Crédito original: C$200 ✓

## Archivos Modificados

1. `/lib/historial-sync.ts` - Funciones de cálculo
2. `/scripts/013_add_numero_arreglo_to_movimientos.sql` - Migración DB

## Pasos para Activar

1. Ejecutar script 013 en Supabase para agregar columna
2. El sistema automáticamente detectará refinanciamientos para nuevos movimientos
3. Para movimientos históricos existentes, ejecutar backfill si es necesario

## Validación

✅ Cada refinanciamiento calcula sus propios movimientos únicamente
✅ Pago total del ciclo suma todos los movimientos
✅ No hay mezcla de movimientos entre refinanciamientos
✅ Cálculos son automáticos y acurados
