# GUÍA RÁPIDA DE REFERENCIA - Solución de Informes

**Para cuando necesitas respuestas RÁPIDAS**

---

## 🔴 EL PROBLEMA EN 30 SEGUNDOS

```
Los logs muestran: "historial records=0" para TODOS los clientes

¿Por qué?
→ Tabla historial_prestamos tiene cliente_id = NULL

¿Consecuencia?
→ Arreglos de pago siempre = 0
→ Último crédito siempre = "Sin préstamos"
→ Último préstamo siempre = "Sin préstamos"
```

---

## 🟢 LA SOLUCIÓN EN 1 MINUTO

### 4 Pasos Simples

1. **Limpiar datos**
   ```sql
   UPDATE historial_prestamos h
   SET cliente_id = p.cliente_id
   WHERE h.cliente_id IS NULL AND h.prestamo_id = p.id;
   ```

2. **Validar en código**
   ```typescript
   if (!prestamoSeleccionado?.cliente_id) {
     throw new Error("cliente_id faltante");
   }
   ```

3. **Centralizar lógica**
   - Crear: `lib/contador-estadisticas.ts`
   - Usar en: informes + central (MISMO código)

4. **Sincronizar en tiempo real**
   - Crear: `lib/supabase-subscriptions.ts`
   - Implementar en: informes + central

---

## 📋 CHECKLIST RÁPIDO

### Pre-Implementación
- [ ] Backup de BD realizado
- [ ] Diagnóstico SQL ejecutado
- [ ] Supabase Real-time habilitado

### Durante Implementación
- [ ] FASE 0: Diagnóstico completado
- [ ] FASE 1: Preparación completada
- [ ] FASE 2: Correcciones completadas
- [ ] FASE 3: Sincronización completada
- [ ] FASE 4: Testing completado
- [ ] FASE 5: Validación completada

### Post-Implementación
- [ ] Hacer arreglo → contador incrementa
- [ ] Informes y Central sincronizados
- [ ] Sin errores en consola
- [ ] Tests 100% pasan

---

## 🎯 MÉTRICAS DE ÉXITO

| Métrica | Mínimo | Objetivo |
|---------|--------|----------|
| cliente_id NULL | 0 | 0 |
| Arreglos se cuentan | SÍ | SÍ |
| Sincronización (segundos) | < 5 | < 3 |
| Tests pasando | 100% | 100% |
| Errores en consola | 0 | 0 |

---

## 📁 ARCHIVOS A CREAR/MODIFICAR

### 📄 Crear (5 archivos)
```
scripts/010_backfill_historial_cliente_id.sql
lib/types/estadisticas.ts
lib/contador-estadisticas.ts ⭐
lib/supabase-subscriptions.ts ⭐
__tests__/contador-estadisticas.test.ts
```

### ✏️ Modificar (3 archivos)
```
components/admin/prestamos-pendientes.tsx
components/admin/informes-estadisticas.tsx
components/admin/central.tsx
```

---

## ⏱️ TIMELINE

| Fase | Horas |
|------|-------|
| 0: Diagnóstico | 0.25 |
| 1: Preparación | 1.5 |
| 2: Correcciones | 2.5 |
| 3: Sincronización | 1.5 |
| 4: Testing | 2.5 |
| 5: Entrega | 1 |
| **TOTAL** | **9** |

---

## 🔍 DIAGNÓSTICO RÁPIDO

### ¿Cliente_id NULL existe?
```sql
SELECT COUNT(*) FROM historial_prestamos WHERE cliente_id IS NULL;
-- Esperado: 0
-- Si > 0: Ejecutar backfill
```

### ¿Datos en historial?
```sql
SELECT * FROM historial_prestamos 
WHERE tipo = 'refinanciamiento'
LIMIT 5;
-- Debe mostrar registros CON cliente_id
```

### ¿Supabase Real-time?
```
Settings → Replication
✓ historial_prestamos: ON
✓ prestamos: ON
```

---

## 💾 CÓDIGO CRÍTICO

### 1. Validación en Insert (prestamos-pendientes.tsx)
```typescript
if (!prestamoSeleccionado?.cliente_id) {
  throw new Error("[v0] ERROR CRÍTICO: cliente_id faltante");
}

await supabase.from("historial_prestamos").insert([{
  cliente_id: prestamoSeleccionado.cliente_id, // ← CRÍTICO
  prestamo_id: prestamoSeleccionado.id,
  tipo: "refinanciamiento",
  // ...
}]);
```

