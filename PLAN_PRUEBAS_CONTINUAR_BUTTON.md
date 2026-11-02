# Plan de Pruebas: Flujo Completo del Botón "Continuar"

## Escenarios de Prueba

### Prueba 1: Flujo Exitoso Completo
**Precondición**: Admin autenticado, cliente válido seleccionado
```
1. Admin llena formulario cuotario completo
2. Hace clic en botón de guardar
3. Se abre dialog con preview del contrato
4. Admin hace clic en "Continuar"
```
**Resultado Esperado**:
- No aparece prompt de contraseña
- Préstamo se guarda en BD
- Se crea entrada en cuotas_programadas
- Se actualiza saldo de caja
- Se registra movimiento
- Alert: "Préstamo guardado correctamente"
- Dialog se cierra
- Formulario se resetea
- Lista de préstamos se recarga

**Validación**:
- ✅ BD: Verificar que prestamos tenga nuevo registro
- ✅ BD: Verificar que cuotas_programadas tenga X registros
- ✅ BD: Verificar que caja.total se decrementó
- ✅ BD: Verificar que movimientos tenga nuevo "prestamo"

---

### Prueba 2: Botón Cancelar Funciona
**Precondición**: Dialog abierto con preview
```
1. Admin hace clic en "Cancelar"
```
**Resultado Esperado**:
- Dialog se cierra sin guardar
- Formulario permanece con datos ingresados
- No hay cambios en BD

**Validación**:
- ✅ UI: Dialog cerrado
- ✅ BD: No hay nuevos registros
- ✅ Estado: previewData = null

---

### Prueba 3: Botón WhatsApp Abre Correctamente
**Precondición**: Dialog abierto, cliente con teléfono válido
```
1. Admin hace clic en "WhatsApp"
```
**Resultado Esperado**:
- Nueva ventana abre (wa.me/505XXXXXXXX)
- Mensaje precargado con contrato digital
- Se puede enviar al cliente

**Validación**:
- ✅ URL: https://wa.me/505XXXXXXXX?text=...
- ✅ Mensaje: Contiene todos los datos del crédito
- ✅ Cliente: Recibe mensaje completo

---

### Prueba 4: Validación de Teléfono Inválido
**Precondición**: Cliente con teléfono vacío o inválido
```
1. Admin hace clic en "WhatsApp"
```
**Resultado Esperado**:
- Alert: "El cliente no tiene teléfono registrado"
- O Alert: "Número de teléfono inválido..."
- WhatsApp no abre

**Validación**:
- ✅ Validación ejecutada
- ✅ Usuario informado claramente

---

### Prueba 5: Error en BD (Ej: Conexión perdida)
**Precondición**: Supabase no responde
```
1. Admin hace clic en "Continuar"
2. guardarPrestamo intenta guardar
3. Supabase retorna error
```
**Resultado Esperado**:
- Alert: "Error al guardar el préstamo: [error message]"
- Dialog permanece abierto
- Admin puede reintentar
- Logs: `[v0] Error guardando préstamo: [error]`

**Validación**:
- ✅ BD: No hay registros incompletos
- ✅ UI: Usuario informado
- ✅ Logs: Error capturado

---

### Prueba 6: Redondeo de Decimales
**Precondición**: Monto que genera decimales infinitos
```
Ejemplo: 1000 / 3 = 333.3333...
1. Admin crea crédito con monto 1000, 3 cuotas
2. Hace clic en "Continuar"
3. Sistema redondea automáticamente
```
**Resultado Esperado**:
- BD: monto_cuota = 333.33 (exactamente 2 decimales)
- Movimiento: "C$ 333.33"
- WhatsApp: "C$ 333.33"

**Validación**:
- ✅ DB Query: SELECT * FROM cuotas_programadas WHERE prestamo_id = X
- ✅ Verificar: monto_cuota = 333.33

---

### Prueba 7: Validaciones Previas (No llega al Dialog)
**Precondición**: Datos incompletos
```
Escenarios:
a) Cliente no seleccionado
b) Monto vacío
c) Interés vacío
d) Fechas inválidas
e) Cliente bloqueado
f) Frecuencia de cuota vacía (cuotario)
```
**Resultado Esperado**:
- Alert descriptivo
- Dialog NO se abre
- Formulario permanece visible

**Validación**:
- ✅ Cada validación funciona
- ✅ Mensajes son claros

---

## Lista de Verificación Final

### Funcionalidad
- [ ] Botón "Cancelar" cierra dialog sin guardar
- [ ] Botón "WhatsApp" abre chat con mensaje
- [ ] Botón "Continuar" guarda crédito sin pedir contraseña
- [ ] No hay prompt() de contraseña
- [ ] Todos los 3 botones visibles y funcionales

### Integridad de Datos
- [ ] Montos redondeados a 2 decimales en BD
- [ ] Movimientos registran montos correctamente
- [ ] Caja se actualiza correctamente
- [ ] Cuotas programadas se crean (si cuotario)

### Experiencia de Usuario
- [ ] Dialog se cierra después de guardar
- [ ] Formulario se resetea
- [ ] Lista de préstamos se recarga
- [ ] Mensajes de alerta son claros

### Robustez
- [ ] Manejo de errores de BD
- [ ] Validaciones de teléfono
- [ ] Logs de debug en consola
- [ ] No hay datos incompletos en caso de error

---

## Comando SQL para Verificación

```sql
-- Ver últimos préstamos cuotarios creados
SELECT id, cliente_id, tipo_prestamo, monto_prestado, total_pagar, created_at 
FROM prestamos 
WHERE tipo_prestamo = 'cuotario' 
ORDER BY created_at DESC 
LIMIT 5;

-- Ver cuotas programadas del último préstamo
SELECT numero_cuota, fecha_programada, monto_cuota 
FROM cuotas_programadas 
WHERE prestamo_id = (SELECT id FROM prestamos WHERE tipo_prestamo = 'cuotario' ORDER BY created_at DESC LIMIT 1)
ORDER BY numero_cuota;

-- Ver movimiento de creación
SELECT tipo, monto, descripcion, created_at 
FROM movimientos 
WHERE tipo = 'prestamo' 
ORDER BY created_at DESC 
LIMIT 1;

-- Verificar saldo de caja
SELECT total FROM caja;
```
