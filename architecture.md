# 3. Architecture

```
                  ┌──────────────────────────────────────────────────────┐
                  │                  Stave Data Tools                    │
                  └──────────────────────────────────────────────────────┘

  ┌─── Definition tables ───┐    ┌── Engines (Script Includes) ──┐    ┌── Outputs / Logs ──┐

  data_mask_definition  ───────► DataMaskerHelper.maskRecord()  ─────► data_mask_alert
                                 (or Masker.getBool())                  data_mask_replaced

  data_scramble_definition ────► (UI Action "Scramble Data")  ────────► data_scramble_log
                                  + DataScramble_Ajax.getTotalCount()    data_mask_alert
                                                                         (when alerts=true)

  data_generator ──────────────► DataToolsHelper.generateData()  ─────► target table (new rows)

  dictionary_entry  ────────────► consulted by Mask Definitions
                                  when dictionary_lookup = true
```

**Real-time mask trigger flow:**

1. A user saves any record on `task` (or a child class).
2. The `DataMasker Check` business rule (before, insert/update) fires.
3. It calls `new x_stave_dt.DataMaskerHelper().maskRecord(current)`.
4. The helper queries `x_stave_dt_data_mask_definition` records where `active = true` and `table = current.getTableName()`, ordered by `order`.
5. For each match, the helper either creates an alert record (`mode = Alert`) or replaces the field value in `current` (`mode = Replace`), and writes a row to `data_mask_replaced` when an actual replacement occurred.