### 2. Función Centralizada (lib/contador-estadisticas.ts)
```typescript
export function calcularEstadisticasCliente(
  cliente,
  prestamos,
  historial
) {
  return {
    totalPrestamos,
    prestamosActivos,
    prestamosPagados,
    diasRetraso,
    arreglosDePago,      // ← Conta refinanciamientos
    ultimoPrestamo,      // ← Mora del último PAGADO
    ultimoPrestamoArreglos, // ← Arreglos del último
  };
}
```

### 3. Usar en Informes (informes-estadisticas.tsx)
```typescript
import { calcularEstadisticasCliente } from "@/lib/contador-estadisticas";

const stats = calcularEstadisticasCliente(
  cliente,
  prestamosCliente,
  historialData
);

return {
  ...cliente,
  ...stats, // Expande todos los valores
};
```

### 4. Usar en Central (central.tsx)
```typescript
// EXACTAMENTE igual que informes
const stats = calcularEstadisticasCliente(
  clienteSeleccionado,
  prestamosData,
  historialData
);

setCreditInfo(stats);
```

---

## 🧪 TEST CRÍTICO

```typescript
// Crear arreglo → Debe aparecer en contador
const arreglos_antes = informes[0].arreglosDePago;
hacerArregloDePago();
await esperar(2000);
const arreglos_despues = cargarInformes()[0].arreglosDePago;

assert(arreglos_despues === arreglos_antes + 1);
✓ TEST PASÓ
```

---

## 🚨 PROBLEMAS COMUNES

### "Arreglos siguen siendo 0"
**Causa:** cliente_id NULL en historial_prestamos
**Solución:** Ejecutar backfill SQL (FASE 1.1)

### "No sincroniza en tiempo real"
**Causa:** Subscriptions no implementadas
**Solución:** Ver lib/supabase-subscriptions.ts (FASE 3)

### "Informes y Central muestran diferente"
**Causa:** Usando código diferente para calcular
**Solución:** Usar calcularEstadisticasCliente() en AMBOS (FASE 2)

### "Último crédito = null"
**Causa:** No hay préstamos pagados
**Solución:** Hacer arreglo + pagar crédito → Aparecerá

---

## 📞 ESCALACIÓN RÁPIDA

| Problema | Consulta | Pasos |
|----------|----------|-------|
| ¿Cuál es el problema? | ANALISIS_PROBLEMAS_CRITICOS.md | Lee sección "Problema Raíz" |
| ¿Cómo implemento? | ESTRATEGIA_IMPLEMENTACION_COMPLETA.md | Sigue FASE correspondiente |
| ¿Cómo valido? | PLAN_TESTING_EXHAUSTIVO.md | Ejecuta escenario correspondiente |
| ¿Estado general? | DOCUMENTO_ENTREGA_FINAL.md | Ve "Criterios de Aceptación" |

---

## 📊 ESTADO DE IMPLEMENTACIÓN

```
[ ] FASE 0: Diagnóstico
    └─ Verificar cliente_id NULL
    
[ ] FASE 1: Preparación
    ├─ Crear tipos en lib/types/estadisticas.ts
    ├─ Ejecutar backfill (si aplica)
    └─ Crear lib/contador-estadisticas.ts
    
[ ] FASE 2: Correcciones
    ├─ Validar cliente_id en prestamos-pendientes.tsx
    ├─ Usar función en informes-estadisticas.tsx
    └─ Usar función en central.tsx
    
[ ] FASE 3: Sincronización
    ├─ Crear lib/supabase-subscriptions.ts
    ├─ Implementar en informes-estadisticas.tsx
    └─ Implementar en central.tsx
    
[ ] FASE 4: Testing
    ├─ Tests automatizados
    └─ Tests manuales (10 escenarios)
    
[ ] FASE 5: Entrega
    ├─ Validación final
    ├─ Documentación
    └─ Confirmación de éxito
```

---

## 💡 PRO TIPS

1. **Siempre hacer backup antes de backfill SQL**
2. **Usar BD test primero, luego producción**
3. **Copiar código exactamente de ESTRATEGIA_IMPLEMENTACION_COMPLETA.md**
4. **No omitir FASE 4 (Testing) - es crítica**
5. **Si algo falla, consultar logs en browser console**

---

## 🎓 DOCUMENTACIÓN COMPLETA

Para información detallada, consulta:

- **Problema:** ANALISIS_PROBLEMAS_CRITICOS.md
- **Implementación:** ESTRATEGIA_IMPLEMENTACION_COMPLETA.md
- **Testing:** PLAN_TESTING_EXHAUSTIVO.md
- **General:** DOCUMENTO_ENTREGA_FINAL.md
- **Índice:** INDICE_MAESTRO_SOLUCION_COMPLETA.md

---

**Última actualización:** 2026-05-04
**Versión:** 1.0
**Estado:** Listo para implementación
