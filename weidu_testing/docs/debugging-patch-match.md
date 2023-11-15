# A. Initial set up.

With this setup, the relevant log section shows:

In order to make it easier to patch items, I coded a function:

```weidu
DEFINE_PATCH_FUNCTION set_item_field STR_VAR field = "" value = "" BEGIN
    //Lower-case field to avoid trivial mismatches.
    TO_LOWER field
    
    //Match on field to decide which write function to use.
    field_offset = $item_offsets("%field%")
    PATCH_MATCH "%field%" WITH
        "id_name"
        "name"
        "id_descr"
        "descr"
        "flags"
        "usability"
        "price"
        "weight"
        "enchantment" BEGIN
            PATCH_PRINT "set_item_field: writing long '%value%' to field '%field%' at offset '%field_offset%'."
            WRITE_LONG field_offset value
        END
        "strength_bonus"
        "intelligence"
        "constitution"
        "dexterity"
        "wisdom"
        "proficiency"
        "kit_usability1"
        "kit_usability2"
        "kit_usability3"
        "kit_usability4" BEGIN
            WRITE_BYTE field_offset value
        END
        "type"
        "level"
        "strength"
        "charisma"
        "stack"
        "lore" BEGIN
            PATCH_PRINT "set_item_field: writing short '%value%' to field '%field%' at offset '%field_offset%'."
            WRITE_SHORT field_offset value
        END
        "animation" BEGIN
            WRITE_EVALUATED_ASCII field_offset "%value%" #2
        END
        "replacement"
        "inventory_icon"
        "ground_icon"
        "description_icon" BEGIN
            WRITE_EVALUATED_ASCII field_offset "%value%" #8
        END
        DEFAULT
            PATCH_FAIL "set_item_field: field '%field%' is not a valid item field."
    END
END
```

The function is pretty simple: a PATCH_MATCH over the field to select the appropriate write function. Besides the arguments, the only input to the function is the array "item_offsets". The PATCH_PRINT statements were added for debugging. Now running install on a mod where this is is used, we get in the log (just looking at the relevant section):

```bash
copy_items_from_table: copying 'spell_overhaul/components/main/resources/itm/spells/ghoul_touch_a.itm' to 'override/ldwi218a.itm'.

set_item_field: writing long '13048' to field 'name' at offset '8'.

set_item_field: writing short '28' to field 'type' at offset '28'.

set_item_field: writing long '0' to field 'price' at offset '52'.

set_item_field: writing long '1' to field 'stack' at offset '56'.

set_item_field: writing short '10' to field 'lore' at offset '66'.

set_item_field: writing long '0' to field 'weight' at offset '76'.

set_item_field: writing long '1' to field 'enchantment' at offset '96'.
Copying and patching 1 file ...

copy_items_from_table: copying 'spell_overhaul/components/main/resources/itm/spells/magical_stone_a.itm' to 'override/ldpr106a.itm'.

set_item_field: writing long '12902' to field 'name' at offset '8'.

set_item_field: writing long '104228' to field 'descr' at offset '80'.

set_item_field: writing short '14' to field 'type' at offset '28'.

set_item_field: writing long '0' to field 'price' at offset '52'.

set_item_field: writing short '20' to field 'stack' at offset '56'.

set_item_field: writing long '0' to field 'lore' at offset '66'.

set_item_field: writing long '0' to field 'weight' at offset '76'.

set_item_field: writing long '1' to field 'enchantment' at offset '96'.
Copying 1 file ...
```

Let me home in on the two relevant lines:

```
set_item_field: writing short '10' to field 'lore' at offset '66'.

set_item_field: writing long '0' to field 'lore' at offset '66'.
```

As you can see in the second line, the `lore` field ended up in the long branch instead of the short. I do not see any possible world in which that could happen, outside of a bug in WeiDU with PATCH_MATCH, but maybe someone is smarter than me and can see how this could possibly happen. The problem is compounded by the fact that I can only trigger this apparent bug with the somewhat complicated setup of my mod; all my efforts to narrow it down to a more manageable test case have failed, suggesting that some really weird interaction is going on.

Any suggestions?

## B. Testing setup.

* In main, all files loaded until `%component_dir%/lib/spells/items.tpa`.
* Add print statements to `copy_items_from_table` before `COPY` and `set_item_field` on the relevant branches (long and short).
* Install only two items in `spells/items.2da` table.
* Changed lore to 10 on one of them (both have it at 0).
* You can further comment out all files except symbol_spells and bams.
* Trimming down even further to only copy the subspell and bam for ghoul touch, magical stone aux items.
* Added print statements to the other PATCH_MATCH branches.
* Dropped the patch `ghoul_touch_a`.
* Droppped icons on both remaining items. Dropped bams file installer.
* Dropped setting names and description on the two items.
* Dropped patches arg from `copy_items_from_table` call.
* Dropped tra arg from `copy_items_from_table` call.
* Dropped `copy_data_resource` call from `items.tpa` file.
* Dropped components library from `items.tpa` file.
* Imported items installer directly.

At this point managed to trim enough fat to transfer to a component of `weidu_library_testing`. This transfer also shaved off the intermediate `load_file` call.

* expanded variables on `copy_items_from_table` call.
* use copy of items installer file.
* replacing `weidu_library` imports by local imports one by one.
* started trimming down libraries: cre and spell validators dropped. Drop `creatures` library.
* Drop opcode validators. Drop enumeration validators.
* Drop functions from tables.
* `spells` library reduced to get_spell_res.
* Items library substantially reduced. Drop `opcodes` library. Dropped `spells` library.
* Moved all needed arrays to local arrays.
* Removed dependency from weidu_library.
* Moved to its own mod.

This still triggers the bug, so by now we have a self-contained mod, exposing the problem.

* Removed post-actions from `copy_items_from_table`.
* Removed patches and tra code from `copy_items_from_table`.
* Drop flags. Drop `bits` library.
* Drop higher-order functions form lists and arrays except `array_filter`.
* More simplifications of `lists` and `arrays` libraries.
* Moved `create_list_from_string` to arrays and dropped `lists` library.
* Dropped name, descr, icons, exclusion, patch, flags fields.

Changed values of weight and enchantment to *, and the `stack` field of `ghoul_touch_a.itm` _now_ writes correctly, although `lore` of `magical_stone_a` is still borked. So retract the change.

* Dropped `proficiency` and `type` fields and related code.
* Dropped all validators from name_validators except affix.
* Moved `create_list_from_string` to tables and dropped `INCLUDE arrays` from it.
* Set destination to `*` and dropped related code. Dropped `override` field and related code.
* Dropped `names` library.
* Dropped `destination` variable from `copy_items_from_table`.
* Dropped `install` variable and field from `copy_items_from_table`.
* Simplifed filter.


As soon as injection of local variables is dropped, things start working. Backtrack.

* Expand local variable injection.
* Use custom validator and drop filtering code.
* Move modules and resources to shorten their names.
* Dropped `components` library.
* Dropped `empty.tra` and related code in `copy_items_from_table`.
* Dropped `validators` library.
* Dropped `arrays` library.
* Renamed `item` to `resource` in `copy_items_from_table`.
* Unrolled `alter_item_with_array` and dropped related code.
* Dropped `price` field.
* Dropped tests to a single item.
* Drop `load_table` and `get_row_from_table` code.
* Inlined array assignment.

At this point, the cutting down to a usable test case is essentially done -- there is a weird interaction with setting variables in the calling scope and PATCH_MATCH that triggers the bug.