
```tcl
# ============================================================
# pr_space_1 : <gap>
#
# Example:
#   pr_space_1 3
#
# Behavior:
#   1. 只处理 FC GUI 当前选中的 hard macro
#   2. 按 Y 坐标自动分行
#   3. 每行以当前最右侧 macro 为基准
#   4. 最右侧 macro 的 orientation 保持不变
#   5. 其余 macro 全部复制最右侧 macro 的 orientation
#   6. 从右向左按照统一 gap 摆放
#
# Usage:
#   先手动把最右侧 macro 调成希望的方向
#   再框选整排 macro
#
#   pr_space_1 2
# ============================================================

proc pr_space_1 {MACRO_GAP} {

    # -------------------- 固定参数 ------------------------------------------
    set ROW_Y_TOLERANCE 12.0
    set APPLY_PLACEMENT 1

    # -------------------- 参数检查 ------------------------------------------
    if {![string is double -strict $MACRO_GAP]} {
        error "MACRO_GAP 必须是数字，例如: pr_space_1 3"
    }

    if {$MACRO_GAP < 0} {
        error "MACRO_GAP 不能小于 0"
    }

    # -------------------- 获取 GUI 当前选中的 macro --------------------------
    set selected_objects [get_selection]

    if {[sizeof_collection $selected_objects] == 0} {
        error "没有选中对象，请先在 GUI 中框选需要调整的 macro"
    }

    set selected_macros [get_cells -quiet $selected_objects \
        -filter "is_hard_macro == true"]

    set selected_count [sizeof_collection $selected_macros]

    if {$selected_count == 0} {
        error "当前选中的对象中没有 hard macro"
    }

    puts ""
    puts "============================================================"
    puts "INFO: pr_space_1"
    puts "INFO: selected macro count = $selected_count"
    puts "INFO: gap = $MACRO_GAP"
    puts "============================================================"

    # -------------------- 按顶边 Y 坐标分行 ---------------------------------
    set row_dict [dict create]

    foreach_in_collection selected_object $selected_macros {

        set macro [get_object_name $selected_object]
        set macro_bbox [get_attribute $selected_object bbox]

        set macro_x   [lindex [lindex $macro_bbox 0] 0]
        set macro_lly [lindex [lindex $macro_bbox 0] 1]
        set macro_ury [lindex [lindex $macro_bbox 1] 1]

        set matched_row_y ""

        dict for {existing_row_y existing_row_macros} $row_dict {

            if {[expr {abs($macro_ury - $existing_row_y)}] \
                    <= $ROW_Y_TOLERANCE} {

                set matched_row_y $existing_row_y
                break
            }
        }

        if {$matched_row_y eq ""} {
            set matched_row_y $macro_ury
        }

        dict lappend row_dict \
            $matched_row_y \
            [list $macro_x $macro_lly $macro]
    }

    # -------------------- 每行独立处理 --------------------------------------
    dict for {row_y row_macros} $row_dict {

        # X 从大到小：最右 -> 最左
        set row_macros \
            [lsort -decreasing -real -index 0 $row_macros]

        puts ""
        puts "INFO: processing row Y=$row_y"

        # --------------------------------------------------------------------
        # 最右侧 macro 作为 anchor
        # --------------------------------------------------------------------
        set anchor_info [lindex $row_macros 0]

        set anchor_x     [lindex $anchor_info 0]
        set anchor_y     [lindex $anchor_info 1]
        set anchor_macro [lindex $anchor_info 2]

        # 读取最右侧 macro 当前 orientation
        set anchor_orientation \
            [get_attribute [get_cells $anchor_macro] orientation]

        puts "INFO: anchor macro       = $anchor_macro"
        puts "INFO: anchor orientation = $anchor_orientation"
        puts "INFO: anchor position    = {$anchor_x $anchor_y}"

        # --------------------------------------------------------------------
        # current_x 表示上一颗已经放好的 macro 左边界
        # --------------------------------------------------------------------
        set current_x $anchor_x
        set i 0

        foreach macro_info $row_macros {

            set old_x [lindex $macro_info 0]
            set old_y [lindex $macro_info 1]
            set macro [lindex $macro_info 2]

            # --------------------------------------------------------------
            # 所有 macro 都使用 anchor 的 orientation
            # --------------------------------------------------------------
            if {$APPLY_PLACEMENT} {

                set_attribute \
                    [get_cells $macro] \
                    orientation \
                    $anchor_orientation
            }

            # 改 orientation 后重新读取 bbox
            set macro_bbox \
                [get_attribute [get_cells $macro] bbox]

            set current_width \
                [expr {
                    [lindex [lindex $macro_bbox 1] 0] -
                    [lindex [lindex $macro_bbox 0] 0]
                }]

            if {$i == 0} {

                # anchor 本身不移动
                set target_x $anchor_x
                set gap 0.0

            } else {

                set gap $MACRO_GAP

                set target_x \
                    [expr {
                        $current_x -
                        $current_width -
                        $MACRO_GAP
                    }]
            }

            puts [format \
                "INFO: %-50s i=%-3d orient=%-5s gap=%6.3f target={%.3f %.3f}" \
                $macro \
                $i \
                $anchor_orientation \
                $gap \
                $target_x \
                $old_y]

            if {$APPLY_PLACEMENT && $i != 0} {

                move_objects \
                    [get_cells $macro] \
                    -to "$target_x $old_y"
            }

            # 当前 macro 左边界作为下一颗的基准
            set current_x $target_x

            incr i
        }
    }

    puts ""
    puts "============================================================"
    puts "INFO: selected macros have been placed"
    puts "INFO: all macros follow rightmost macro orientation"
    puts "INFO: gap = $MACRO_GAP"
    puts "============================================================"
}


# ============================================================
# pr_space_2 : <pin2pin_gap> <back2back_gap>
#
# Example:
#   pr_space_2 16 1
#
# Behavior:
#   1. 只处理 FC GUI 当前选中的 macro
#   2. 按 macro 顶边 Y 坐标分行
#   3. 每行从当前最右侧 macro 向左摆放
#   4. 最右侧为 R180，向左依次 MX / R180 / MX ...
#   5. 相邻间距：
#        R180 -> MX   : pin-to-pin gap
#        MX   -> R180 : back-to-back gap
# ============================================================

proc pr_space_2 {PIN_TO_PIN_GAP BACK_TO_BACK_GAP} {

    # -------------------- 固定参数 ------------------------------------------
    set ROW_Y_TOLERANCE 12.0
    set APPLY_PLACEMENT 1

    # -------------------- 参数检查 ------------------------------------------
    if {![string is double -strict $PIN_TO_PIN_GAP]} {
        error "PIN_TO_PIN_GAP 必须是数字，例如: pr_space_2 16 1"
    }

    if {![string is double -strict $BACK_TO_BACK_GAP]} {
        error "BACK_TO_BACK_GAP 必须是数字，例如: pr_space_2 16 1"
    }

    if {$PIN_TO_PIN_GAP < 0} {
        error "PIN_TO_PIN_GAP 不能小于 0"
    }

    if {$BACK_TO_BACK_GAP < 0} {
        error "BACK_TO_BACK_GAP 不能小于 0"
    }

    puts ""
    puts "============================================================"
    puts "INFO: pr_space_2"
    puts "INFO: PIN_TO_PIN_GAP   = $PIN_TO_PIN_GAP"
    puts "INFO: BACK_TO_BACK_GAP = $BACK_TO_BACK_GAP"
    puts "INFO: ROW_Y_TOLERANCE  = $ROW_Y_TOLERANCE"
    puts "============================================================"

    # -------------------- 获取当前 GUI 选中对象 -----------------------------
    set selected_objects [get_selection]

    if {[sizeof_collection $selected_objects] == 0} {
        error "没有选中对象，请先在 GUI 中框选需要调整的 macro"
    }

    # 只保留 hard macro
    set selected_macros [get_cells -quiet $selected_objects \
        -filter "is_hard_macro == true"]

    set selected_count [sizeof_collection $selected_macros]

    if {$selected_count == 0} {
        error "当前选中的对象中没有 hard macro"
    }

    puts "INFO: selected macro count = $selected_count"

    # -------------------- 按顶边 Y 坐标容差分行 -----------------------------
    set row_dict [dict create]

    foreach_in_collection selected_object $selected_macros {

        set macro [get_object_name $selected_object]

        set macro_bbox [get_attribute $selected_object bbox]

        set macro_x   [lindex [lindex $macro_bbox 0] 0]
        set macro_lly [lindex [lindex $macro_bbox 0] 1]
        set macro_ury [lindex [lindex $macro_bbox 1] 1]

        set matched_row_y ""

        dict for {existing_row_y existing_row_macros} $row_dict {

            if {[expr {abs($macro_ury - $existing_row_y)}] \
                    <= $ROW_Y_TOLERANCE} {

                set matched_row_y $existing_row_y
                break
            }
        }

        if {$matched_row_y eq ""} {
            set matched_row_y $macro_ury
        }

        dict lappend row_dict \
            $matched_row_y \
            [list $macro_x $macro_lly $macro]
    }

    # -------------------- 每行从右向左摆放 ---------------------------------
    dict for {row_y row_macros} $row_dict {

        # X 越大越靠右
        set row_macros \
            [lsort -decreasing -real -index 0 $row_macros]

        puts ""
        puts "INFO: processing row Y=$row_y"

        set i 0
        set current_x 0.0

        foreach macro_info $row_macros {

            set old_x [lindex $macro_info 0]
            set old_y [lindex $macro_info 1]
            set macro [lindex $macro_info 2]

            # --------------------------------------------------------------
            # Orientation:
            #
            # index 0 : R180
            # index 1 : MX
            # index 2 : R180
            # index 3 : MX
            # --------------------------------------------------------------
            if {$i % 2 == 0} {
                set current_orientation R180
            } else {
                set current_orientation MX
            }

            # 先改 orientation
            if {$APPLY_PLACEMENT} {
                set_attribute \
                    [get_cells $macro] \
                    orientation \
                    $current_orientation
            }

            # 获取当前 orientation 下 bbox / width
            set macro_bbox \
                [get_attribute [get_cells $macro] bbox]

            set current_width \
                [expr {
                    [lindex [lindex $macro_bbox 1] 0] -
                    [lindex [lindex $macro_bbox 0] 0]
                }]

            # --------------------------------------------------------------
            # 最右侧 macro 保持原 X 不动
            #
            # 后续向左：
            #
            # i=1 : pin-to-pin
            # i=2 : back-to-back
            # i=3 : pin-to-pin
            # i=4 : back-to-back
            # --------------------------------------------------------------
            if {$i == 0} {

                set MACRO_GAP_X 0.0
                set current_x $old_x

            } elseif {$i % 2 == 1} {

                set MACRO_GAP_X $PIN_TO_PIN_GAP

                set current_x \
                    [expr {
                        $current_x -
                        $current_width -
                        $MACRO_GAP_X
                    }]

            } else {

                set MACRO_GAP_X $BACK_TO_BACK_GAP

                set current_x \
                    [expr {
                        $current_x -
                        $current_width -
                        $MACRO_GAP_X
                    }]
            }

            puts [format \
                "INFO: %-50s i=%-3d orient=%-4s gap=%6.3f target={%.3f %.3f}" \
                $macro \
                $i \
                $current_orientation \
                $MACRO_GAP_X \
                $current_x \
                $old_y]

            if {$APPLY_PLACEMENT} {

                move_objects \
                    [get_cells $macro] \
                    -to "$current_x $old_y"
            }

            incr i
        }
    }

    puts ""
    puts "============================================================"
    puts "INFO: selected macros have been placed"
    puts "INFO: pin2pin=$PIN_TO_PIN_GAP  back2back=$BACK_TO_BACK_GAP"
    puts "============================================================"
}

```

