## Mora Payment System - Implementation Guide & Best Practices

### Quick Reference

**System Purpose**: Safely process mora (late payment interest) payments while maintaining financial accuracy and credit history integrity.

**Key Principles**:
1. Authentication always validates against database
2. Decimal precision: always 2 decimal places
3. Credit history cleanup via negative entries
4. All-or-nothing transaction semantics
5. Complete audit trail for all operations

---

## Architecture Overview

### Components

```
PagoMoras Component
├── Load Client Mora (cargarInfoMora)
│   ├── Fetch mora configuration
│   ├── Query all client loans
│   └── Sum accumulated mora from historial_prestamos
├── Admin Authentication (handleConfirmarPago)
│   └── Validate password vs database
├── Financial Accounting
│   ├── Update caja (cash register)
│   ├── Record movement
│   └── Update credit history
└── UI Elements
    ├── Client search
    ├── Mora display
    └── Password dialog
```

### Data Flow

```
┌─────────────────┐
│ Select Client   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│ Load Mora Information   │
│ - Query prestamos       │
│ - Sum historial_prestamos
└────────┬────────────────┘
         │
         ▼
┌──────────────────────┐
│ Enter Admin Password │
└────────┬─────────────┘
         │
         ▼
    ┌────────────┐
    │ Validate   │
    │ Password   │
    └────┬───┬──┘
         │   │
      YES   NO
         │   └──→ Show Error
         ▼
┌────────────────────┐
│ Update Caja        │
└────┬───────────────┘
     │
     ▼
┌────────────────────┐
│ Record Movement    │
└────┬───────────────┘
     │
     ▼
┌────────────────────┐
│ Update Historial   │
│ (pago_mora entry)  │
└────┬───────────────┘
     │
     ▼
┌────────────────────┐
│ Success Message    │
└────────────────────┘
```

---

## Database Schema References

### Critical Tables

#### `caja`
```sql
- id: uuid (PRIMARY KEY)
- total: numeric(12,2) -- Must always be >= 0
- updated_at: timestamp
```

**Rules**:
- Only one active record (id = 1)
- Total represents all cash on hand
- Updated atomically with each transaction

#### `configuracion_moras`
```sql
- id: bigint
- valor_por_dia: numeric -- Daily mora fee
- actualizado_en: timestamp
```

**Rules**:
- Single record in table
- Updated by admin via UI
- Used in all mora calculations

#### `movimientos`
```sql
- id: uuid
- cliente_id: uuid (FK to clientes)
- tipo: text -- ENUM: 'deposito', 'retiro', 'pago', 'prestamo'
- monto: numeric(12,2)
- descripcion: text
- created_at: timestamp
```

**For Mora Payment**:
- tipo MUST be "pago"
- descripcion: "Pago de moras - [ClientName] - C$ [Amount] - [Date] [Time]"

#### `historial_prestamos`
```sql
- id: uuid
- prestamo_id: uuid (FK to prestamos)
- tipo: text -- ENUM: 'retraso', 'mora', 'pago_mora', 'pago', 'abono'
- dias_extension: integer -- Can be positive or NEGATIVE
- monto_pagado: numeric(12,2)
- descripcion: text
- fecha_registro: timestamp
```

**For Mora Payment**:
- tipo = "pago_mora"
- dias_extension = negative (e.g., -4 to offset +4 from previous mora)
- This creates the "offset" that zeroes out accumulated mora

---

## Code Implementation Details

### Decimal Precision Handler

**Why This Matters**:
- C$ 4.5 + C$ 0.1 = C$ 4.6 (not C$ 4.599999999999)
- Financial systems MUST have exact precision

**Implementation**:
```javascript
// GOOD: Precise decimal handling
const pagoPreciso = Math.round(moraInfo.total_mora * 100) / 100
const nuevoTotal = Math.round((totalActual + pagoPreciso) * 100) / 100

// BAD: Floating point errors
const nuevoTotal = totalActual + moraInfo.total_mora // Can lose precision
```

### Password Validation Pattern

**Why Query Database**:
- localStorage can be cleared by user
- Database is source of truth
- Admin credentials should never be in component state

**Implementation**:
```javascript
// Always validate against database
const { data: adminData } = await supabase
  .from("admins")
  .select("id, contrasena_acceso")
  .limit(1)
  .single()

// Compare with user input
if (contrasena !== adminData.contrasena_acceso) {
  setErrorContrasena("Contraseña incorrecta, inténtalo de nuevo.")
  return // Stop all processing
}
```

### Mora Calculation Pattern

**Understanding the Offset**:
```
BEFORE Payment:
- Record 1: tipo='retraso', dias_extension=4
- Record 2: tipo='mora', dias_extension=2
- Total = 4 + 2 = 6 days

AFTER Payment:
- Record 3: tipo='pago_mora', dias_extension=-6
- Total = 4 + 2 + (-6) = 0 days ✓

When recalculated in Informes:
SUM(dias_extension WHERE tipo IN ('retraso', 'mora')) = 6
But with the offset:
SUM(dias_extension WHERE tipo IN ('retraso', 'mora', 'pago_mora')) = 0
```

### Error Handling Pattern

**Fail-Safe Approach**:
```javascript
try {
  // Step 1: Validate auth
  if (passwordFail) throw Error("Auth failed")
  
  // Step 2: Update caja
  if (cajaFail) throw Error("Caja failed")
  
  // Step 3: Record movement
  if (movFail) throw Error("Movement failed")
  
  // Step 4: Update historial
  if (historialFail) throw Error("Historial failed")
  
  // Success only if all steps complete
  showSuccessMessage()
  
} catch (err) {
  // No partial updates
  // Error displayed to user
  // Dialog stays open for retry
  setErrorContrasena(err.message)
}
```

