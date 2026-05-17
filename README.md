# SAP ABAP Cloud — Internal Early Numbering in RAP

> **Tutorial:** SAP Learning Studio — Part 3 of the Sales Order App Series  
> **Topic:** Internal Early Numbering using `cl_numberrange_runtime` in a RAP Behavior Implementation Class

---

## ⚠️ Prerequisite —  Configuration (Do This First!)

Before deploying any code, you **must** enable buffering on the Number Range Object.
Without this, the app will crash with dump `BEHAVIOR_ILLEGAL_STATEMENT`.

```
Object      : ZNU_RANGE
Action      : Edit → Buffering → Main Memory Buffering
Buffer Size : 10
→ Save & Activate
```

**Why this matters:**
Without buffering, `cl_numberrange_runtime=>number_get` executes an `UPDATE nriv`
on the **primary DB connection**, which is forbidden during the RAP interaction phase.
With buffering ON, SAP switches to a secondary DB connection (`R/3*NRIV`),
which is fully permitted in `EarlyNumbering`.

---

## 📁 File Structure

```
your-rap-project/
│
├── ZBP_SLS_I_ORDER_HDR.clas.abap     ← Behavior Implementation Class (this file)
├── ZBEHV_SLS_I_ORDER_HDR.bdef        ← Behavior Definition
└── README.md
```

---

## 💻 Behavior Implementation Class

```abap
CLASS lhc_ZSLS_I_ORDER_HDR DEFINITION INHERITING FROM cl_abap_behavior_handler.
  PRIVATE SECTION.

    METHODS get_global_authorizations FOR GLOBAL AUTHORIZATION
      IMPORTING REQUEST requested_authorizations FOR zsls_i_order_hdr RESULT result.

    METHODS earlynumbering_create FOR NUMBERING
      IMPORTING entities FOR CREATE zsls_i_order_hdr.

ENDCLASS.

CLASS lhc_ZSLS_I_ORDER_HDR IMPLEMENTATION.

  METHOD get_global_authorizations.
  ENDMETHOD.

  METHOD earlynumbering_create.
    DATA:
      entity       TYPE STRUCTURE FOR CREATE zsls_i_order_hdr,
      order_id_max TYPE zorder_id.

    " Idempotency guard: carry forward entities that already have an Order ID
    " Required for draft-enabled BOs to avoid re-numbering on re-activation
    LOOP AT entities INTO entity WHERE OrderId IS NOT INITIAL.
      APPEND CORRESPONDING #( entity ) TO mapped-zsls_i_order_hdr.
    ENDLOOP.

    DATA(entities_wo_orderid) = entities.
    DELETE entities_wo_orderid WHERE OrderId IS NOT INITIAL.

    " Nothing left to number — exit cleanly
    CHECK entities_wo_orderid IS NOT INITIAL.

    " *** NUMBER RANGE PATH (always active) ***
    " PREREQUISITE: Buffering MUST be enabled on object ZNU_RANGE in  .
    " Without buffering, cl_numberrange_runtime=>number_get executes
    " UPDATE nriv on the primary DB connection, which violates the RAP
    " interaction phase contract and causes dump BEHAVIOR_ILLEGAL_STATEMENT.
    " With buffering ON, SAP uses a secondary DB connection (CONNECTION 'R/3*NRIV')
    " which is fully permitted in EarlyNumbering.
    TRY.
        cl_numberrange_runtime=>number_get(
          EXPORTING
            nr_range_nr       = '01'
            object            = 'ZNU_RANGE'
            quantity          = CONV #( lines( entities_wo_orderid ) )
          IMPORTING
            number            = DATA(number_range_key)
            returncode        = DATA(number_range_return_code)
            returned_quantity = DATA(number_range_returned_quantity)
        ).
      CATCH cx_number_ranges INTO DATA(lx_number_ranges).
        " Number range object error (e.g. no interval defined, object missing)
        LOOP AT entities_wo_orderid INTO entity.
          APPEND VALUE #( %cid = entity-%cid
                          %key = entity-%key
                          %msg = lx_number_ranges
                        ) TO reported-zsls_i_order_hdr.
          APPEND VALUE #( %cid = entity-%cid
                          %key = entity-%key
                        ) TO failed-zsls_i_order_hdr.
        ENDLOOP.
        RETURN.
    ENDTRY.

    " Validate return code
    " '1' = last number of interval issued (warning only, still usable)
    " '2' = number range is fully exhausted — must fail
    IF number_range_return_code = '2'.
      LOOP AT entities_wo_orderid INTO entity.
        APPEND VALUE #( %cid = entity-%cid
                        %key = entity-%key
                      ) TO failed-zsls_i_order_hdr.
      ENDLOOP.
      RETURN.
    ENDIF.

    " Guard against partial allocation
    " number_get may return fewer numbers than requested if near interval end
    IF number_range_returned_quantity < lines( entities_wo_orderid ).
      DATA(lv_allocated) = CONV i( number_range_returned_quantity ).
      " Fail the entities that did not get a number
      LOOP AT entities_wo_orderid INTO entity FROM lv_allocated + 1.
        APPEND VALUE #( %cid = entity-%cid
                        %key = entity-%key
                      ) TO failed-zsls_i_order_hdr.
      ENDLOOP.
      " Only process the ones that were actually allocated
      DELETE entities_wo_orderid FROM lv_allocated + 1.
    ENDIF.

    " number_range_key = LAST number in the allocated block
    " Subtract returned_quantity to get the value just before the block starts
    " then increment by 1 per entity in the loop below
    order_id_max = number_range_key - number_range_returned_quantity.

    " Assign unique Order ID to each new entity
    LOOP AT entities_wo_orderid INTO entity.
      order_id_max += 1.

      entity-OrderId      = order_id_max.
      entity-%key-OrderId = order_id_max.

      APPEND VALUE #( %cid = entity-%cid
                      %key = entity-%key
                    ) TO mapped-zsls_i_order_hdr.
    ENDLOOP.

  ENDMETHOD.

ENDCLASS.
```

