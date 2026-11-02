# MORA PAYMENT SYSTEM - QUICK REFERENCE GUIDE

## System At-a-Glance

The mora payment system handles late fee collections with automatic verification that ensures:
- Exact financial amounts (decimal precision)
- Complete credit history cleanup
- Fresh mora tracking restart
- Comprehensive audit trail

## Key Files

1. **`/components/admin/pago-moras.tsx`** - Main payment interface
2. **`/components/admin/configurar-moras.tsx`** - Mora rate configuration
3. **`/MORA_PAYMENT_COMPREHENSIVE_SYSTEM.md`** - Full system documentation
4. **`/MORA_PAYMENT_TESTING_GUIDE.md`** - Test scenarios and verification

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    MORA PAYMENT SYSTEM                          │
└─────────────────────────────────────────────────────────────────┘

1. CLIENT SEARCH
   └─→ Input: Name or ID
   └─→ Search: clientes table
   └─→ Output: Matching clients list

2. CLIENT SELECTION → Load Mora Data
   └─→ Query: prestamos WHERE cliente_id
   └─→ Calculate: SUM(dias_extension) from historial_prestamos
   └─→ Formula: dias_mora × valor_por_dia = total_mora
   └─→ Output: Display mora amount to user

3. PAYMENT INITIATION → Admin Password Dialog
   └─→ Display: "Pagar Mora de C$ X.XX"
   └─→ Input: Admin password
   └─→ Validate: Against admins.contrasena_acceso

4. FINANCIAL PROCESSING (if password correct)
   ├─→ Update caja: total += pago_amount
   ├─→ Record: Insert into movimientos
   ├─→ History: Insert pago_mora into historial_prestamos
   └─→ Reset: Set prestamos.dias_mora = 0

5. POST-PAYMENT VERIFICATION
   ├─→ Check: dias_mora in prestamos = 0
   ├─→ Check: SUM(dias_extension) = 0
   ├─→ Check: caja.total = expected
   └─→ Result: Success or error

6. REPORT UPDATE (next time user views)
   └─→ Informes shows: 0 días de mora
   └─→ Status: Sin mora / Clean
```

## Component Architecture

### PagoMoras Component

**State Management**
```javascript
- busqueda: Search term
- clientes: Search results
- clienteSeleccionado: Selected client
- moraInfo: Calculated mora (dias_mora, total_mora)
- valorMoraConfig: Daily mora rate
- mostrarDialogoContrasena: Password dialog toggle
- contrasena: Entered password
- errorContrasena: Error message
- cargando: Processing flag
```

**Main Functions**
- `buscarCliente()` - Search clients by name/ID
- `seleccionarCliente()` - Select and load mora
- `cargarInfoMora()` - Calculate total mora from database
- `handlePagarMora()` - Open payment dialog
- `handleConfirmarPago()` - Process entire payment

## Database Schema Summary

### prestamos
```
- id: uuid (PK)
- cliente_id: uuid (FK to clientes)
- dias_mora: integer (0 after payment)
- estado: text (activo, pagado, etc.)
- fecha_corte: date
- total_pagar: numeric
```

### historial_prestamos
```
- id: uuid (PK)
- prestamo_id: uuid (FK)
- tipo: text (mora, retraso, pago_mora, etc.)
- dias_extension: integer (positive for mora, negative for payment offset)
- monto_pagado: numeric
- descripcion: text
- fecha_registro: timestamp
```

### caja
```
- id: uuid (PK)
- total: numeric (C$ amount)
```

### movimientos
```
- id: uuid (PK)
- cliente_id: uuid (FK)
- tipo: text (pago, deposito, retiro, prestamo)
- monto: numeric
- descripcion: text
- created_at: timestamp
```

## Calculation Examples

### Example 1: Standard Mora Payment
```
Client: Maria (ID: m123)
Mora days: 4
Daily rate: C$ 5
Total: 4 × 5 = C$ 20

After payment:
- prestamos.dias_mora = 0
- historial_prestamos insert: tipo='pago_mora', dias_extension=-4
- caja.total += 20
- movimientos insert: tipo='pago', monto=20
- Informes: Historial crediticio = 0 días
```

### Example 2: Multiple Previous Mora Registrations
```
Client: Juan (ID: j456)
Mora records:
  - Record 1: dias_extension=2, tipo='mora'
  - Record 2: dias_extension=3, tipo='mora'
