
```tcl
# ============================================================
# pr_split_blk : Split selected placement blockage into
#                10x10 small blockages
#
#   <seq> : 序号 (1/2/3 ...),嵌入新 blockage 名字中。
#           每次运行传不同的 seq,避免与上一次运行生成的名字冲突,
#           从而可以多次对不同的 blockage 打散。
#           缺省为 1。
#
#   1. 对 GUI 当前选中的 placement blockage 进行切分
#   2. 正常块大小：10 x 10
#   3. 最右侧不足 10：自动变成 剩余宽度 x 10
#   4. 最上侧不足 10：自动变成 10 x 剩余高度
#   5. 右上角两边都不足时：剩余宽度 x 剩余高度
#   6. 保留原 blockage 的 hard / soft / partial 类型
#   7. partial blockage 保留 blocked_percentage
#   8. 切分完成后自动删除原 blockage
#      (仅当切分数量与预期一致时才删除,防止切一半丢失原块)
# ============================================================

proc pr_split_blk {{seq 1}} {

    # --------------------------------------------------------
    # Tile size settings
    # --------------------------------------------------------

    set TILE_W 10.0
    set TILE_H 10.0

    # 防止因为浮点误差生成极小 blockage
    set EPS 0.000001


    # --------------------------------------------------------
    # Check seq
    # --------------------------------------------------------

    if {![string is integer -strict $seq] || $seq < 1} {

        puts ""
        puts "ERROR: seq must be a positive integer."
        puts ""
        puts "Examples:"
        puts "  pr_split_blk 1"
        puts "  pr_split_blk 2"
        puts "  pr_split_blk 3"
        puts ""
        return
    }


    # --------------------------------------------------------
    # Get current GUI selection
    # --------------------------------------------------------

    set sel [get_selection]

    if {[sizeof_collection $sel] == 0} {
        puts "ERROR: No object selected."
        return
    }

    puts ""
    puts "============================================================"
    puts " Split Placement Blockage"
    puts " TILE_W = $TILE_W"
    puts " TILE_H = $TILE_H"
    puts "============================================================"
    puts ""


    # --------------------------------------------------------
    # Process selected objects
    # --------------------------------------------------------

    set blk_idx 0
    set total_created 0

    foreach_in_collection blk $sel {

        incr blk_idx

        # ----------------------------------------------------
        # Get blockage information
        # ----------------------------------------------------

        set blk_name [get_object_name $blk]
        set bbox     [get_attribute $blk bbox]

        if {$bbox eq ""} {
            puts "WARNING: $blk_name has no bbox. Skip."
            continue
        }

        # 强制转换成 double
        set llx [expr {double([lindex $bbox 0 0])}]
        set lly [expr {double([lindex $bbox 0 1])}]
        set urx [expr {double([lindex $bbox 1 0])}]
        set ury [expr {double([lindex $bbox 1 1])}]

        # ----------------------------------------------------
        # Check bbox
        # ----------------------------------------------------

        set width  [expr {$urx - $llx}]
        set height [expr {$ury - $lly}]

        if {$width <= $EPS || $height <= $EPS} {
            puts "WARNING: Invalid bbox for $blk_name"
            puts "         bbox = $bbox"
            puts "         width = $width"
            puts "         height = $height"
            puts "         Skip."
            continue
        }

        # ----------------------------------------------------
        # Get blockage type
        # ----------------------------------------------------

        set blk_type ""

        if {[catch {
            set blk_type [get_attribute $blk blockage_type]
        } err]} {
            puts "WARNING: Cannot get blockage_type from $blk_name. Skip."
            continue
        }

        if {$blk_type eq ""} {
            puts "WARNING: blockage_type is empty for $blk_name. Skip."
            continue
        }

        # partial blockage percentage
        set blocked_percentage ""

        if {$blk_type eq "partial"} {

            if {[catch {
                set blocked_percentage \
                    [get_attribute $blk blocked_percentage]
            } err]} {
                puts "WARNING: Cannot get blocked_percentage from $blk_name. Skip."
                continue
            }
        }


        # ----------------------------------------------------
        # Calculate number of columns / rows
        #
        # 例如：
        # width = 999.475
        #
        # ceil(999.475 / 10)
        # = 100 columns
        # ----------------------------------------------------

        set num_cols [expr {int(ceil($width  / $TILE_W))}]
        set num_rows [expr {int(ceil($height / $TILE_H))}]

        set num_total [expr {$num_cols * $num_rows}]

        puts "------------------------------------------------------------"
        puts "Original blockage : $blk_name"
        puts "Type              : $blk_type"

        if {$blk_type eq "partial"} {
            puts "Blocked percentage: $blocked_percentage"
        }

        puts "BBox              : {{$llx $lly} {$urx $ury}}"
        puts "Width             : $width"
        puts "Height            : $height"
        puts "Columns           : $num_cols"
        puts "Rows              : $num_rows"
        puts "New blockages     : $num_total"
        puts "------------------------------------------------------------"


        # ----------------------------------------------------
        # Create small blockages
        # ----------------------------------------------------

        set created_this_blk 0

        for {set row 0} {$row < $num_rows} {incr row} {

            # 当前 row 的 Y 坐标
            set y1 [expr {$lly + $row * $TILE_H}]
            set y2 [expr {$lly + ($row + 1) * $TILE_H}]

            # 最后一排不能超过原 blockage
            if {$y2 > $ury} {
                set y2 $ury
            }

            # 防止浮点误差导致非法高度
            if {[expr {$y2 - $y1}] <= $EPS} {
                continue
            }


            for {set col 0} {$col < $num_cols} {incr col} {

                # 当前 column 的 X 坐标
                set x1 [expr {$llx + $col * $TILE_W}]
                set x2 [expr {$llx + ($col + 1) * $TILE_W}]

                # 最后一列不能超过原 blockage
                if {$x2 > $urx} {
                    set x2 $urx
                }

                # 防止浮点误差导致非法宽度
                if {[expr {$x2 - $x1}] <= $EPS} {
                    continue
                }


                # --------------------------------------------
                # Final safety check
                # --------------------------------------------

                if {$x2 <= $x1} {
                    puts "ERROR: Invalid X:"
                    puts "       x1 = $x1"
                    puts "       x2 = $x2"
                    continue
                }

                if {$y2 <= $y1} {
                    puts "ERROR: Invalid Y:"
                    puts "       y1 = $y1"
                    puts "       y2 = $y2"
                    continue
                }


                # --------------------------------------------
                # New blockage name
                # --------------------------------------------

                set new_name \
                    [format "SPLIT_BLK_%d_%03d_%03d_%03d" \
                        $seq $blk_idx $row $col]


                # --------------------------------------------
                # New boundary
                # --------------------------------------------

                set new_bbox [list \
                    [list $x1 $y1] \
                    [list $x2 $y2]]


                # --------------------------------------------
                # Create blockage
                # --------------------------------------------

                if {$blk_type eq "partial"} {

                    create_placement_blockage \
                        -name $new_name \
                        -type partial \
                        -blocked_percentage $blocked_percentage \
                        -boundary $new_bbox

                } else {

                    create_placement_blockage \
                        -name $new_name \
                        -type $blk_type \
                        -boundary $new_bbox
                }


                incr created_this_blk
                incr total_created
            }
        }


        # ----------------------------------------------------
        # Result check
        # ----------------------------------------------------

        puts ""
        puts "INFO: $blk_name"
        puts "      Expected = $num_total"
        puts "      Created  = $created_this_blk"


        # ----------------------------------------------------
        # Remove original blockage
        # 只有创建数量符合预期才删除，避免切一半把原 blockage 删了
        # ----------------------------------------------------

        if {$created_this_blk == $num_total} {

            puts "INFO: Removing original blockage: $blk_name"

            remove_placement_blockages $blk

        } else {

            puts "WARNING: Created count != expected count."
            puts "         Original blockage will NOT be removed."
        }

        puts ""
    }


    # --------------------------------------------------------
    # Finish
    # --------------------------------------------------------

    puts "============================================================"
    puts "INFO: Split blockage finished."
    puts "INFO: Total created = $total_created"
    puts "============================================================"
}

```
