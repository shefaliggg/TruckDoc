# VCG Transport — Canonical End-to-End Workflow Spec

Source of truth for app development (Shipper, Driver, Admin/Broker apps + backend). Adopted 2026-09-26.
"Driver" = the carrier/person hauling the load (a separate Carrier entity may come later).

**Model:** Onboard → Review → Bid → Award → Execute → Document → Settle

## 1. Roles

| Role | Can |
|---|---|
| Shipper | Create/post loads, view status, receive & accept/reject driver quotes, view docs, track, confirm delivery/issues, view invoices/payments |
| Driver | Onboard, submit docs, view available loads, submit quotes, accept/sign rate con, complete pickup/delivery stages, upload paperwork, request payout, view earnings |
| Admin/Broker | Approve users/docs, review & publish loads, monitor bidding, intervene in assignments, manage pricing/financials/commission, verify docs, approve exceptions, approve settlements, resolve disputes |

## 2. The Four Gates

1. **Can this person use the platform?** Admin verifies shipper/driver.
2. **Can this load go live?** Admin reviews shipper's load.
3. **Is there a binding agreement?** Shipper accepts quote → Rate Confirmation generated → Admin reviews & approves → Driver signs.
4. **Can money be settled?** Delivery → POD → Document verification → Settlement → Payout.

## 3. Stages

### 0. Shipper onboarding
Submit: legal name, business address, contact, authorized rep, tax/business info, billing info, payment method, supporting docs.
Sign: Shipper Terms & Service Agreement (+ Broker/Shipper Agreement where applicable).
Status: `Pending Verification → Approved`, or `Changes Requested → Shipper Updates → Admin Reviews Again`.
**Only Approved shippers can publish loads.**

### 1. Driver onboarding
Submit: legal name, contact, address, license, vehicle/equipment, insurance, operating/carrier info, banking/payment info, compliance docs.
Sign: Driver/Carrier Agreement + Terms & Conditions + Payment/settlement agreement + Platform/service-fee agreement.
Status: `Pending → Under Review → Approved`, or `Changes Required → Driver Resubmits → Under Review`.
**Driver cannot bid until approved.**

### 2. Shipper creates load
- Pickup: location, date, time/window, appointment req., contact, instructions
- Delivery: same fields
- Freight: commodity, description, equipment type, weight, pieces, dimensions, special handling, temperature, hazmat
- Documents: BOL (if available), load instructions, commodity docs, appointment info, other
- **Shipper does NOT enter the driver rate.** System may compute mileage/route/transit estimate (informational).
- Submit → **Pending Admin Review** (not visible to drivers).

### 3. Admin load review (Gate 2)
Reviews operational info, documents, and commercial structure (Customer charge → expected driver cost → expected margin; driver amount unknown until bidding).
Actions: **Approve & Publish** (→ Open for Bids) · **Request Changes** (→ back to shipper) · **Reject**.

### 4. Load published
Drivers see Available Loads (lane, dates, equipment, weight, Submit Quote). **Never expose shipper pricing/margin to drivers.**

### 5. Driver submits quote
Amount, availability confirmation, optional note, optional supporting doc. Status `Submitted`. No money deducted.

### 6. Shipper reviews quotes
My Loads → Load → View Quotes. Can view driver profile/qualifications/doc status, accept or decline. Shipper selects a driver quote, does not set a price.

### 7. Quote acceptance
System records shipper charge, driver gross, gross margin (e.g. $2,000 / $1,900 / $100). Admin sees all three. Other quotes → `Not Selected` + drivers notified.

### 8. Assignment (mandatory Admin RC review gate)
Shipper accepts → System generates Rate Confirmation → **Admin reviews & approves every Rate Confirmation** before it reaches the driver (not exception-only — every load passes through this gate).
Admin checks: driver qualification/doc status current, insurance not expired, load info correct, pricing within permitted parameters, no operational exception.
Actions: **Approve & Send to Driver** (→ Rate Con released) · **Request Changes** (edit RC, re-review) · **Reject Assignment** (back to bidding or reassign).
Status: `Rate Con Pending Admin Review` → `Rate Con Approved`.

