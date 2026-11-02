# Plan de Pruebas: Carga Automática de Datos del Perfil del Usuario

## Escenarios de Prueba

### Escenario 1: Cliente con todos los datos completos

**Precondiciones:**
- Cliente registrado en la base de datos con:
  - Nombre completo ✓
  - Cédula ✓
  - Teléfono (8 dígitos) ✓
  - Comarca ✓
  - Municipio ✓
  - Departamento ✓

**Pasos:**
1. Ir a Administrador → Gestión de Préstamos
2. Crear nuevo préstamo → Seleccionar tipo "Cuotario"
3. Seleccionar cliente del dropdown
4. Rellenar formulario (monto, fechas, interés)
5. Hacer clic en "Guardar"

**Resultados Esperados:**
- ✓ Se abre dialog de confirmación
- ✓ Contrato digital muestra TODOS los datos sin "N/A"
- ✓ Botón WhatsApp está habilitado
- ✓ Al hacer clic en WhatsApp:
  - Extrae teléfono correctamente
  - Limpia formato (quita espacios, guiones)
  - Genera URL: `https://wa.me/505XXXXXXXX`
  - Abre chat de WhatsApp

**Verificación en Consola:**
```
[v0] Cliente data loaded: {
  id: "...",
  nombre_completo: "...",
  numero_cedula: "...",
  telefono: "...",
  comarca: "...",
  municipio: "...",
  departamento: "...",
  ...
}

[v0] WhatsApp URL generated: {
  telefonoRaw: "58243212",
  telefonoLimpio: "58243212",
  whatsappURL: "https://wa.me/50558243212?text=..."
}
```

---

### Escenario 2: Cliente con datos parciales

**Precondiciones:**
- Cliente registrado con:
  - Nombre ✓
  - Cédula ✓
  - Teléfono ✓
  - Comarca: (vacío)
  - Municipio ✓
  - Departamento: (vacío)

**Pasos:**
1. Seguir mismo flujo que Escenario 1

**Resultados Esperados:**
- ✓ Contrato muestra:
  - Comarca: N/A
  - Departamento: N/A
- ✓ Otros campos muestran valores reales
- ✓ WhatsApp sigue funcionando
- ✓ Botón "Continuar" funciona normalmente

**Verificación:**
Los campos vacíos muestran "N/A" sin error, pero no impiden la operación.

---

### Escenario 3: Cliente sin teléfono

**Precondiciones:**
- Cliente registrado sin teléfono (campo vacío o nulo)

**Pasos:**
1. Seguir flujo hasta preview
2. Hacer clic en botón "WhatsApp"

**Resultados Esperados:**
- ✓ Se muestra alerta: "El cliente no tiene teléfono registrado"
- ✓ No intenta abrir WhatsApp
- ✓ Botón "Continuar" sigue disponible (no depende de WhatsApp)
- ✓ Se puede guardar el préstamo sin enviar WhatsApp

---

### Escenario 4: Teléfono con formato incorrecto

**Precondiciones:**
- Cliente con teléfono: "5824-32" (7 dígitos)

**Pasos:**
1. Hacer clic en botón WhatsApp

**Resultados Esperados:**
- ✓ Se muestra alerta: "Número de teléfono inválido. Se esperan 8 dígitos, se encontraron 7. Teléfono recibido: '5824-32'"
- ✓ No abre WhatsApp
- ✓ Se puede seguir con el proceso (guardar)

---

### Escenario 5: Teléfono con código país duplicado

**Precondiciones:**
- Cliente con teléfono: "505 58243212"

**Pasos:**
1. Hacer clic en botón WhatsApp

**Resultados Esperados:**
- ✓ Sistema limpia y detecta el 505
- ✓ Genera URL correcta: `https://wa.me/50558243212`
- ✓ No duplica: `https://wa.me/505505...` ✗ (esto NO pasa)

**Verificación en Consola:**
```
[v0] WhatsApp URL generated: {
  telefonoRaw: "505 58243212",
  telefonoLimpio: "58243212",
  whatsappURL: "https://wa.me/50558243212?text=..."
}
```

---

### Escenario 6: Cliente seleccionado no existe

**Precondiciones:**
- El ID del cliente en `formData.cliente_id` no coincide con ninguno en el array

**Pasos:**
1. Intentar crear préstamo cuotario

**Resultados Esperados:**
- ✓ Se muestra alerta: "No se encontró el cliente seleccionado. Por favor, intente nuevamente."
- ✓ No abre el dialog de confirmación
- ✓ No guarda datos

---

### Escenario 7: Múltiples clientes diferentes

**Prueba de repetibilidad:**
1. Crear préstamo Cliente A ✓
2. Cancelar, volver atrás
3. Crear préstamo Cliente B ✓
4. Verificar que cada uno muestra sus datos correctos

**Resultados Esperados:**
- ✓ Cada cliente muestra SUS datos exactos
- ✓ Sin mezcla de datos entre clientes
- ✓ WhatsApp usa el número correcto de cada cliente

---

## Verificaciones Técnicas

### 1. Consulta a Base de Datos

```sql
SELECT id, nombre_completo, numero_cedula, telefono, comarca, municipio, departamento
FROM clientes
```

- ✓ Retorna todos los registros
- ✓ Campos están poblados o null (no errores)
- ✓ Respuesta rápida (<1s)

### 2. Flujo de React

- ✓ `useEffect` carga clientes al montar
- ✓ `setClientes(data)` actualiza estado
- ✓ Find() localiza cliente por ID correctamente
- ✓ Preview recibe todos los datos

### 3. Validación WhatsApp

- ✓ Trim() elimina espacios
- ✓ `/\D/g` elimina caracteres especiales
- ✓ Validación de 8 dígitos es exacta
- ✓ Prevención de 505 duplicado funciona
- ✓ URL se genera sin caracteres especiales

### 4. Logs en Consola

- ✓ `[v0] Cliente data loaded:` muestra todos los campos
- ✓ `[v0] WhatsApp URL generated:` muestra transformación
- ✓ `[v0] WhatsApp validation failed:` muestra errores

---

## Checklist de Validación

- [ ] Todos los clientes muestran datos sin N/A innecesarios
- [ ] WhatsApp abre correctamente con número válido
- [ ] Mensajes de error son claros y ayudan a diagnosticar
- [ ] Botón Continuar guarda el préstamo correctamente
- [ ] Los datos se persisten en la base de datos
- [ ] No hay errores en la consola del navegador
- [ ] El contrato se genera con información completa
- [ ] El historial de cuotas se crea correctamente
- [ ] La caja se actualiza correctamente
- [ ] Los movimientos se registran correctamente

---

## Notas Finales

- La consulta a la BD incluye TODOS los campos necesarios
- La lógica defensiva maneja valores null/undefined
- El sistema es tolerante a errores pero informativo
- Mensajes claros permiten que el admin diagностique problemas
- La integración con WhatsApp es robusta y validada
