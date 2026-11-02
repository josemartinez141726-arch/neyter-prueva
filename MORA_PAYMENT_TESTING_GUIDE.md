# MORA PAYMENT SYSTEM - TESTING & VERIFICATION GUIDE

## Pre-Implementation Checklist

### Environment Setup
- [ ] Supabase connection verified
- [ ] All tables created and accessible
- [ ] RLS policies configured correctly
- [ ] Admin credentials set in database
- [ ] Mora configuration value set (default: C$ 10/day)

### Database Verification
- [ ] `prestamos` table has `dias_mora` column
- [ ] `historial_prestamos` table has `dias_extension` column
- [ ] `caja` table exists with `total` field
- [ ] `movimientos` table has `tipo` constraint ("pago" is valid)
- [ ] `clientes` table accessible
- [ ] `admins` table with `contrasena_acceso`

## Test Scenarios

### Scenario 1: Simple Mora Payment

**Setup**
```
Client: Jose Martinez
Mora: 4 days
Value/day: C$ 5
Total: C$ 20
Password: correctpassword
```

**Test Steps**
1. Search for "Jose Martinez"
2. Select client from results
3. System shows: "4 días de mora", "Total a pagar por mora: C$ 20"
4. Click "Pagar Mora"
5. Enter password: "correctpassword"
6. Click "Confirmar"

**Expected Results**
- ✓ Success alert shown
- ✓ Dialog closes
- ✓ Client cleared from selection
- ✓ Search box cleared
- ✓ Can search for next client

**Database Verification**
```sql
-- Check prestamos.dias_mora reset
SELECT dias_mora FROM prestamos WHERE cliente_id = 'jose_id';
-- Expected: 0

-- Check historial_prestamos offset
SELECT dias_extension, tipo FROM historial_prestamos 
WHERE prestamo_id IN (SELECT id FROM prestamos WHERE cliente_id = 'jose_id');
-- Expected: -4 (negative offset)

-- Check caja increase
SELECT total FROM caja;
-- Expected: previous_total + 20

-- Check movimientos record
SELECT * FROM movimientos 
WHERE cliente_id = 'jose_id' AND descripcion LIKE 'Pago de moras%';
-- Expected: Record with tipo='pago', monto=20
```

### Scenario 2: Incorrect Password

**Setup**
```
Client: Maria Hernandez
Mora: 2 days
Password entered: wrongpassword
Correct password: correctpassword
```

**Test Steps**
1. Select client with mora
2. Click "Pagar Mora"
3. Enter incorrect password
4. Click "Confirmar"

**Expected Results**
- ✓ Error message shown in red: "Contraseña incorrecta, inténtalo de nuevo."
- ✓ Dialog remains open
- ✓ Password field keeps focus
- ✓ No changes to database

**Database Verification**
```sql
-- Verify NO changes occurred
SELECT dias_mora FROM prestamos WHERE cliente_id = 'maria_id';
-- Expected: Still 2 (unchanged)

SELECT total FROM caja;
-- Expected: Previous total (unchanged)

-- Verify NO movimiento created
SELECT COUNT(*) FROM movimientos 
WHERE cliente_id = 'maria_id' AND created_at > NOW() - INTERVAL '5 minutes';
-- Expected: 0
```

### Scenario 3: Client with No Mora

**Setup**
```
Client: Juan Perez
Mora: 0 days
```

**Test Steps**
1. Search for "Juan Perez"
2. Select client
3. System loads mora information

**Expected Results**
- ✓ Message shown: "Este usuario no tiene mora registrada"
- ✓ "Pagar Mora" button disabled or not shown
- ✓ No payment can be processed

### Scenario 4: Decimal Precision Test

**Setup**
```
Client: Luis Rodriguez
Mora days: 3
Value/day: C$ 12.50
Expected total: C$ 37.50
```

**Test Steps**
1. Select client
2. Verify total displays as "C$ 37.50"
3. Enter password and process payment

**Database Verification**
```sql
-- Verify exact decimal stored
SELECT monto FROM movimientos 
WHERE cliente_id = 'luis_id' ORDER BY created_at DESC LIMIT 1;
-- Expected: 37.50 (exactly, not 37.5 or 37.4999)

-- Verify caja precision
SELECT total FROM caja;
-- Expected: Exact value with 2 decimals
```

### Scenario 5: Multiple Previous Mora Registrations

**Setup**
```
Client: Ana Sanchez
Previous mora: Day 1-2 registered as "mora" type (2 days)
Previous mora: Day 3-5 registered as "mora" type (3 days)
Total mora: 5 days
Current value/day: C$ 8
Expected total: C$ 40
```