Total accumulated: 2+3 = 5 days
Daily rate: C$ 8
Total payment: 5 × 8 = C$ 40

After payment:
- historial_prestamos insert: dias_extension=-5, tipo='pago_mora'
- Total mora: 2 + 3 + (-5) = 0 ✓
- Informes: Clean history
```

### Example 3: Decimal Precision
```
Client: Ana (ID: a789)
Mora days: 3
Daily rate: C$ 12.50
Calculation: 3 × 12.50 = 37.50

Precision handling:
Math.round(37.50 * 100) / 100 = 37.50 ✓

Caja update:
Previous: 1000.00
Payment: 37.50
New: 1037.50 ✓
```

## Error Handling Flow

```
Payment Dialog
    ↓
[User enters password]
    ↓
Is password correct?
    ├─→ NO: Show error in red, allow retry
    └─→ YES: Continue to processing
    ↓
Can update caja?
    ├─→ NO: Show error, abort
    └─→ YES: Continue
    ↓
Can record movimiento?
    ├─→ NO: Show error, abort (money added but not logged)
    └─→ YES: Continue
    ↓
Can update historial?
    ├─→ NO: Show error, abort
    └─→ YES: Continue
    ↓
Can reset dias_mora?
    ├─→ NO: Show error, abort
    └─→ YES: Continue
    ↓
Verification checks
    ├─→ FAIL: Show error (data inconsistent)
    └─→ PASS: Show success, clear UI
```

## Integration Points

### With Informes Component
- Mora payment clears `dias_mora` field
- `historial_prestamos` recalculated with offset
- Reports automatically show 0 mora
- No manual refresh needed

### With Configurar Moras Component
- Reads `valor_por_dia` from configuration
- Updates dynamically if admin changes rate
- New rate applies only to future mora

### With Gestion/Total en Caja
- Increases cash total exactly
- Recorded in movimientos with timestamp
- Visible in cash register reconciliation

## Security Considerations

1. **Admin Password Verification**
   - Always fetched from database (not localStorage)
   - Compared on every payment
   - No password stored in code

2. **Data Validation**
   - Amount must be positive
   - No negative cash totals allowed
   - All input sanitized

3. **Audit Trail**
   - Every payment logged with timestamp
   - Admin ID could be added for tracking
   - Full transaction history preserved

## Performance Optimization

1. **Search Optimization**
   - Minimum 2 characters before search
   - Limit results to 10 clients
   - Indexed on nombre_completo and numero_cedula

2. **Query Optimization**
   - Single query to get all prestamos
   - Single query to get all historial_prestamos
   - Post-payment verification uses same queries

3. **UI Responsiveness**
   - Loading flag during processing
   - Buttons disabled during operation
   - User feedback via alerts and error messages

## Troubleshooting Quick Reference

| Issue | Check | Solution |
|-------|-------|----------|
| Payment not processing | Password in admin table | Verify correct password stored |
| Mora not clearing | dias_mora field value | Ensure update query successful |
| Caja not updating | RLS policy on caja | Verify admin has update permission |
| Reports not refreshing | Browser cache | Hard refresh (Ctrl+F5) |
| Decimal errors | Math.round usage | Check calculation precision |
| Search not working | Client exists? | Verify cliente in database |

## Environment Variables

Required for system to function:
```
NEXT_PUBLIC_SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY
SUPABASE_SERVICE_ROLE_KEY
```

## Testing Checklist

Before going live:
- [ ] Can search clients successfully
- [ ] Mora calculation accurate
- [ ] Password verification works
- [ ] Caja updates correctly
- [ ] Reports show zero mora
- [ ] Error handling works
- [ ] Decimal precision maintained
- [ ] Database RLS allows operations

## Contact Information

For issues or questions:
1. Check logs in browser console
2. Review MORA_PAYMENT_COMPREHENSIVE_SYSTEM.md
3. Check MORA_PAYMENT_TESTING_GUIDE.md
4. Review error messages in UI
5. Check Supabase status
