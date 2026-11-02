# MORA PAYMENT COMPREHENSIVE SYSTEM DOCUMENTATION

## System Overview

The Mora Payment System is designed to handle user mora (late fees) payments with complete financial accounting accuracy and credit history management. This system ensures data integrity at every transaction step and automatically verifies that all changes are correctly reflected in financial reports.

## Key Principles

1. **Financial Accuracy**: Every peso is tracked with decimal precision (2 places)
2. **Data Integrity**: All-or-nothing transactions with verification checks
3. **Credit History Cleanup**: Complete reset of mora indicators across all tables
4. **Audit Trail**: Full logging of all transactions and state changes
5. **Report Synchronization**: Automatic verification that reports reflect accurate data

## System Architecture

### Database Tables Involved

1. **prestamos** - Main loan records
   - `dias_mora`: Current mora days for the loan
   - `cliente_id`: Reference to client
   - `estado`: Loan status

2. **historial_prestamos** - Loan transaction history
   - `tipo`: Transaction type (retraso, mora, pago_mora, etc.)
   - `dias_extension`: Days added or subtracted
   - `prestamo_id`: Reference to loan

3. **caja** - Cash register
   - `total`: Total cash accumulated

4. **movimientos** - Transaction log
   - `tipo`: Payment type (pago, deposito, etc.)
   - `monto`: Amount paid
   - `descripcion`: Transaction description

5. **clientes** - Client information
   - `id`: Client unique identifier
   - `nombre_completo`: Client name

6. **admins** - Administrator credentials
   - `contrasena_acceso`: Admin password for verification

## Complete Mora Payment Process

### Phase 1: User Selection and Mora Verification

**Step 1.1: Search Client**
- User enters client name or ID in search box
- System queries `clientes` table for matches
- Displays matching clients with name and ID

**Step 1.2: Select Client and Load Mora**
- User clicks on selected client
- System queries all `prestamos` where `cliente_id` matches
- System calculates actual mora by:
  - Getting all `historial_prestamos` records for client's loans
  - Filtering for records with `tipo` = "retraso" or "mora"
  - Summing all `dias_extension` values
- System retrieves `valor_por_dia` from `configuracion_moras`
- Total mora = dias_mora × valor_por_dia
- Display results to user

**Result**: Client mora information displayed accurately

### Phase 2: Payment Initiation

**Step 2.1: User Clicks "Pagar Mora"**
- System validates that mora > 0
- If mora = 0, displays message: "This user has no mora"
- If mora > 0, opens password verification dialog
- Dialog shows exact amount to be charged: C$ X.XX

**Result**: Payment authorization dialog opened

### Phase 3: Security Validation

**Step 3.1: Admin Password Verification**
- User enters administrator password
- System queries `admins` table for stored password
- Compares entered password with database value
- If incorrect: Shows error message in red, returns to password entry
- If correct: Proceeds to payment processing

**Result**: Admin authentication confirmed

### Phase 4: Financial Processing

**Step 4.1: Cash Register Update**
- Retrieve current `caja.total` from database
- Calculate: newTotal = currentTotal + pagoMonto
- Use precise decimal rounding: Math.round(amount × 100) / 100
- Update `caja.total` with newTotal
- Validate: Ensure newTotal >= 0

**Step 4.2: Movement Recording**
- Create movement record with:
  - `cliente_id`: User's ID
  - `tipo`: "pago"
  - `monto`: Exact payment amount (C$ X.XX)
  - `descripcion`: "Pago de moras - [Client Name] - C$ [Amount] - [Date Time]"
  - `created_at`: Current timestamp
- Insert into `movimientos` table

**Step 4.3: Historial Crediticio Registration**
- Get all `prestamos` for the client
- For primary prestamo, insert new `historial_prestamos` record:
  - `tipo`: "pago_mora"
  - `dias_extension`: -diasMora (negative to offset accumulated days)
  - `monto_pagado`: pagoMonto
  - `descripcion`: "Pago de moras procesado - X días liquidados por C$ Y"
  - `prestamo_id`: Primary loan ID

**Step 4.4: Mora Reset**
- Update ALL `prestamos` where `cliente_id` = user
- Set `dias_mora = 0`
- This ensures clean slate for future mora tracking

**Result**: All financial records updated atomically

### Phase 5: Post-Payment Verification

**Step 5.1: Mora Cleanup Verification**
- Query all `prestamos` for client
- Query all `historial_prestamos` for those loans
- Calculate actual mora: sum of `dias_extension` where type = "mora" or "retraso"
- Verify: actual_mora === 0
- Verify: dias_mora field in prestamos === 0
- If verification fails: Throw error, alert admin

