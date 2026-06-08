# 10. Extension Points

- **External name service.** The Random-User scramble mode and the synthetic-user generator both hit the URL in `x_stave_dt.name_generator_api`. Override the property to point at an internal mirror if egress is restricted.
- **Reusable utilities.** `x_stave_dt.RegexUtils` and `x_stave_dt.Masker` are `public` script includes — other scoped apps can `new x_stave_dt.Masker().getBool(gr)` as a quick "would any mask rule fire here?" probe without taking on the full alert-creation path of `DataMaskerHelper.maskRecord`.
- **Dictionary-based masking.** `dictionary_table` on a mask definition can point at any table that has a `name` column (not strictly the shipped `x_stave_dt_dictionary_entry`) — useful for reusing existing taxonomy/CMDB lists as match sources.
- **Disabled scaffolding.** `Run Mask Definitions` on `live_message` ships **inactive** — re-enable to extend masking into Connect Chat / live messages.
