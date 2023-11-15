# `weidu_testing`.

This small mod is a test case to reveal a bug with PATCH_MATCH. See [Gibberlings Thread](https://www.gibberlings3.net/forums/topic/37350-possible-bug-in-weidu/) for more info.

## A. Description of the bug.

The bug is in the [function `set_item_field`](./weidu_testing/lib/items.tpa). The function is a simple PATCH_MATCH over field names, with each branch selecting a specific WRITE_* function. The bug is that for the string value "lore", the WRITE_SHORT branch should be taken, but instead the WRITE_LONG one is taken. This can be witnessed by strewning PRINT statements around and running weinstall to install the mod. The log shows:

```bash
copy_items: copying 'weidu_testing/resources/itm/magical_stone_a.itm' to 'override/magical_stone_a.itm'.

set_item_field: writing short '20' to field 'stack' at offset '56'.

set_item_field: writing long '0' to field 'lore' at offset '66'.

set_item_field: writing long '0' to field 'weight' at offset '76'.

set_item_field: writing long '1' to field 'enchantment' at offset '96'.
```

If you now comment out the block in the function [function `copy_items`](./weidu_testing/lib/installer.tpa) injecting variables in the local scope and replace the variables by the values in the respective calls, that is, change the function to:

```weidu
DEFINE_ACTION_FUNCTION copy_items BEGIN
    OUTER_TEXT_SPRINT dir "%MOD_FOLDER%/resources/itm"
    OUTER_TEXT_SPRINT resource "magical_stone_a"
    ////Inject names into current scope.
    //OUTER_TEXT_SPRINT stack "20"
    //OUTER_TEXT_SPRINT lore "0"
    //OUTER_TEXT_SPRINT weight "0"
    //OUTER_TEXT_SPRINT enchantment "1"

    COPY "%dir%/%resource%.itm" "override/%resource%.itm"
        PATCH_PRINT "copy_items: copying '%dir%/%resource%.itm' to 'override/%resource%.itm'."
        //Patch item.
        LPF set_item_field STR_VAR field = "stack" value = "20" END
        LPF set_item_field STR_VAR field = "lore" value = "0" END
        LPF set_item_field STR_VAR field = "weight" value = "0" END
        LPF set_item_field STR_VAR field = "enchantment" value = "1" END
END
```

You now get in the log,

```bash
copy_items: copying 'weidu_testing/resources/itm/magical_stone_a.itm' to 'override/magical_stone_a.itm'.

set_item_field: writing short '20' to field 'stack' at offset '56'.

set_item_field: writing short '0' to field 'lore' at offset '66'.

set_item_field: writing long '0' to field 'weight' at offset '76'.

set_item_field: writing long '1' to field 'enchantment' at offset '96'.
```

which is the correct behavior. So for some unfathomable reason, setting these local variables in the calling scope screws up the PATCH_MATCH in the called function.
