# PLAN DE TESTING EXHAUSTIVO - Funcionalidad de Informes

## OBJETIVO
Garantizar que arreglos de pago, último crédito y último préstamo funcionen correctamente con datos en tiempo real sin errores.

---

## ESCENARIOS DE TESTING

### ESCENARIO 1: Cliente Sin Historial (Base)
**Setup:**
- Cliente nuevo SIN préstamos
- SIN arreglos de pago

**Pruebas:**
```
TEST 1.1: Mostrar valores por defecto
  VERIFICAR: Arreglos de pago = 0
  VERIFICAR: Último crédito = "Sin préstamos"
  VERIFICAR: Último préstamo = "Sin préstamos"
  RESULTADO: PASS/FAIL

TEST 1.2: Cargar Informes varias veces
  ACCIÓN: Recargar Informes 5 veces
  VERIFICAR: Valores no cambian
  VERIFICAR: Sin errores en consola
  RESULTADO: PASS/FAIL
```

---

### ESCENARIO 2: Primer Préstamo (Crédito Nuevo)
**Setup:**
- Cliente con 1 préstamo activo (recién creado)
- Sin arreglos, sin mora

**Pruebas:**
```
TEST 2.1: Actualizar contadores después de crear préstamo
  ACCIÓN: Crear nuevo préstamo
  ESPERAR: 2 segundos
  VERIFICAR: En Informes: totalPrestamos = 1
  VERIFICAR: En Informes: prestamosActivos = 1
  VERIFICAR: En Informes: prestamosPagados = 0
  VERIFICAR: En Central: Totales coinciden
  RESULTADO: PASS/FAIL

TEST 2.2: "Último crédito" sin mora
  VERIFICAR: No hay préstamos pagados
  VERIFICAR: Mostrar "Sin préstamos" (no muestra activo)
  RESULTADO: PASS/FAIL

TEST 2.3: "Último préstamo" sin arreglos
  VERIFICAR: Mostrar "Sin préstamos"
  RESULTADO: PASS/FAIL
```

---

### ESCENARIO 3: Préstamo con Días de Mora
**Setup:**
- Cliente con 1 préstamo activo
- Préstamo tiene 5 días de mora (vencimiento pasado)

**Pruebas:**
```
TEST 3.1: Mostrar mora actual del crédito activo
  VERIFICAR: diasRetraso = 5
  VERIFICAR: En Informes muestra el valor
  VERIFICAR: En Central muestra el valor
  RESULTADO: PASS/FAIL

TEST 3.2: Sincronización sin refresh
  ACCIÓN: Cambiar mora en BD directamente
  ESPERAR: 3 segundos para refresco automático
  VERIFICAR: Informes muestra valor nuevo
  VERIFICAR: Central muestra valor nuevo
  RESULTADO: PASS/FAIL
```

---

### ESCENARIO 4: Primer Arreglo de Pago
**Setup:**
- Cliente con 1 préstamo activo (con mora)
- Se va a hacer primer arreglo

**Pruebas:**
```
TEST 4.1: Contador incrementa después de arreglo
  ACCIÓN: Hacer arreglo de pago (botón rojo)
  ESPERAR: 2 segundos
  VERIFICAR: En Informes: arreglosDePago = 1
  VERIFICAR: En Central: arreglosDePago = 1
  VERIFICAR: Sin errores en consola
  RESULTADO: PASS/FAIL

TEST 4.2: Total préstamos incrementa
  VERIFICAR: totalPrestamos = 2 (antiguo + arreglo como nuevo)
  O: totalPrestamos sigue = 1 (si no es nuevo crédito)
  [ACLARAR: Regla de negocio - ¿arreglo crea nuevo crédito?]
  RESULTADO: PASS/FAIL

TEST 4.3: Prestamos activos sigue siendo 1
  VERIFICAR: prestamosActivos = 1 (siempre hay uno activo)
  RESULTADO: PASS/FAIL

TEST 4.4: Datos guardados en BD
  ACCIÓN: Consultar historial_prestamos directamente
  VERIFICAR: existe registro tipo="refinanciamiento"
  VERIFICAR: cliente_id NO es NULL
  VERIFICAR: prestamo_id correcto
  RESULTADO: PASS/FAIL

TEST 4.5: Sincronización automática en tiempo real
  ACCIÓN: Hacer arreglo
  VERIFICAR: Informes actualiza en < 5 segundos
  VERIFICAR: Central actualiza sin hacer click
  VERIFICAR: Ambos muestran el mismo valor
  RESULTADO: PASS/FAIL
```

---

### ESCENARIO 5: Múltiples Arreglos (2-5 arreglos)
**Setup:**
- Cliente con 1 préstamo que ya ha tenido 1 arreglo
- Va a hacer arreglos adicionales

