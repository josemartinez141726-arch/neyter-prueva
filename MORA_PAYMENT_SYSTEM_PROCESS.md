## Comprehensive Mora Payment System Process Documentation

### Overview
This document defines the complete process for handling mora (late payment interest) payments in the credit system. The process ensures accurate financial accounting, complete credit history cleanup, and seamless future mora tracking if needed.

---

## 1. PAYMENT INITIATION AND VALIDATION

### Step 1.1: Load Client Mora Information
**Function**: `cargarInfoMora(clienteId: string)`

**Process**:
1. Fetch mora configuration from `configuracion_moras` table
   - Get `valor_por_dia` (daily mora fee)
   - Fail if not found (default to C$ 10)

2. Retrieve all loans for the client from `prestamos` table
   - Query: `SELECT id FROM prestamos WHERE cliente_id = clienteId`
   - If no loans exist: set mora to 0, display "no mora"

3. Sum accumulated mora from credit history via `historial_prestamos`
   - Query all records for each loan with `tipo = 'retraso' OR tipo = 'mora'`
   - Sum `dias_extension` values to get total accumulated mora days
   - Calculate: `total_mora = dias_mora × valor_por_dia`

4. Display mora information to admin
   - Days of mora
   - Daily rate
   - Total amount due

**Error Handling**:
- If configuration fails: throw error, display message to user
- If client has no history: display "No mora registered"
- If database query fails: display error, allow retry

**Data Integrity Check**:
- Verify mora calculation matches credit history report
- Ensure no negative values
- Validate currency formatting

---

## 2. PAYMENT AUTHORIZATION

### Step 2.1: Secure Admin Password Validation
**Function**: `handleConfirmarPago()`

**Security Protocol**:
1. Require admin password for payment authorization
   - Do NOT use localStorage credentials
   - Query directly from `admins` table in Supabase

2. Retrieve admin credentials from database
   - Query: `SELECT id, contrasena_acceso FROM admins LIMIT 1`
   - Compare user input with `contrasena_acceso` field

3. Validate password
   - If INCORRECT:
     * Display: "Contraseña incorrecta, inténtalo de nuevo." in red
     * Clear password field
     * Return without processing
     * Log unauthorized attempt
   
   - If CORRECT:
     * Clear error message
     * Proceed to Step 2.2

**Security Notes**:
- Never store password in component state after validation
- Always validate against live database
- Implement rate limiting for failed attempts (future enhancement)

---

## 3. FINANCIAL ACCOUNTING

### Step 3.1: Update Cash Register (Caja)

**Process**:
1. Fetch current cash total
   - Query: `SELECT total, id FROM caja LIMIT 1`
   - Handle missing record (create with id=1 if needed)

2. Calculate new total
   ```
   newTotal = currentTotal + moraPaymentAmount
   ```
   - Ensure numerical precision (round to 2 decimals)
   - Verify amount is positive

3. Update cash register
   - Query: `UPDATE caja SET total = newTotal WHERE id = cajaData.id`
   - Verify update succeeded
   - Throw error if update fails

**Financial Controls**:
- Amount must be exact payment due (no partial payments yet)
- No manual overrides allowed
- All changes logged with timestamp
- Decimal precision: enforce 2 decimal places

**Error Handling**:
```javascript
if (cajaError) {
  console.error("[v0] Error updating caja:", cajaError)
  throw new Error("Failed to update cash register")
  // Rollback: all subsequent operations must not proceed
}
```

---

## 4. TRANSACTION RECORDING

### Step 4.1: Log Movement in Transacciones System

**Process**:
1. Create timestamp
   - Get current date/time with timezone
   - Format: "YYYY-MM-DD HH:MM:SS"

2. Insert movement record
   - Table: `movimientos`
   - Fields:
     * `cliente_id`: User ID
     * `descripcion`: "Pago de moras - [Client Name] - C$ [Amount] - [Date] [Time]"
     * `monto`: Exact mora payment amount
     * `tipo`: "pago" (MUST be valid constraint value)
     * `created_at`: Current timestamp (auto)

3. Verify insertion
   - Confirm record saved successfully
   - Throw error if constraint violation
   - Valid tipo values: "deposito", "retiro", "pago", "prestamo"

