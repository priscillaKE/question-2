# QUESTION TWO - Payment Processing Module Design

## Question
You are building a Payment Processing module for the university system. Students can pay using Cash, Bank Deposit, Mobile Money, Card, and Cheque. A payment must always produce a receipt, but verification rules vary:
- Bank payments require bank confirmation.
- Mobile Money requires transaction ID verification.
- A cheque requires clearance days.
- The card requires authorization approval

## (a) Problems with Inheritance Design (6 Marks)

The inheritance approach becomes weak when requirements change because each payment type is fixed as its own class and assumes one method per transaction. If one student pays using two methods, inheritance alone cannot model that cleanly without creating many extra subclasses. It also puts verification inside classes, so changing rules (for example bank confirmation logic) would force code changes and recompilation. As integrations with external APIs grow, tightly coupled subclasses become harder to extend and maintain.

Key problems:
1. **Rigid hierarchy** - one class per method assumes one payment = one method; split payment breaks this model
2. **Combinatorial explosion** - supporting combinations would require many extra subclasses
3. **Hard to change rules** - verification logic inside subclasses means changing rules needs code edits + recompilation
4. **Poor runtime configurability** - cannot change rules without recompiling
5. **Weak API extensibility** - external integrations need pluggable adapters/strategies
6. **Violates open/closed** - every new method/rule forces editing existing class logic

## (b) New Design Using Composition + Polymorphism (9 Marks)

### (i) Classes/Interfaces

- **PaymentMethod** (interface)
- **VerificationStrategy** (interface)
- **PaymentPart** (value object)
- **PaymentTransaction** (aggregate/root)
- **ReceiptService**
- **VerificationRuleProvider** (DB/API/config driven)
- **ExternalGatewayClient** (for bank/mobile/card APIs)

Implementations:
- CashMethod, BankMethod, MobileMoneyMethod, CardMethod, ChequeMethod
- BankConfirmation, TxnIdVerification, CardAuthorization, ChequeClearance

### (ii) Responsibilities

- **PaymentMethod**: defines pay(amount) and methodName() polymorphically
- **VerificationStrategy**: defines verify(context); implementations per rule type
- **PaymentPart**: holds one slice of payment (method + amount + verification data)
- **PaymentTransaction**: contains many PaymentPart objects, validates totals, orchestrates processing, returns one final status
- **VerificationRuleProvider**: loads active verification rules at runtime (DB/config/API), so rules change without recompilation
- **ReceiptService**: generates one consolidated receipt from all successful parts
- **ExternalGatewayClient**: wraps calls to bank/mobile/card external systems

### (iii) Split Payment Support

- PaymentTransaction has List<PaymentPart>
- Each part uses a different PaymentMethod object
- Total of all parts must equal target amount
- Process each part independently (pay + verify)
- Issue one consolidated receipt for the transaction

## (c) Pseudocode - UGX 120,000 Split Payment (5 Marks)

```text
total = 120000
bankPart = 0.40 * total      // 48000
mobilePart = 0.60 * total    // 72000

txn = new PaymentTransaction(studentId, total)

txn.addPart(PaymentPart(BankMethod, bankPart, BankVerification, {bankRef:"BK123"}))
txn.addPart(PaymentPart(MobileMoneyMethod, mobilePart, MobileVerification, {txnId:"MM456"}))

result = txn.process()

if result.success:
    receipt = ReceiptService.generateConsolidatedReceipt(txn)
    print(receipt)
else:
    print(result.errors)
```

## (d) Design Mistakes to Avoid (5 Marks)

### Mistake 1: Hard-coding verification rules in method classes
Why bad: Any policy change (e.g., new bank check) requires code edits/rebuild; defeats runtime rule updates.

### Mistake 2: Using if/else or switch on payment type in one big processor
Why bad: Central God-class grows with every new method; high coupling, harder testing, violates polymorphism and open/closed principle.