### 9. Rate Confirmation
Contains: load no., shipper, driver/carrier, pickup/delivery, dates/times, equipment, commodity, weight, special instructions, agreed driver rate, terms, accessorial terms, cancellation terms.
Driver receives the admin-approved Rate Con, taps **Acknowledge & Accept**; accepts/signs, timestamp recorded (shipper/broker acknowledgment configurable). Status `Rate Confirmed`; load → `Assigned`. Driver can then proceed to pickup.

### 10. Pre-pickup
Driver receives load details, rate con, pickup instructions, BOL/docs. Confirms **Ready for Pickup** (electronic acknowledgment, not a formal signature). → `En Route to Pickup`.

### 11. Arrival at pickup
**Arrived at Pickup** records date, time, location (if captured), status → `At Pickup`. Driver accesses BOL/load docs; e-sign or upload signed doc if needed.

### 12. Loading
`Loading` → `Picked Up`. Driver may upload signed BOL, pickup receipt, confirmation, photos, inspection/condition docs, exception docs. Stored against the load.

### 13. In transit
Driver can: view route, navigate, update ETA, report delay/breakdown, upload exception docs, message shipper/admin.
Issue flow: Driver reports → Exception created → Admin notified → Admin reviews → Shipper notified if applicable.
**Admin approval required for:** rate adjustment, major route change, additional charges, cancellation, replacement driver.

### 14. Arrival at delivery
**Arrived at Delivery** → `At Delivery` → `Unloading`.

### 15. Delivery & POD
Driver uploads signed POD, delivery receipt, receiver signature, photos, exception/damage docs. Receiver signs POD → driver submits. Record signature, name/role, date/time, document, submission timestamp. Status `Delivered — POD Submitted`.

### 16. Admin document review
Checklist: Rate Confirmation, pickup docs, BOL, POD, required signatures, exception docs, invoice docs (if required).
Outcomes: **Documents Verified** / **Documents Missing** (driver notified, uploads, admin re-reviews) / **Correction Required**.
Load moves to settlement only after required docs are accepted.

**POD rejection/correction loop:**
```
Admin rejects POD
      ↓
Driver gets notification (reason attached)
      ↓
Driver uploads corrected POD
      ↓
Admin verifies again
```
Applies the same way to any rejected delivery/pickup document, not just POD. Status while looping: `POD Submitted — Correction Required` (does not advance to Documents Verified until admin accepts).

### 17. Customer billing
Separate from driver payout. Invoice: freight charge + approved additional charges = total. Payment `Pending → Authorized/Captured or Invoiced → Paid` (timing per payment terms).

### 18. Driver settlement
Service fee is NOT charged at quote, acceptance, or delivery. Only at settlement:
`Delivery → POD → Docs verified → Eligible for settlement → Driver requests payout → Fee calculated → Fee deducted → Net payout sent`
Example: $1,900 gross − 17% ($323) = **$1,577 net**.

### 19. Driver service plans
| Plan | Fee | Benefits |
|---|---:|---|
| Basic | 10% | Brokerage |
| Fuel | 12% | Brokerage + fuel-card discounts |
| Premium | 17% | Brokerage + fuel-card discounts + cargo coverage benefits |

Plan stored on account; shown in Driver → Plan & Billing. Coverage wording finalized separately with legal/insurance.

### 20. Broker commission
`Gross margin = shipper charge − driver gross` ($100). `Broker commission = 30% × margin` ($30). `Company margin = margin − commission` ($70).
**The driver service fee ($323) is a separate revenue stream from freight margin — keep separate in the ledger.**

| Ledger item | Amount |
|---|---:|
| Shipper charge | $2,000 |
| Driver gross | $1,900 |
| Gross load margin | $100 |
| Broker commission | $30 |
| Company margin | $70 |
| Driver service fee | $323 |
| Driver net payout | $1,577 |

Money flow: Shipper → VCG ($2,000) → driver settlement ($1,900, less 17% fee → $1,577) + $100 margin (→ $30 broker / $70 company). Who collects/pays must be reflected in contracts/accounting.