**Pruebas:**
```
TEST 5.1: Contador suma correctamente cada arreglo
  ACCIÓN: Hacer arreglo #2
  ESPERAR: 2 segundos
  VERIFICAR: Informes: arreglosDePago = 2
  VERIFICAR: Central: arreglosDePago = 2
  RESULTADO: PASS/FAIL

TEST 5.2: Incremento continuo en 3+ arreglos
  ACCIÓN: Hacer arreglos #3, #4, #5
  ESPERAR: 2 segundos después de cada uno
  VERIFICAR: arreglosDePago incrementa: 3, 4, 5
  VERIFICAR: Sin saltos ni repeticiones
  RESULTADO: PASS/FAIL

TEST 5.3: Historial completo en BD
  ACCIÓN: Consultar historial_prestamos
  VERIFICAR: 5 registros tipo="refinanciamiento"
  VERIFICAR: TODOS tienen cliente_id válido
  VERIFICAR: Todos para el mismo prestamo_id
  RESULTADO: PASS/FAIL

TEST 5.4: Sincronización consistente
  ACCIÓN: Abrir Informes en pestaña A, Central en pestaña B
  ACCIÓN: Hacer arreglo en pestaña A
  ESPERAR: 3 segundos
  VERIFICAR: Pestaña B actualiza automáticamente sin click
  VERIFICAR: Ambas muestran mismo valor
  RESULTADO: PASS/FAIL
```

---

### ESCENARIO 6: Pago Completo del Crédito
**Setup:**
- Cliente con préstamo que ha tenido N arreglos
- Presionar botón PAGO (verde) para finalizar

**Pruebas:**
```
TEST 6.1: Crédito pasa a "pagado"
  ACCIÓN: Presionar botón PAGO
  ESPERAR: 2 segundos
  VERIFICAR: prestamo.estado = "pagado"
  VERIFICAR: prestamosActivos = 0 (o disminuye en 1)
  VERIFICAR: prestamosPagados = 1 (o incrementa)
  RESULTADO: PASS/FAIL

TEST 6.2: "Último crédito" muestra dato del préstamo completado
  VERIFICAR: Muestra días de mora que tuvo
  VERIFICAR: Muestra "Puntual" si mora = 0
  VERIFICAR: Muestra "X días de mora" si mora > 0
  RESULTADO: PASS/FAIL

TEST 6.3: "Último préstamo" cuenta arreglos del crédito terminado
  VERIFICAR: Muestra cantidad exacta de arreglos
  EJEMPLO: Si tuvo 3 arreglos → "3 arreglos de pago"
  RESULTADO: PASS/FAIL

TEST 6.4: Contador "Arreglos de pago" total sigue acumulando
  VERIFICAR: No se reinicia después de pagar
  VERIFICAR: Sigue siendo 5 (si hubiera 5 en total)
  RESULTADO: PASS/FAIL

TEST 6.5: Sincronización en pago completo
  ACCIÓN: Presionar PAGO
  VERIFICAR: Central actualiza sin click
  VERIFICAR: Informes actualiza automáticamente
  RESULTADO: PASS/FAIL
```

---

### ESCENARIO 7: Segundo Préstamo (Ciclo Repetido)
**Setup:**
- Cliente con 1 crédito completado
- Va a crear nuevo crédito

**Pruebas:**
```
TEST 7.1: Totales se actualizan correctamente
  ACCIÓN: Crear nuevo préstamo
  ESPERAR: 2 segundos
  VERIFICAR: totalPrestamos = 2
  VERIFICAR: prestamosActivos = 1
  VERIFICAR: prestamosPagados = 1
  RESULTADO: PASS/FAIL

TEST 7.2: "Último crédito" muestra dato del nuevo si se completa
  ACCIÓN: Hacer arreglo + Pagar nuevo crédito
  VERIFICAR: "Último crédito" muestra mora del nuevo crédito
  VERIFICAR: NO muestra datos del anterior
  RESULTADO: PASS/FAIL

TEST 7.3: "Último préstamo" actualiza al nuevo crédito
  VERIFICAR: Muestra arreglos del último crédito completado
  RESULTADO: PASS/FAIL

TEST 7.4: Contador "Arreglos de pago" total acumula
  VERIFICAR: Si anterior tenía 3 y nuevo 2 → total 5
  VERIFICAR: No se reinicia
  RESULTADO: PASS/FAIL
```

---

### ESCENARIO 8: Sincronización Multi-Pestaña
**Setup:**
- Informes abierto en pestaña A
- Central abierto en pestaña B (mismo cliente)

**Pruebas:**
```
TEST 8.1: Cambio en Informes refleja en Central
  ACCIÓN: Hacer arreglo desde pestaña A
  ESPERAR: 3 segundos
  VERIFICAR: Pestaña B actualiza automáticamente
  RESULTADO: PASS/FAIL

TEST 8.2: Cambio en Central refleja en Informes
  ACCIÓN: Pagar crédito desde pestaña B
  ESPERAR: 3 segundos
  VERIFICAR: Pestaña A muestra datos nuevos
  RESULTADO: PASS/FAIL

TEST 8.3: Sin inconsistencias entre pestañas
  ACCIÓN: Hacer 5 operaciones alternadas en A y B
  ESPERAR: 3 segundos después de cada una
  VERIFICAR: Nunca hay valores diferentes entre A y B
  RESULTADO: PASS/FAIL
```