**Test Steps**
1. Select client
2. System should sum all mora entries: 2 + 3 = 5 days
3. Calculate total: 5 × 8 = 40
4. Process payment

**Database Verification**
```sql
-- Verify all mora entries summed correctly
SELECT SUM(dias_extension) as total_mora
FROM historial_prestamos
WHERE prestamo_id IN (SELECT id FROM prestamos WHERE cliente_id = 'ana_id')
AND (tipo = 'mora' OR tipo = 'retraso');
-- Expected: 0 (due to -5 offset from payment)

-- Verify historical record of payment
SELECT dias_extension, tipo FROM historial_prestamos
WHERE prestamo_id IN (SELECT id FROM prestamos WHERE cliente_id = 'ana_id')
ORDER BY fecha_registro DESC LIMIT 1;
-- Expected: tipo='pago_mora', dias_extension=-5
```

### Scenario 6: Report Synchronization Test

**Setup**
```
Client has 3 days mora
Payment processed successfully
```

**Test Steps**
1. Process mora payment
2. Navigate to Informes section
3. Search for same client
4. Check "Historial crediticio" field

**Expected Results**
- ✓ Historial crediticio shows: 0 días de mora
- ✓ Último préstamo status: Puntual (if applicable)
- ✓ No mora indicators in report
- ✓ Credit status: Clean

## Verification Checklist

### Financial Accuracy Verification

After each mora payment, verify:

- [ ] Cash register increased by exact amount
- [ ] No rounding errors (check to 2 decimals)
- [ ] Movimientos record matches caja increase
- [ ] No duplicate transactions
- [ ] Transaction timestamp is current

### Credit History Verification

After each mora payment, verify:

- [ ] `prestamos.dias_mora = 0` for all client loans
- [ ] `historial_prestamos` has negative offset entry
- [ ] Sum of mora entries = 0
- [ ] `tipo = "pago_mora"` record created
- [ ] Historical audit trail complete

### Data Integrity Verification

After each mora payment, verify:

- [ ] No orphaned records
- [ ] Client selection cleared properly
- [ ] UI returned to initial state
- [ ] Search results cleared
- [ ] No errors in console

### Report Verification

After each mora payment, verify in Informes:

- [ ] Historial crediticio: 0 días
- [ ] Último préstamo: Puntual status updated
- [ ] Client list shows clean history
- [ ] No stale cache issues
- [ ] Refresh shows correct data

## Performance Benchmarks

Expected performance targets:

- Search results: < 1 second
- Load mora info: < 1 second
- Process payment: < 3 seconds
- Post-payment verification: < 2 seconds
- Report refresh: < 2 seconds
- **Total end-to-end**: < 8 seconds

## Common Issues & Solutions

### Issue: Payment processed but mora still shows in reports

**Solution:**
1. Check `prestamos.dias_mora` field value
2. Verify negative offset in `historial_prestamos`
3. Check if report is using cache (refresh browser)
4. Verify query filter for mora type

### Issue: Decimal rounding causes discrepancies

**Solution:**
1. Always use: `Math.round(amount * 100) / 100`
2. Verify database field precision (numeric type)
3. Check for floating-point accumulation
4. Use string comparison for verification

### Issue: Password validation fails intermittently

**Solution:**
1. Verify admin password in database
2. Check for whitespace in stored password
3. Verify character encoding
4. Check database query returning correct value

### Issue: Caja total doesn't update

**Solution:**
1. Verify RLS policies allow update
2. Check if correct caja record ID
3. Verify amount calculation
4. Check database error logs

## Sign-Off Checklist

Before deploying to production:

- [ ] All test scenarios passed
- [ ] Financial accuracy verified
- [ ] Credit history cleanup confirmed
- [ ] Report synchronization working
- [ ] Error handling tested
- [ ] Performance benchmarks met
- [ ] Decimal precision validated
- [ ] Security (password verification) tested
- [ ] Audit trail complete
- [ ] Documentation reviewed

## Rollback Plan

If issues found in production:

1. **Partial Payment**: Adjust caja manually via UI
2. **Incomplete Cleanup**: Update `prestamos.dias_mora` directly
3. **Duplicate Transaction**: Delete from movimientos (keep audit)
4. **Data Discrepancy**: Contact Supabase support, restore from backup

## Monitoring

Post-deployment monitoring:

- Daily: Review mora payments processed
- Weekly: Audit cash register accuracy
- Monthly: Verify mora history cleanup
- Quarterly: Performance analysis