**Audit Trail**:
```javascript
const { error: movError } = await supabase
  .from("movimientos")
  .insert({
    cliente_id: clienteSeleccionado.id,
    descripcion: `Pago de moras - ${clienteSeleccionado.nombre_completo} - C$ ${moraInfo.total_mora}`,
    monto: moraInfo.total_mora,
    tipo: "pago"
  })

if (movError) {
  console.error("[v0] Error registering movement:", movError)
  throw movError
}
```

---

## 5. CREDIT HISTORY CLEANUP

### Step 5.1: Clear Mora from Credit History

**Critical Step**: Complete elimination of mora records to reset credit status

**Process**:
1. Retrieve all client loans
   - Query: `SELECT id FROM prestamos WHERE cliente_id = clienteId`

2. For each loan, create "pago_mora" record in `historial_prestamos`
   - Table: `historial_prestamos`
   - Fields:
     * `prestamo_id`: Primary loan ID
     * `tipo`: "pago_mora" (new type for cleanup)
     * `descripcion`: "Pago de moras procesado - [Days] días liquidados por C$ [Amount]"
     * `dias_extension`: Negative value = -1 × (accumulated mora days)
     * `monto_pagado`: Total mora amount paid

3. Net Effect of Negative dias_extension
   - Previous calculation: `mora_days = SUM(dias_extension WHERE tipo IN ('retraso', 'mora'))`
   - After payment: negative entry cancels out accumulated days
   - Result: Total mora = 0 on next calculation

**Example**:
```
Before payment:
- Record 1: tipo='mora', dias_extension=4 (Total = 4 days)

After payment:
- Record 2: tipo='pago_mora', dias_extension=-4 (Total = 4 + (-4) = 0 days)
```

**Implementation**:
```javascript
const { error: historialError } = await supabase
  .from("historial_prestamos")
  .insert({
    prestamo_id: allPrestamos[0].id,
    tipo: "pago_mora",
    descripcion: `Pago de moras - ${moraInfo.dias_mora} días × C$ ${valorMoraConfig} = C$ ${moraInfo.total_mora}`,
    dias_extension: -moraInfo.dias_mora, // NEGATIVE to cancel out
    monto_pagado: moraInfo.total_mora
  })

if (historialError) throw historialError
```

---

## 6. VERIFICATION AND REPORTING

### Step 6.1: Verify Complete Payment Processing

**Verification Checklist**:
- [ ] Cash register updated with correct amount
- [ ] Movement record created in transaction log
- [ ] Negative entry added to credit history
- [ ] No partial payments left
- [ ] User mora days now equal 0
- [ ] Status indicators reset for reports

### Step 6.2: Update Informes (Reports)

When user requests credit report (`informes.tsx`):
1. Calculate historial crediticio:
   ```javascript
   const diasMora = historial
     .filter(h => h.tipo === 'retraso' || h.tipo === 'mora')
     .reduce((sum, h) => sum + (h.dias_extension || 0), 0)
   // After payment: this will be 0 because negative entry cancels out
   ```

2. Credit indicators show:
   - Mora days: 0
   - Last payment: [Mora payment date]
   - Status: "Historial limpio" (Clean history)

---

## 7. ERROR HANDLING AND RECOVERY

### Step 7.1: Transaction Failure Points

**Critical Failures** (Stop and Rollback):
1. **Admin authentication fails**
   - Action: Return to password entry
   - No data changes
   - Display error message

2. **Cash register update fails**
   - Action: Stop process
   - Alert admin: "Could not update cash register"
   - No subsequent operations
   - Ensure consistency: if caja fails, mora payment doesn't proceed

3. **Movement recording fails**
   - Action: Stop process
   - Alert: "Could not record transaction"
   - Manual review required

4. **Credit history cleanup fails**
   - Action: Stop process
   - Alert: "Could not finalize mora payment"
   - Previous cash update becomes inconsistent (requires manual audit)

### Step 7.2: Graceful Error Display