---

## 🔍 Code Explained — Key Sections

### 1. Idempotency Guard
Entities that already have an `OrderId` are passed straight through to `mapped`.
This prevents re-numbering in draft-enabled Business Objects.

### 2. Early Exit
`CHECK entities_wo_orderid IS NOT INITIAL` exits the method cleanly
if there is nothing left to number.

### 3. Number Range Call
`cl_numberrange_runtime=>number_get` reserves a block of sequential numbers.
`number_range_key` holds the **last** number in the allocated block.

### 4. Return Code Validation

| Return Code | Meaning | Action |
|---|---|---|
| `0` | All good | Continue |
| `1` | Last number of interval issued | Continue (monitor   interval) |
| `2` | Number range fully exhausted | Fail all entities, return |

### 5. Partial Allocation Guard
If SAP returns fewer numbers than requested (near interval end),
excess entities are explicitly failed rather than silently dropped.

### 6. Key Sync — Why Both Lines Are Required
```abap
entity-OrderId      = order_id_max.   " Sets the field value
entity-%key-OrderId = order_id_max.   " Syncs the RAP key structure
```
Without the second line, the `%key` in the `mapped` append
still carries an empty `OrderId` — the framework loses track of the assignment.

---

## 🛠️ Return Code Reference

```
returncode = '0'  → Numbers assigned successfully
returncode = '1'  → Warning: last number of interval just issued
                    → Action: extend interval in   before it runs out
returncode = '2'  → Error: number range exhausted
                    → Action: add new interval in   immediately
```

---

## 🐛 Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `BEHAVIOR_ILLEGAL_STATEMENT` |   buffering is OFF | Enable Main Memory Buffering in   |
| `cx_number_ranges` exception | Number range object missing or no interval defined | Check `ZNU_RANGE` exists in   with interval `01` |
| IDs assigned as `0` or blank | `%key-OrderId` not synced | Ensure both `entity-OrderId` and `entity-%key-OrderId` are set |
| Some records silently unnumbered | No partial allocation guard | Ensure the partial allocation block is present |

---

## 📺 Full Tutorial

Watch the full step-by-step video on **SAP Learning Studio** on YouTube.
Link in channel — subscribe so you don't miss Part 4!

---

## 📄 License

This code is shared for educational purposes as part of the SAP Learning Studio tutorial series.
Free to use and adapt in your own projects.
