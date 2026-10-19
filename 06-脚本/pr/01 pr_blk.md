
```tcl

# ============================================================
# pr_blk : Selected Macro Gap Blockage Utility
#
# Only create placement blockages between currently selected
# hard macros.
#
# Usage:
#
#   pr_blk <gap> soft
#   pr_blk <gap> hard
#   pr_blk <gap> part <blocked_percentage>
#
# Examples:
#
#   pr_blk 3 soft
#   pr_blk 3 hard
#   pr_blk 3 part 70
#   pr_blk 35 part 50
#
# Match condition:
#   1. Both objects are selected hard macros
#   2. Horizontal edge-to-edge gap > <gap>
#   3. Two macros overlap in Y direction
#   4. Adjacent only: no other selected macro lies inside
#      the target region (blockages never overlap or cover
#      an intermediate macro)
#
# Note:
#   Before creating a new blockage, any existing placement
#   blockage whose bbox overlaps the target region is removed
#   first (replace behavior).
# ============================================================


# ============================================================
# pr_blk : Selected Macro Gap Blockage Utility
#
# Only create placement blockages between currently selected
# hard macros.
#
# Match condition:
#   1. Both objects are selected hard macros
#   2. Horizontal edge-to-edge gap > <gap>
#   3. Two macros overlap in Y direction
#   4. Adjacent only: no other selected macro lies inside
#      the target region
# ============================================================

proc pr_blk {target_gap type {blocked_pct ""}} {

    # ========================================================
    # 1. Check target gap
    # ========================================================

    if {![string is double -strict $target_gap]} {

        puts ""
        puts "ERROR: gap must be a number."
        puts ""
        puts "Examples:"
        puts "  pr_blk 3 soft"
        puts "  pr_blk 35 part 50"
        puts ""

        return
    }


    if {$target_gap < 0} {

        puts ""
        puts "ERROR: gap must be >= 0."
        puts ""

        return
    }


    # ========================================================
    # 2. Normalize blockage type
    # ========================================================

    set type [string tolower $type]

    # allow shorthand:
    #
    #   part
    #
    # internally convert to:
    #
    #   partial

    if {$type eq "part"} {
        set type "partial"
    }


    # ========================================================
    # 3. Check blockage type
    # ========================================================

    if {[lsearch -exact {soft hard partial} $type] < 0} {

        puts ""
        puts "ERROR: unsupported blockage type: $type"
        puts ""
        puts "Supported:"
        puts ""
        puts "  soft"
        puts "  hard"
        puts "  part"
        puts "  partial"
        puts ""
        puts "Examples:"
        puts ""
        puts "  pr_blk 3 soft"
        puts "  pr_blk 3 hard"
        puts "  pr_blk 3 part 70"
        puts ""

        return
    }


    # ========================================================
    # 4. Check partial percentage
    # ========================================================

    if {$type eq "partial"} {

        if {$blocked_pct eq ""} {

            puts ""
            puts "ERROR: partial blockage needs blocked_percentage."
            puts ""
            puts "Example:"
            puts ""
            puts "  pr_blk 35 part 50"
            puts ""

            return
        }


        if {![string is double -strict $blocked_pct]} {

            puts ""
            puts "ERROR: blocked_percentage must be a number."
            puts ""

            return
        }


        if {$blocked_pct < 0 || $blocked_pct > 100} {

            puts ""
            puts "ERROR: blocked_percentage must be between 0 and 100."
            puts ""

            return
        }

    } else {

        if {$blocked_pct ne ""} {

            puts ""
            puts "WARN: blocked_percentage ignored for type=$type"
            puts ""
        }
    }


    # ========================================================
    # 5. Get current GUI selection
    # ========================================================

    set selected_objects [get_selection]

    set selected_count [sizeof_collection $selected_objects]


    if {$selected_count == 0} {

        puts ""
        puts "ERROR: no GUI objects selected."
        puts ""
        puts "Please select at least 2 macros in GUI first."
        puts ""
        puts "NOTE:"
        puts "  RulerTool / Measure Tool does NOT select macros."
        puts ""
        puts "Check selection with:"
        puts ""
        puts "  sizeof_collection \[get_selection\]"
        puts "  get_object_name \[get_selection\]"
        puts ""

        return
    }


    # ========================================================
    # 6. Filter selected hard macros
    #
    # macro_list format:
    #
    #   {name llx lly urx ury}
    #
    # Important:
    #
    # GUI selection may contain:
    #
    #   cell
    #   placement_blockage
    #   net
    #   pin
    #   shape
    #
    # Only cell + is_hard_macro=true is accepted.
    # ========================================================

    set macro_list {}

    set skipped_count 0


    foreach_in_collection obj $selected_objects {

        set obj_name [get_object_name $obj]


        # ----------------------------------------------------
        # Get object class safely
        # ----------------------------------------------------

        set obj_class ""

        if {[catch {
            set obj_class [get_attribute $obj object_class]
        } err]} {

            puts "SKIP: $obj_name"
            puts "      cannot determine object_class"

            incr skipped_count

            continue
        }


        # ----------------------------------------------------
        # Only CELL objects can be hard macros
        # ----------------------------------------------------

        if {$obj_class ne "cell"} {

            puts "SKIP: $obj_name"
            puts "      class = $obj_class"

            incr skipped_count

            continue
        }


        # ----------------------------------------------------
        # Check is_hard_macro safely
        # ----------------------------------------------------

        set is_macro ""

        if {[catch {
            set is_macro [get_attribute $obj is_hard_macro]
        } err]} {

            puts "SKIP: $obj_name"
            puts "      cannot read is_hard_macro"

            incr skipped_count

            continue
        }


        # Do NOT directly use:
        #
        #   if {!$is_macro}
        #
        # Avoid boolean conversion errors.
        # ----------------------------------------------------

        if {![string equal -nocase $is_macro "true"] &&
            $is_macro ne "1"} {

            puts "SKIP: $obj_name"
            puts "      cell but not hard macro"

            incr skipped_count

            continue
        }


        # ----------------------------------------------------
        # Get bbox safely
        # ----------------------------------------------------

        set bbox ""

        if {[catch {
            set bbox [get_attribute $obj bbox]
        } err]} {

            puts "SKIP: $obj_name"
            puts "      cannot read bbox"

            incr skipped_count

            continue
        }


        if {[llength $bbox] != 2} {

            puts "SKIP: $obj_name"
            puts "      invalid bbox = $bbox"

            incr skipped_count

            continue
        }


        # ----------------------------------------------------
        # Extract bbox
        # ----------------------------------------------------

        set ll [lindex $bbox 0]
        set ur [lindex $bbox 1]

        set llx [lindex $ll 0]
        set lly [lindex $ll 1]

        set urx [lindex $ur 0]
        set ury [lindex $ur 1]


        # ----------------------------------------------------
        # Add valid macro
        # ----------------------------------------------------

        lappend macro_list \
            [list \
                $obj_name \
                $llx \
                $lly \
                $urx \
                $ury \
            ]
    }


    # ========================================================
    # 7. Check valid macro count
    # ========================================================

    set macro_count [llength $macro_list]


    if {$macro_count < 2} {

        puts ""
        puts "============================================================"
        puts " ERROR"
        puts "============================================================"
        puts ""
        puts " Selected objects    : $selected_count"
        puts " Selected hard macro : $macro_count"
        puts " Skipped objects     : $skipped_count"
        puts ""
        puts "Need at least 2 selected hard macros."
        puts ""
        puts "============================================================"
        puts ""

        return
    }


    # ========================================================
    # 8. Print run information
    # ========================================================

    puts ""
    puts "============================================================"
    puts " Selected Macro Gap Blockage"
    puts "============================================================"
    puts ""
    puts " Selected objects : $selected_count"
    puts " Hard macros      : $macro_count"
    puts " Skipped objects  : $skipped_count"
    puts ""
    puts " Target gap       : gap > $target_gap um"
    puts " Type             : $type"


    if {$type eq "partial"} {

        puts " Blocked          : ${blocked_pct}%"
        puts " Available        : [expr {100.0 - $blocked_pct}]%"
    }


    puts ""
    puts "============================================================"
    puts ""


    # ========================================================
    # 9. Print valid macro list
    # ========================================================

    puts "Selected HARD macros:"
    puts ""


    foreach m $macro_list {

        puts "  [lindex $m 0]"
    }


    puts ""


    # ========================================================
    # 10. Search selected macro pairs
    # ========================================================

    set blk_id 0


    # Used to avoid blockage name conflict between runs

    set run_tag [clock milliseconds]


    # ----------------------------------------------------
    # Query existing placement blockages once.
    #
    # Used to remove blockages overlapping the target region
    # before creating a new one (replace behavior).
    # ----------------------------------------------------

    set existing_blks {}

    if {[catch {
        set existing_blks [get_placement_blockages]
    } err]} {
        puts "WARN: cannot query existing placement blockages."
        puts "      skip remove-existing step."
    }

    # Names of blockages already removed (persists across pairs,
    # so one blockage is never removed twice)

    set removed_list {}


    for {set i 0} {$i < $macro_count} {incr i} {


        # ----------------------------------------------------
        # Macro A
        # ----------------------------------------------------

        set A [lindex $macro_list $i]

        set A_name [lindex $A 0]

        set A_llx [lindex $A 1]
        set A_lly [lindex $A 2]

        set A_urx [lindex $A 3]
        set A_ury [lindex $A 4]


        # ----------------------------------------------------
        # Compare A with B
        # ----------------------------------------------------

        for {set j [expr {$i + 1}]} \
            {$j < $macro_count} \
            {incr j} {


            set B [lindex $macro_list $j]

            set B_name [lindex $B 0]

            set B_llx [lindex $B 1]
            set B_lly [lindex $B 2]

            set B_urx [lindex $B 3]
            set B_ury [lindex $B 4]


            # =================================================
            # 11. Determine LEFT / RIGHT macro
            # =================================================

            if {$A_llx <= $B_llx} {

                set L_name $A_name

                set L_llx $A_llx
                set L_lly $A_lly

                set L_urx $A_urx
                set L_ury $A_ury


                set R_name $B_name

                set R_llx $B_llx
                set R_lly $B_lly

                set R_urx $B_urx
                set R_ury $B_ury

            } else {

                set L_name $B_name

                set L_llx $B_llx
                set L_lly $B_lly

                set L_urx $B_urx
                set L_ury $B_ury


                set R_name $A_name

                set R_llx $A_llx
                set R_lly $A_lly

                set R_urx $A_urx
                set R_ury $A_ury
            }


            # =================================================
            # 12. Calculate horizontal edge-to-edge gap
            #
            #
            #       target_gap
            #
            #       <------->
            #
            # +---------+      +---------+
            # | LEFT    |      | RIGHT   |
            # | macro   |      | macro   |
            # +---------+      +---------+
            #
            #          ^        ^
            #          |        |
            #        L_urx    R_llx
            #
            # gap = R_llx - L_urx
            #
            # =================================================

            set gap [expr {$R_llx - $L_urx}]

            # -------------------------------------------------
            # Negative gap means macros overlap.
            # Never block an overlapping pair.
            # -------------------------------------------------

            if {$gap < 0} {

                continue
            }


            # -------------------------------------------------
            # Only block pairs whose gap is strictly greater
            # than the requested gap.
            # -------------------------------------------------

            if {$gap <= $target_gap} {

                continue
            }


            # =================================================
            # 13. Calculate Y overlap
            # =================================================

            set y1 [expr {max($L_lly, $R_lly)}]

            set y2 [expr {min($L_ury, $R_ury)}]


            # -------------------------------------------------
            # No vertical overlap
            # -------------------------------------------------

            if {$y2 <= $y1} {

                continue
            }


            # =================================================
            # 13.5 Skip non-adjacent pairs
            #
            #     If another selected macro lies inside the
            #     target region [L_urx, R_llx] x [y1, y2],
            #     this pair is not adjacent -> skip.
            #
            #     Otherwise larger-span pairs (e.g. M1-M3)
            #     would create blockages overlapping the
            #     intermediate macro itself.
            # =================================================

            set adjacent 1

            foreach K $macro_list {

                set K_name [lindex $K 0]

                # Skip L and R themselves
                if {$K_name eq $L_name || $K_name eq $R_name} {
                    continue
                }

                set K_llx [lindex $K 1]
                set K_lly [lindex $K 2]
                set K_urx [lindex $K 3]
                set K_ury [lindex $K 4]

                # K intersects the target region?
                if {[expr {min($R_llx, $K_urx)}] > [expr {max($L_urx, $K_llx)}] &&
                    [expr {min($y2, $K_ury)}] > [expr {max($y1, $K_lly)}]} {

                    set adjacent 0
                    break
                }
            }

            if {!$adjacent} {

                continue
            }


            # =================================================
            # 14. Valid pair found
            # =================================================

            incr blk_id


            # -------------------------------------------------
            # Generate name
            #
            # Convert:
            #
            #   3.0 -> 3p0
            #   35  -> 35
            #
            # -------------------------------------------------

            set gap_tag [string map {. p} $target_gap]

            set type_tag [string toupper $type]


            set blk_name \
                [format \
                    "B2B_%s_G%s_%s_%03d" \
                    $type_tag \
                    $gap_tag \
                    $run_tag \
                    $blk_id \
                ]


            # =================================================
            # 15. Blockage boundary
            # =================================================

            set blk_boundary \
                [list \
                    [list $L_urx $y1] \
                    [list $R_llx $y2] \
                ]


            # =================================================
            # 16. Print pair information
            # =================================================

            puts "PAIR $blk_id"
            puts ""
            puts "  Left macro  : $L_name"
            puts "  Right macro : $R_name"
            puts ""
            puts "  X gap       : $gap um"
            puts "  Y overlap   : [expr {$y2 - $y1}] um"
            puts ""
            puts "  Boundary    : $blk_boundary"
            puts "  Type        : $type"
            puts "  Name        : $blk_name"


            if {$type eq "partial"} {

                puts "  Blocked     : ${blocked_pct}%"
            }


            puts ""


            # =================================================
            # 17. Remove existing blockages overlapping the
            #     target region (replace behavior)
            #
            #     Target region:
            #       X : [L_urx, R_llx]
            #       Y : [y1, y2]
            #
            #     Any existing blockage whose bbox overlaps
            #     this region is removed first.
            # =================================================

            foreach_in_collection blk $existing_blks {

                set old_name [get_object_name $blk]
                set old_bbox [get_attribute $blk bbox]

                if {$old_bbox eq "" || [llength $old_bbox] != 2} {
                    continue
                }

                # Already removed by a previous pair
                if {$old_name in $removed_list} {
                    continue
                }

                set blx  [lindex $old_bbox 0 0]
                set bly  [lindex $old_bbox 0 1]
                set burx [lindex $old_bbox 1 0]
                set bury [lindex $old_bbox 1 1]

                if {[expr {min($R_llx, $burx)}] > [expr {max($L_urx, $blx)}] &&
                    [expr {min($y2, $bury)}] > [expr {max($y1, $bly)}]} {

                    lappend removed_list $old_name

                    remove_placement_blockages $blk

                    puts "  REMOVED existing: $old_name"
                }
            }

            puts ""


            # =================================================
            # 18. Create placement blockage
            # =================================================

            if {$type eq "partial"} {

                create_placement_blockage \
                    -boundary $blk_boundary \
                    -type partial \
                    -blocked_percentage $blocked_pct \
                    -name $blk_name

            } else {

                create_placement_blockage \
                    -boundary $blk_boundary \
                    -type $type \
                    -name $blk_name
            }


            puts "  --> CREATED"
            puts ""
            puts "------------------------------------------------------------"
            puts ""
        }
    }


    # ========================================================
    # 19. Summary
    # ========================================================

    puts ""
    puts "============================================================"
    puts " DONE"
    puts "============================================================"
    puts ""
    puts " Selected objects : $selected_count"
    puts " Hard macros      : $macro_count"
    puts " Target gap       : gap > $target_gap um"
    puts " Type             : $type"
    puts " Created          : $blk_id"


    if {$type eq "partial"} {

        puts " Blocked          : ${blocked_pct}%"
    }


    puts ""
    puts "============================================================"
    puts ""


    # ========================================================
    # 20. Nothing found
    # ========================================================

    if {$blk_id == 0} {

        puts "WARN: no matching macro pairs found."
        puts ""
        puts "Required conditions:"
        puts ""
        puts "  X gap > $target_gap um"
        puts "  Y overlap > 0"
        puts "  Adjacent pair only"
        puts ""
        puts "Only selected HARD macros are checked."
        puts ""
    }
}

```