**User Feedback**:
```javascript
try {
  // All payment steps
} catch (err) {
  const errorMsg = err instanceof Error ? err.message : "Error al procesar pago"
  console.error("[v0] Error procesando pago de mora:", errorMsg)
  setErrorContrasena(errorMsg) // Display in red
  // Dialog stays open, user can retry
}
```

---

## 8. FUTURE MORA TRACKING

### Step 8.1: New Mora Occurrence After Payment

**How System Restarts Tracking**:

If client takes new loan and incurs new mora:
1. New loan created in `prestamos` table
2. If payment delayed, new record added to `historial_prestamos`
   - `tipo: "retraso"` or `tipo: "mora"`
   - `dias_extension: [new count starting from 0]`

3. Mora calculation begins fresh
   - Previous "pago_mora" record remains in history (audit trail)
   - New mora days calculated independently
   - System treats as completely separate instance

**Clean Slate Behavior**:
- Previous mora payment doesn't affect new mora
- Historical record preserved for audit
- Credit calculation starts from zero days
- New payment process follows identical steps

---

## 9. DATA INTEGRITY STANDARDS

### Step 9.1: Database Constraints

**Table: `caja`**
- `total` must be numeric(12,2)
- Must never be negative
- Always updated atomically

**Table: `movimientos`**
- `tipo` must be one of: "deposito", "retiro", "pago", "prestamo"
- `monto` must be positive numeric(12,2)
- `cliente_id` must exist in clientes table

**Table: `historial_prestamos`**
- `tipo` values: "retraso", "mora", "pago_mora", "pago", "abono"
- `dias_extension` can be positive or negative
- `prestamo_id` must exist in prestamos table

### Step 9.2: Validation Rules

| Field | Validation | Error Message |
|-------|-----------|------------------|
| Payment Amount | > 0 and = calculated total | "Monto debe coincidir con mora calculada" |
| Admin Password | Must match database | "Contraseña incorrecta, inténtalo de nuevo." |
| Mora Days | Must sum correctly | "Discrepancia en cálculo de moras" |
| Cash Update | Must succeed atomically | "No se pudo actualizar caja" |

---

## 10. AUDIT AND COMPLIANCE

### Step 10.1: Complete Audit Trail

Every mora payment creates:
1. **Cash Register Entry**: Updated total with timestamp
2. **Transaction Record**: Movement with full description
3. **Credit History Record**: pago_mora entry with:
   - Admin who processed
   - Exact amount paid
   - Days cleared
   - Date and time
   - Original and payment records visible

### Step 10.2: Report Generation

Users can verify payments via:
- **Informes**: Credit history shows mora = 0
- **Movimientos**: Lists "Pago de moras" transaction
- **Caja**: Shows updated total (accessible to admin only)

---

## 11. TESTING CHECKLIST

### Pre-Deployment Tests

- [ ] Payment processes with correct admin password
- [ ] Payment blocked with incorrect password
- [ ] Cash register updates by exact amount
- [ ] No discrepancies in accounting
- [ ] Movement record created with correct type
- [ ] Credit history shows mora = 0 after payment
- [ ] New mora can be incurred separately if loan is taken
- [ ] Error messages display correctly
- [ ] Database remains consistent on all paths
- [ ] Decimal precision maintained (no rounding errors)
- [ ] Negative dias_extension cancels out accumulation

---

## 12. MAINTENANCE AND MONITORING

### Regular Checks
- Monthly: Verify caja totals against movement logs
- Weekly: Check for payment discrepancies in historial_prestamos
- Daily: Monitor error logs for failed payment attempts

### Future Enhancements
1. Partial mora payments (pay portion of accumulated mora)
2. Automatic mora accrual (daily interest on delayed payments)
3. Mora payment plans (spread payments over time)
4. Automated notifications (SMS/email before mora threshold)
5. Rollback capability (admin can reverse payment within grace period)

---

## System Implementation Summary

The mora payment system operates with three critical safeguards:
1. **Authentication**: Admin password verified against live database
2. **Financial Accuracy**: Exact amount tracking with decimal precision
3. **Credit Cleanup**: Negative entries in history completely offset accumulated days

This design ensures that when a client pays mora, their credit history is completely reset, and if they incur new mora on future loans, the system accurately tracks it from zero.