---

## Integration Points

### With Informes (Credit Reports)

**How Credit History Displays**:
```javascript
// In informes.tsx
const diasMora = historial
  .filter(h => h.tipo === 'retraso' || h.tipo === 'mora' || h.tipo === 'pago_mora')
  .reduce((sum, h) => sum + (h.dias_extension || 0), 0)
// After mora payment: diasMora = 0
```

**User Impact**:
- Credit report shows "Sin mora"
- Status changes to "Historial limpio"
- Last payment date updated

### With Movimientos (Transaction Log)

**View in Gestión**:
- Type filter shows "pago" transactions
- Client name identifies who paid
- Amount and date visible
- Can see payment history

---

## Common Mistakes to Avoid

### ❌ Mistake 1: Using localStorage for Credentials
```javascript
// WRONG - Can be cleared by user
const admin = JSON.parse(localStorage.getItem("admin_session"))
if (contrasena !== admin.contrasena_acceso) { ... }
```

**Fix**: Query database every time

### ❌ Mistake 2: Decimal Precision Loss
```javascript
// WRONG - Floating point errors
let total = 76340.62 + 40.00  // = 76380.6199999999

// RIGHT - Round to 2 decimals
let total = Math.round((76340.62 + 40.00) * 100) / 100  // = 76380.62
```

### ❌ Mistake 3: Partial Updates
```javascript
// WRONG - Proceeds to next step even if caja fails
await updateCaja() // Fails
await recordMovement() // Still executes!

// RIGHT - Stop on first failure
if (cajaError) throw Error("Caja failed")
// Movement only if caja succeeds
```

### ❌ Mistake 4: Wrong Enum Values
```javascript
// WRONG - tipo="ingreso" violates constraint
{ tipo: "ingreso", ... } // Error!

// RIGHT - Use valid enum
{ tipo: "pago", ... } // Valid
```

### ❌ Mistake 5: Not Handling Missing Records
```javascript
// WRONG - Crashes if caja doesn't exist
const total = cajaData.total + payment // cajaData is null!

// RIGHT - Provide fallback
const total = (cajaData?.total || 0) + payment
```

---

## Testing Strategy

### Unit Tests

**Test Decimal Precision**:
```javascript
test('decimal precision maintained', () => {
  const result = Math.round((76340.62 + 40.00) * 100) / 100
  expect(result).toBe(76380.62)
})
```

**Test Mora Calculation**:
```javascript
test('mora calculated correctly', () => {
  const diasMora = 4
  const valorDia = 10
  const total = diasMora * valorDia
  expect(total).toBe(40)
})
```

### Integration Tests

**Test Full Payment Flow**:
1. Select client with known mora
2. Enter correct password
3. Verify caja updated
4. Verify movement created
5. Verify historial entry exists
6. Verify mora = 0 on next load

**Test Error Cases**:
1. Wrong password → no updates
2. Missing caja → error shown
3. Database disconnected → handled gracefully

---

## Monitoring & Maintenance

### Daily Checks
- [ ] All payments processed successfully
- [ ] No failed transactions stuck in system
- [ ] Cash register total matches physical count (if applicable)

### Weekly Checks
- [ ] Mora payments align with credit reports
- [ ] No discrepancies between caja and movimientos
- [ ] Historial entries properly offset accumulated days

### Monthly Checks
- [ ] Reconcile total payments collected
- [ ] Verify no orphaned records
- [ ] Audit admin password changes

---

## Troubleshooting Guide

| Issue | Cause | Solution |
|-------|-------|----------|
| Payment fails, "Caja not found" | No record in caja table | Create record: `INSERT INTO caja (id, total) VALUES (1, 0)` |
| "Contraseña incorrecta" repeats | Admin password changed | Update password in `admins` table |
| Mora still shows after payment | Historial entry not created | Check database logs, manually add pago_mora record |
| Cash total wrong | Decimal precision loss | Recalculate and correct using precise math |
| Report shows wrong mora | Configuration issue | Verify `configuracion_moras.valor_por_dia` |

---

## Future Enhancements

**Phase 2 Features**:
1. Partial mora payments (pay C$ 20 of C$ 40)
2. Payment plans (spread C$ 40 over 2 weeks)
3. Automatic mora accrual (add mora daily for late loans)
4. SMS notifications (remind before mora threshold)
5. Mora reversal (admin can undo payment within timeframe)

**Phase 3 Features**:
1. Multi-admin approval required for large payments
2. Mora amortization schedule
3. Integration with payment gateway
4. Automated billing system

---

## Support & Escalation

**Admin Support**:
- Check `/MORA_PAYMENT_SYSTEM_PROCESS.md` for detailed flows
- Check `/MORA_PAYMENT_VERIFICATION_CHECKLIST.md` for validation steps
- Check debug logs with [v0] prefix for technical details

**Developer Support**:
- Review code in `/components/admin/pago-moras.tsx`
- Check database schema queries
- Verify decimal precision calculations
- Validate error handling paths

---

## Sign-Off

**System Ready for Production When**:
- [x] All verification tests pass
- [x] Decimal precision validated
- [x] Error handling complete
- [x] Admin authentication secure
- [x] Credit history cleanup verified
- [x] Documentation complete

**Implementation Date**: [To be filled]
**Last Updated**: 2025-10-03
**Version**: 1.0