**Step 5.2: Cash Register Verification**
- Query `caja.total`
- Verify: registered total === expected total
- If discrepancy >= C$ 0.01: Throw error, alert admin
- Tolerance: Up to C$ 0.01 for floating-point rounding

**Step 5.3: Final Confirmation**
- If all verifications pass: Show success message
- Log: "Mora pagada completamente y historial limpiado"
- Log all transaction details for audit trail

**Result**: Data integrity confirmed

### Phase 6: UI Reset and Report Update

**Step 6.1: UI Cleanup**
- Clear password field
- Close password dialog
- Clear client selection
- Clear mora information
- Clear search box
- Return to initial state

**Step 6.2: Report Synchronization**
- When user navigates to Informes:
  - Historial crediticio now shows: 0 dias de mora
  - Status shows: "Sin mora"
  - Client credit status: Clean
- Future mora tracking restarts from zero

## Error Handling Strategy

### Validation Errors

**Amount Validation**
- Check: mora > 0
- Check: mora < infinity (sanity check)
- Check: newTotal >= 0
- Error message to user: Clear, non-technical

**Database Errors**
- Catch all Supabase errors
- Log full error details
- Show user: "[Operation] could not be completed"
- Prevent partial updates

**Verification Errors**
- If post-payment mora !== 0: Rollback not possible (transaction committed)
- Alert admin: "Verification failed - contact support"
- Log detailed discrepancy information

### Error Recovery

**Failed Operations**
- No state changes until all operations complete
- Transaction-like behavior via sequential operations
- If any step fails: Subsequent steps don't execute
- User kept informed with error messages

**Partial Failures**
- Money added but mora not cleared: Clear manually via database
- Mora cleared but money not added: Adjust caja via UI
- Log all failures for audit trail

## Data Integrity Guarantees

### Financial Accuracy

- All amounts rounded to 2 decimal places
- No floating-point errors in accumulation
- Cash register precisely matches transaction log
- Every peso accounted for in movimientos

### Credit History Accuracy

- Mora days accurately calculated from historial_prestamos
- Complete reset: dias_mora = 0 + negative offset in history
- Verification: Both sources confirm zero mora
- Fresh start: New mora tracking begins at zero

### Audit Trail

- Every transaction logged in movimientos
- Every mora payment recorded in historial_prestamos
- Admin authentication tracked
- Timestamps recorded for all operations

## Report Verification

### Informes Integration

After mora payment, reports automatically reflect:

1. **Historial Crediticio**
   - Shows: 0 días de mora
   - Calculation: Sum of (dias_extension where type = "retraso" or "mora")
   - Expected: 0 (negative offset cancels positive mora)

2. **Último Préstamo Status**
   - Shows: Puntual (if no current mora)
   - Status: Clean credit history

3. **Client Profile**
   - Mora status: Limpio (Clean)
   - Days accumulated: 0
   - Can take new loans without mora restrictions

## Testing Checklist

### Unit Tests
- [ ] Decimal precision maintained in calculations
- [ ] Negative offsets correctly cancel positive mora
- [ ] Password validation works correctly
- [ ] Error handling catches all error types

### Integration Tests
- [ ] Caja total updates correctly
- [ ] Movimiento created with correct data
- [ ] Historial_prestamos offset applied
- [ ] Dias_mora reset to zero
- [ ] Post-payment verification passes

### Report Tests
- [ ] Informes shows 0 mora after payment
- [ ] No lingering mora in any report
- [ ] Client credit history clean
- [ ] Future mora tracking restarts at zero

### End-to-End Tests
- [ ] Complete payment process succeeds
- [ ] All reports updated immediately
- [ ] No data discrepancies
- [ ] Audit trail complete
- [ ] Admin can see transaction history

## Future Enhancements

1. **Batch Mora Payments**: Process multiple clients at once
2. **Partial Mora Payments**: Pay partial mora amounts
3. **Mora Forgiveness**: Admin can forgive mora instead of collecting
4. **Mora History Export**: Export mora payment history for reporting
5. **Automated Mora Reminders**: Notify users of accumulated mora
6. **Multi-currency Support**: Handle multiple currencies

## Maintenance

### Regular Checks
- Weekly: Verify caja balance matches movimientos total
- Monthly: Audit mora payments for accuracy
- Quarterly: Review historial_prestamos for anomalies

### Data Cleanup
- Archive old movimientos (older than 1 year)
- Consolidate historial_prestamos for inactive loans
- Verify no orphaned records exist

## Contact & Support

For issues with the mora payment system, check:
1. Debug logs in console
2. Database constraints and RLS policies
3. Admin password configuration
4. Supabase connectivity