---

### ESCENARIO 9: Datos NULL en Base de Datos (Diagnóstico)
**Setup:**
- Verificar estado de historial_prestamos en BD

**Pruebas:**
```
TEST 9.1: No hay registros con cliente_id = NULL
  ACCIÓN: Consultar directamente en BD
  SELECT COUNT(*) FROM historial_prestamos WHERE cliente_id IS NULL;
  VERIFICAR: Resultado = 0
  RESULTADO: PASS/FAIL

TEST 9.2: Todos los registros tienen cliente_id válido
  ACCIÓN: Consultar registros
  SELECT COUNT(*) FROM historial_prestamos WHERE cliente_id IS NOT NULL;
  VERIFICAR: Resultado = total de registros
  RESULTADO: PASS/FAIL

TEST 9.3: cliente_id coincide con prestamo.cliente_id
  ACCIÓN: Consultar y validar
  SELECT h.*, p.cliente_id FROM historial_prestamos h
    JOIN prestamos p ON h.prestamo_id = p.id
    WHERE h.cliente_id != p.cliente_id;
  VERIFICAR: Resultado = 0 (sin discrepancias)
  RESULTADO: PASS/FAIL
```

---

### ESCENARIO 10: Rendimiento y Estabilidad
**Setup:**
- Sistema bajo carga

**Pruebas:**
```
TEST 10.1: Sin lag en Informes (recarga automática)
  ACCIÓN: Dejar Informes abierto 5 minutos
  VERIFICAR: Refresco cada 5 segundos sin lag
  VERIFICAR: Sin errores en consola
  RESULTADO: PASS/FAIL

TEST 10.2: Sin memory leaks
  ACCIÓN: Abrir/cerrar Informes 10 veces
  VERIFICAR: Memoria estable
  VERIFICAR: Sin eventos duplicados
  RESULTADO: PASS/FAIL

TEST 10.3: Operaciones concurrentes
  ACCIÓN: Hacer 3 arreglos casi simultáneamente
  ESPERAR: 2 segundos
  VERIFICAR: Todos se guardan correctamente
  VERIFICAR: Contador = 3
  RESULTADO: PASS/FAIL
```

---

## MATRIZ DE PRUEBAS CRÍTICAS

| Test | Prioridad | Bloqueante |
|------|-----------|-----------|
| 4.1: Contador incrementa | CRÍTICA | SÍ |
| 4.4: cliente_id guardado | CRÍTICA | SÍ |
| 4.5: Sincronización tiempo real | CRÍTICA | SÍ |
| 5.4: Sincronización multi-pestaña | ALTA | SÍ |
| 6.2: Último crédito correcto | ALTA | SÍ |
| 6.3: Último préstamo correcto | ALTA | SÍ |
| 9.1 & 9.2: Sin NULL en cliente_id | CRÍTICA | SÍ |
| 10.3: Operaciones concurrentes | MEDIA | NO |

---

## PROCESO DE TESTING

### Fase 1: Testing Manual (Humano)
```
Para cada escenario:
  1. Preparar setup
  2. Ejecutar acciones paso a paso
  3. Verificar cada resultado inmediatamente
  4. Documentar PASS/FAIL
  5. Si FAIL: capturar logs y pantallazos
```

### Fase 2: Testing Automatizado (Código)
```
npm test -- --watch

Crear tests en:
  __tests__/informes-estadisticas.test.tsx
  __tests__/central.test.tsx
  __tests__/contador-prestamos.test.ts
```

### Fase 3: Testing de Integración
```
1. Supabase en vivo
2. Múltiples usuarios simultáneamente
3. Verificar replicación en tiempo real
```

---

## CRITERIOS DE ACEPTACIÓN

### PASAR todos los tests = ENTREGA
- ✓ 4.1, 4.4, 4.5: Arreglos funcionan
- ✓ 6.2, 6.3: Último crédito y préstamo correctos
- ✓ 5.4, 8.x: Sincronización multi-pestaña
- ✓ 9.1, 9.2, 9.3: Sin datos NULL
- ✓ 10.1: Sin lag en rendimiento

### NO PASAR ningún test crítico = RECHAZADO
- ✗ Si cliente_id es NULL → RECHAZADO
- ✗ Si contador no incrementa → RECHAZADO
- ✗ Si no sincroniza en tiempo real → RECHAZADO
- ✗ Si "Último crédito" incorrecto → RECHAZADO

---

## ENTREGABLES

1. Documento de resultados de testing
2. Logs de ejecución de tests
3. Screenshots de casos críticos
4. Código de tests automatizados
5. Script de backfill (si aplica)
6. Confirmación: "Todos los problemas resueltos"
