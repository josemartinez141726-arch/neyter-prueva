# INSTRUCCIONES RÁPIDAS DE PRUEBA

## Verificación Rápida en 5 Minutos

### Requisitos:
- Sistema NEYTER en funcionamiento
- Acceso como Administrador
- Al menos 1 cliente con datos completos
- Navegador con consola de desarrollador abierta (F12)

---

## PRUEBA 1: Carga de Datos del Cliente

### Pasos:
1. Ir a **Administrador → Gestión de Préstamos**
2. Hacer clic en **"Crear Préstamo Cuotario"**
3. En el dropdown de cliente, seleccionar un cliente con datos completos
4. Llenar el formulario:
   - Monto: 1,000
   - Interés: 20%
   - Fecha entrega: hoy
   - Fecha corte: 30 días después
5. Hacer clic en **"Guardar"**

### Verificar en Consola (F12):
```
Buscar en consola: [v0] Cliente data loaded:
Debe mostrar: {
  id: "...",
  nombre_completo: "Jose Isaac Martinez",
  numero_cedula: "123456789",
  telefono: "58243212",
  comarca: "Las Carpas",
  municipio: "Telica",
  departamento: "León",
  ...
}
```

### ✅ Éxito si:
- Todos los campos aparecen completos (sin null/undefined)
- Los valores son los del cliente seleccionado
- No hay errores en la consola

---

## PRUEBA 2: Visualización en Contrato Digital

### Desde la prueba anterior:
- En el dialog de confirmación, verificar la sección "Cliente"

### ✅ Éxito si:
```
Nombre: Jose Isaac Martinez     (no N/A)
Cédula: 123456789              (no N/A)
Teléfono: 58243212             (no N/A)
Comarca: Las Carpas            (no N/A)
Municipio: Telica              (no N/A)
Departamento: León             (no N/A)
```

### ❌ Error si:
- Algún campo muestra N/A cuando tiene valor
- Los datos no coinciden con el cliente seleccionado

---

## PRUEBA 3: Botón WhatsApp

### Desde el dialog anterior:
1. Hacer clic en botón **"WhatsApp"**

### Verificar en Consola (F12):
```
Buscar: [v0] WhatsApp URL generated:
Debe mostrar: {
  telefonoRaw: "58243212",
  telefonoLimpio: "58243212",
  whatsappURL: "https://wa.me/50558243212?text=..."
}
```

### ✅ Éxito si:
- Se abre una pestaña con WhatsApp Web o app
- El chat muestra el número correcto
- El contrato aparece en el mensaje

### ❌ Error si:
- No se abre WhatsApp
- Dice "número inválido"
- URL tiene 505 duplicado

---

## PRUEBA 4: Teléfono con Formato

### Escenario: Cliente con teléfono "5824-3212"

### Pasos:
1. Crear préstamo seleccionando este cliente
2. En el dialog, hacer clic en **"WhatsApp"**

### Verificar en Consola:
```
[v0] WhatsApp URL generated: {
  telefonoRaw: "5824-3212",
  telefonoLimpio: "58243212",    (guion eliminado)
  whatsappURL: "https://wa.me/50558243212?text=..."
}
```

### ✅ Éxito si:
- Se limpia el guion automáticamente
- Se abre WhatsApp sin error
- URL es correcta

---

## PRUEBA 5: Teléfono Inválido

### Escenario: Cliente con teléfono "5824" (4 dígitos)

### Pasos:
1. Crear préstamo con este cliente
2. En el dialog, hacer clic en **"WhatsApp"**

### ✅ Éxito si:
- Aparece alerta: "Número de teléfono inválido. Se esperan 8 dígitos, se encontraron 4"
- No intenta abrir WhatsApp
- Se puede seguir con el proceso (guardar el préstamo)

### Verificar en Consola:
```
[v0] WhatsApp validation failed: {
  telefonoRaw: "5824",
  telefonoLimpio: "5824",
  length: 4
}
```

---

## PRUEBA 6: Sin Teléfono

### Escenario: Cliente sin teléfono registrado

### Pasos:
1. Crear préstamo con cliente sin teléfono
2. En el dialog, hacer clic en **"WhatsApp"**

### ✅ Éxito si:
- Aparece alerta: "El cliente no tiene teléfono registrado"
- No intenta abrir WhatsApp
- Botón "Continuar" sigue disponible

---

## PRUEBA 7: Guardar Préstamo

### Desde cualquier dialog de confirmación:
1. Hacer clic en **"Continuar"**
2. Ingresa contraseña del administrador
3. Hacer clic en **"OK"**

### ✅ Éxito si:
- Desaparece el dialog
- Vuelve a la pantalla de préstamos
- El nuevo préstamo aparece en la lista
- Datos en BD coinciden con lo ingresado

### Verificar en BD (Supabase):
```sql
SELECT * FROM prestamos 
WHERE tipo_prestamo = 'cuotario' 
AND cliente_id = '...'
ORDER BY created_at DESC LIMIT 1
```

---

## PRUEBA 8: Múltiples Clientes

### Pasos:
1. Crear préstamo con Cliente A (datos verificados)
2. Cancelar, volver atrás
3. Crear préstamo con Cliente B (datos diferentes)
4. Comparar datos en ambos dialogs

### ✅ Éxito si:
- Cliente A muestra SUS datos exactos
- Cliente B muestra SUS datos exactos
- Sin mezcla entre datos
- WhatsApp de cada uno es correcto

---

## Checklist de Validación

```
Verificar que se cumple lo siguiente:

☐ Los datos del cliente se cargan automáticamente
☐ El contrato digital muestra todos los datos
☐ No hay "N/A" en datos que existen
☐ WhatsApp usa el número correcto
☐ Teléfono con guiones se limpia correctamente
☐ Validación de 8 dígitos funciona
☐ Mensajes de error son claros
☐ Botón Continuar guarda el préstamo
☐ Los datos se persisten en BD
☐ Múltiples clientes no se mezclan
```

---

## Solución de Problemas Rápida

| Problema | Causa Probable | Solución |
|----------|----------------|----------|
| "Cliente no existe" | ID no coincide | Verificar que cliente está seleccionado |
| "Datos son N/A" | BD no tiene datos | Llenar datos del cliente en Usuarios |
| "WhatsApp no abre" | Número inválido | Verificar que teléfono tiene 8 dígitos |
| "Número inválido" | Formato no limpio | Datos guardados están mal (ej: "58 24 32 12") |
| "Error en consola" | Datos faltantes | Verificar que cliente tiene ID válido |

---

## Notas Importantes

- El sistema **automáticamente** carga los datos del cliente
- Los campos vacíos muestran "N/A" (esto es correcto)
- WhatsApp requiere el número completo sin código país
- La validación de 8 dígitos es OBLIGATORIA para Nicaragua
- Los logs en consola ayudan a debugging pero NO ralentizan