### 21. Completion
Delivery done, POD verified, docs complete, customer billing recorded, driver settlement processed → `Completed`. Load stays accessible for history.

## 4. Status flow
```
DRAFT → PENDING ADMIN REVIEW ⇄ CHANGES REQUESTED
→ APPROVED → OPEN FOR BIDS → BIDDING → QUOTE ACCEPTED → RATE CON PENDING ADMIN REVIEW → RATE CON APPROVED → RATE CONFIRMED (ASSIGNED)
→ EN ROUTE TO PICKUP → AT PICKUP → LOADING → PICKED UP → IN TRANSIT
→ AT DELIVERY → UNLOADING → DELIVERED → POD SUBMITTED → DOCUMENTS VERIFIED
→ SETTLEMENT → PAYOUT / PAYMENT → COMPLETED
```
(Reject and Cancellation are terminal side-exits.)

## 5. Document lifecycle
Formal documents vs. electronic acknowledgments — do not force e-signature on every status change.

**Key principle: don't make every status require a document.** Attach documents only where they naturally occur — Assignment → Rate Confirmation, Pickup → BOL, Delivery → POD, Settlement → statement, Payout → payment record. Status changes like En Route, Arrived, In Transit, Unloading normally carry no document requirement; this keeps the Driver App simple.

| Stage | Document | Signature/Approval |
|---|---|---|
| Shipper onboarding | Shipper Agreement | Shipper |
| Driver onboarding | Driver/Carrier Agreement | Driver |
| Load creation | Load information | Shipper submission |
| Admin review | Load approval record | Admin |
| Quote | Driver quote | Driver submission |
| Quote acceptance | Acceptance record | Shipper |
| Assignment | Rate Confirmation | **Admin approval (mandatory, every load)**, then Driver signs |
| Pickup | BOL / pickup receipt | Applicable parties |
| Loading | Pickup documentation | Driver/shipper |
| Transit | Exception documents | Driver/admin |
| Delivery | POD | Receiver + driver |
| Post-delivery | Settlement documentation | Driver/admin |
| Billing | Invoice | Issuer |
| Payout | Settlement statement | Driver ack if required |

## 6. Admin approval matrix
**Mandatory review:** user onboarding, driver onboarding/qualification, new load before publish, **every Rate Confirmation before it reaches the driver**, additional charges, cancellations, document exceptions, settlement exceptions.
**No manual approval for:** every driver quote (pre-acceptance), normal status updates, navigation, routine messages, normal POD uploads.

## 7. What each app shows
- **Shipper:** load detail (route, status, pickup/delivery, equipment, weight, quote count → View Quotes, documents, messages); after acceptance: driver, rate, status, ETA, doc checklist; financials: invoice + payment status.
- **Driver:** assigned load (accepted rate, pickup/delivery, status), documents (Rate Con, BOL, pickup docs, POD, invoice, other), settlement (gross, plan %, fee, net, status).
- **Admin:** load overview, commercial block (all ledger items above), documents checklist, approval history/audit trail (who/when for every gate).

## 8. Load Details architecture
Single central object with tabs: **Overview · Status & Tracking · Quotes · Documents · Messages · Financials · Activity/Audit Trail**.
Documents grouped: Pre-Load, Assignment, Pickup, Delivery, Settlement.

## 9. Audit trail
Every operational event, document, approval and money movement is logged with actor + timestamp (load created, approved, quote accepted, rate confirmed, delivered, POD verified, settlement approved, payout completed).

## 10. Open implementation decisions
- Snapshot the driver's fee % on the load at quote acceptance (plan changes must not alter in-flight loads).
- Admin reviews every Rate Confirmation, but flag rules (expired insurance, price threshold, etc.) should still auto-surface risk items to prioritize the admin's review queue.
- Payment timing / who collects from shipper; broker vs. carrier contracting structure; legally required signatures — finalize with US transportation counsel, accountant, payment provider. Do not hard-code legal assumptions.
