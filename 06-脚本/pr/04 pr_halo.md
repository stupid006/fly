
```tcl
########################################################################
# Macro Halo Placement Blockage
#
# Usage:
#
#   pr_halo L B R T -type hard
#   pr_halo L B R T -type soft
#   pr_halo L B R T -type part PERCENT
#
# Example:
#
#   pr_halo 4 0 4 0 -type hard
#   pr_halo 3 3 3 3 -type soft
#   pr_halo 3 3 3 3 -type part 70
#
#
# Rules:
#
#   SKIP_GAP = 1.1
#
#   macro gap <= 1.1:
#       认为属于同一组
#       中间不加 blockage
#
#   hard:
#       不打散
#
#   soft / partial:
#       最大切成 10 x 10
########################################################################


########################################################################
# 一维区间是否有重叠
########################################################################
proc _mb_range_overlap {a1 a2 b1 b2} {

    if {[expr {min($a2,$b2)}] > [expr {max($a1,$b1)}]} {
        return 1
    }

    return 0
}


########################################################################
# 判断两个 macro 是否属于同一组
#
# 条件：
#   左右相邻 gap <= SKIP_GAP
#   或
#   上下相邻 gap <= SKIP_GAP
#
# 同时要求对应方向有投影重叠
########################################################################
proc _mb_is_neighbor {A B SKIP_GAP} {

    lassign $A aname ax1 ay1 ax2 ay2
    lassign $B bname bx1 by1 bx2 by2


    ####################################################################
    # B 在 A 右边
    ####################################################################
    set gap [expr {$bx1 - $ax2}]

    if {$gap >= 0 &&
        $gap <= $SKIP_GAP &&
        [_mb_range_overlap $ay1 $ay2 $by1 $by2]} {

        return 1
    }


    ####################################################################
    # B 在 A 左边
    ####################################################################
    set gap [expr {$ax1 - $bx2}]

    if {$gap >= 0 &&
        $gap <= $SKIP_GAP &&
        [_mb_range_overlap $ay1 $ay2 $by1 $by2]} {

        return 1
    }


    ####################################################################
    # B 在 A 上边
    ####################################################################
    set gap [expr {$by1 - $ay2}]

    if {$gap >= 0 &&
        $gap <= $SKIP_GAP &&
        [_mb_range_overlap $ax1 $ax2 $bx1 $bx2]} {

        return 1
    }


    ####################################################################
    # B 在 A 下边
    ####################################################################
    set gap [expr {$ay1 - $by2}]

    if {$gap >= 0 &&
        $gap <= $SKIP_GAP &&
        [_mb_range_overlap $ax1 $ax2 $bx1 $bx2]} {

        return 1
    }


    ####################################################################
    # bbox 本身重叠
    ####################################################################
    if {[_mb_range_overlap $ax1 $ax2 $bx1 $bx2] &&
        [_mb_range_overlap $ay1 $ay2 $by1 $by2]} {

        return 1
    }


    return 0
}


########################################################################
# 创建一个 blockage
########################################################################
proc _mb_create_one \
    {name x1 y1 x2 y2 BLK_TYPE BLOCKED_PERCENT} {


    ####################################################################
    # 非法区域直接跳过
    ####################################################################
    if {$x2 <= $x1 || $y2 <= $y1} {
        return
    }


    set boundary [list \
        [list $x1 $y1] \
        [list $x2 $y2] \
    ]


    ####################################################################
    # HARD
    ####################################################################
    if {$BLK_TYPE eq "hard"} {

        create_placement_blockage \
            -name $name \
            -type hard \
            -boundary $boundary

        return
    }


    ####################################################################
    # SOFT
    ####################################################################
    if {$BLK_TYPE eq "soft"} {

        create_placement_blockage \
            -name $name \
            -type soft \
            -boundary $boundary

        return
    }


    ####################################################################
    # PARTIAL
    ####################################################################
    if {$BLK_TYPE eq "partial"} {

        create_placement_blockage \
            -name $name \
            -type partial \
            -blocked_percentage $BLOCKED_PERCENT \
            -boundary $boundary

        return
    }
}


########################################################################
# 将区域切成最大 10 x 10
#
# 例如：
#
#   4 x 37
#
# ->
#
#   4 x 10
#   4 x 10
#   4 x 10
#   4 x 7
########################################################################
proc _mb_create_tiles \
    {prefix x1 y1 x2 y2 BLK_TYPE BLOCKED_PERCENT} {


    set TILE 10.0


    if {$x2 <= $x1 || $y2 <= $y1} {
        return
    }


    set id 0
    set cx $x1


    while {$cx < $x2} {

        set nx [expr {$cx + $TILE}]

        if {$nx > $x2} {
            set nx $x2
        }


        set cy $y1


        while {$cy < $y2} {

            set ny [expr {$cy + $TILE}]

            if {$ny > $y2} {
                set ny $y2
            }


            incr id


            _mb_create_one \
                ${prefix}_${id} \
                $cx \
                $cy \
                $nx \
                $ny \
                $BLK_TYPE \
                $BLOCKED_PERCENT


            set cy $ny
        }


        set cx $nx
    }
}


########################################################################
# 根据 blockage 类型决定是否切块
#
# hard:
#   整块创建
#
# soft / partial:
#   10x10切块
########################################################################
proc _mb_create_area \
    {prefix x1 y1 x2 y2 BLK_TYPE BLOCKED_PERCENT} {


    if {$x2 <= $x1 || $y2 <= $y1} {
        return
    }


    ####################################################################
    # HARD：不切块
    ####################################################################
    if {$BLK_TYPE eq "hard"} {

        _mb_create_one \
            $prefix \
            $x1 \
            $y1 \
            $x2 \
            $y2 \
            $BLK_TYPE \
            $BLOCKED_PERCENT

        return
    }


    ####################################################################
    # SOFT / PARTIAL：切成最大10x10
    ####################################################################
    _mb_create_tiles \
        $prefix \
        $x1 \
        $y1 \
        $x2 \
        $y2 \
        $BLK_TYPE \
        $BLOCKED_PERCENT
}


########################################################################
# 主函数
########################################################################
proc pr_halo \
    {HALO_L HALO_B HALO_R HALO_T args} {


    ####################################################################
    # 固定参数
    ####################################################################
    set SKIP_GAP 1.1


    ####################################################################
    # 检查 HALO
    ####################################################################
    foreach v [list \
        $HALO_L \
        $HALO_B \
        $HALO_R \
        $HALO_T \
    ] {

        if {![string is double -strict $v]} {

            error \
"HALO must be numeric.

Usage:
  pr_halo L B R T -type hard
  pr_halo L B R T -type soft
  pr_halo L B R T -type part PERCENT"
        }


        if {$v < 0} {
            error "HALO cannot be negative."
        }
    }


    ####################################################################
    # 解析 -type
    ####################################################################
    set BLK_TYPE ""
    set BLOCKED_PERCENT ""

    set argc [llength $args]
    set i 0


    while {$i < $argc} {

        set opt [lindex $args $i]


        switch -- $opt {

            -type {

                incr i


                if {$i >= $argc} {
                    error "Missing value after -type."
                }


                set type_value \
                    [string tolower [lindex $args $i]]


                ########################################################
                # HARD
                ########################################################
                if {$type_value eq "hard"} {

                    set BLK_TYPE hard


                ########################################################
                # SOFT
                ########################################################
                } elseif {$type_value eq "soft"} {

                    set BLK_TYPE soft


                ########################################################
                # PART / PARTIAL
                ########################################################
                } elseif {$type_value eq "part" ||
                          $type_value eq "partial"} {

                    set BLK_TYPE partial


                    incr i


                    if {$i >= $argc} {

                        error \
"-type part requires percentage.

Example:
  pr_halo 3 3 3 3 -type part 70"
                    }


                    set BLOCKED_PERCENT \
                        [lindex $args $i]


                    if {![string is double -strict $BLOCKED_PERCENT]} {

                        error \
"Partial blockage percentage must be numeric."
                    }


                    if {$BLOCKED_PERCENT <= 0 ||
                        $BLOCKED_PERCENT > 100} {

                        error \
"Partial percentage must be > 0 and <= 100."
                    }


                ########################################################
                # UNKNOWN
                ########################################################
                } else {

                    error \
"Unknown blockage type: $type_value

Use:
  -type hard
  -type soft
  -type part PERCENT"
                }
            }


            default {

                error \
"Unknown option: $opt

Usage:
  pr_halo L B R T -type hard
  pr_halo L B R T -type soft
  pr_halo L B R T -type part PERCENT"
            }
        }


        incr i
    }


    ####################################################################
    # 必须指定 -type
    ####################################################################
    if {$BLK_TYPE eq ""} {

        error \
"Missing -type.

Usage:
  pr_halo L B R T -type hard
  pr_halo L B R T -type soft
  pr_halo L B R T -type part PERCENT"
    }


    ####################################################################
    # 打印参数
    ####################################################################
    puts ""
    puts "======================================================"
    puts " Macro Blockage"
    puts "======================================================"
    puts " LEFT      = $HALO_L"
    puts " BOTTOM    = $HALO_B"
    puts " RIGHT     = $HALO_R"
    puts " TOP       = $HALO_T"
    puts " SKIP_GAP  = $SKIP_GAP"
    puts " TYPE      = $BLK_TYPE"


    if {$BLK_TYPE eq "hard"} {

        puts " TILE      = OFF"

    } else {

        puts " TILE      = 10 x 10"
    }


    if {$BLK_TYPE eq "partial"} {
        puts " BLOCKED % = $BLOCKED_PERCENT"
    }


    puts "======================================================"


    ####################################################################
    # 获取当前 GUI selection
    ####################################################################
    set sel [get_selection]


    if {[sizeof_collection $sel] == 0} {

        error \
"No objects selected.
Please select hard macros first."
    }


    ####################################################################
    # 只保留 hard macro
    ####################################################################
    set macros \
        [filter_collection \
            $sel \
            "object_class == cell && is_hard_macro == true"]


    if {[sizeof_collection $macros] == 0} {

        error \
"No hard macros found in current selection."
    }


    ####################################################################
    # Collection -> Tcl list
    #
    # 每个元素：
    #
    #   {name x1 y1 x2 y2}
    ####################################################################
    set macro_list {}


    foreach_in_collection m $macros {

        set name [get_object_name $m]

        set bbox [get_attribute $m bbox]


        set x1 [lindex $bbox 0 0]
        set y1 [lindex $bbox 0 1]

        set x2 [lindex $bbox 1 0]
        set y2 [lindex $bbox 1 1]


        lappend macro_list \
            [list \
                $name \
                $x1 \
                $y1 \
                $x2 \
                $y2 \
            ]


        puts ""
        puts "MACRO : $name"
        puts "BBOX  : {$x1 $y1} {$x2 $y2}"
    }


    set NUM [llength $macro_list]


    puts ""
    puts "Selected macros = $NUM"


    ####################################################################
    # 建立 connected group
    #
    # 例如：
    #
    # A --1-- B --1-- C
    #
    # 三个会自动变成同一个 group
    ####################################################################
    array set visited {}

    set groups {}


    for {set i 0} {$i < $NUM} {incr i} {


        if {[info exists visited($i)]} {
            continue
        }


        set queue [list $i]

        set visited($i) 1

        set this_group {}


        while {[llength $queue] > 0} {


            set current [lindex $queue 0]

            set queue [lrange $queue 1 end]


            lappend this_group $current


            set A [lindex $macro_list $current]


            for {set j 0} {$j < $NUM} {incr j} {


                if {[info exists visited($j)]} {
                    continue
                }


                set B [lindex $macro_list $j]


                if {[_mb_is_neighbor \
                    $A \
                    $B \
                    $SKIP_GAP]} {


                    set visited($j) 1

                    lappend queue $j
                }
            }
        }


        lappend groups $this_group
    }


    ####################################################################
    # 创建外围 blockage
    ####################################################################
    set group_id 0

    set RUN_ID [clock clicks]


    foreach group $groups {


        incr group_id


        ################################################################
        # 初始化 group bbox
        ################################################################
        set first_index \
            [lindex $group 0]


        set first_macro \
            [lindex $macro_list $first_index]


        lassign $first_macro \
            first_name \
            gx1 \
            gy1 \
            gx2 \
            gy2


        puts ""
        puts "------------------------------------------------------"
        puts "GROUP $group_id"
        puts "------------------------------------------------------"


        ################################################################
        # 计算整个 group 的 bbox
        ################################################################
        foreach index $group {


            set M [lindex $macro_list $index]


            lassign $M \
                name \
                x1 \
                y1 \
                x2 \
                y2


            puts "  $name"


            if {$x1 < $gx1} {
                set gx1 $x1
            }


            if {$y1 < $gy1} {
                set gy1 $y1
            }


            if {$x2 > $gx2} {
                set gx2 $x2
            }


            if {$y2 > $gy2} {
                set gy2 $y2
            }
        }


        puts ""
        puts "GROUP BBOX:"
        puts "  {$gx1 $gy1} {$gx2 $gy2}"


        set PREFIX \
            "MACRO_HALO_${RUN_ID}_G${group_id}"


        ################################################################
        # LEFT
        ################################################################
        if {$HALO_L > 0} {

            _mb_create_area \
                ${PREFIX}_L \
                [expr {$gx1 - $HALO_L}] \
                [expr {$gy1 - $HALO_B}] \
                $gx1 \
                [expr {$gy2 + $HALO_T}] \
                $BLK_TYPE \
                $BLOCKED_PERCENT
        }


        ################################################################
        # RIGHT
        ################################################################
        if {$HALO_R > 0} {

            _mb_create_area \
                ${PREFIX}_R \
                $gx2 \
                [expr {$gy1 - $HALO_B}] \
                [expr {$gx2 + $HALO_R}] \
                [expr {$gy2 + $HALO_T}] \
                $BLK_TYPE \
                $BLOCKED_PERCENT
        }


        ################################################################
        # BOTTOM
        ################################################################
        if {$HALO_B > 0} {

            _mb_create_area \
                ${PREFIX}_B \
                $gx1 \
                [expr {$gy1 - $HALO_B}] \
                $gx2 \
                $gy1 \
                $BLK_TYPE \
                $BLOCKED_PERCENT
        }


        ################################################################
        # TOP
        ################################################################
        if {$HALO_T > 0} {

            _mb_create_area \
                ${PREFIX}_T \
                $gx1 \
                $gy2 \
                $gx2 \
                [expr {$gy2 + $HALO_T}] \
                $BLK_TYPE \
                $BLOCKED_PERCENT
        }
    }


    puts ""
    puts "======================================================"
    puts " DONE"
    puts " Macro groups = [llength $groups]"
    puts "======================================================"
    puts ""
}

```
