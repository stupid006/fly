```tcl
source /nobackup/users/fulaixiao_sd/fc_shell/owner_script/shellfc/fchrc.tcl
source /nobackup/users/fulaixiao_sd/fc_shell/owner_script/pr/pr_halo.tcl
source /nobackup/users/fulaixiao_sd/fc_shell/owner_script/pr/pr_blk.tcl
source /nobackup/users/fulaixiao_sd/fc_shell/owner_script/pr/pr_split_blk.tcl
source /nobackup/users/fulaixiao_sd/fc_shell/owner_script/pr/pr_space.tcl

# ============================================================
# Macro PR Utility Functions
#
# Source once, call anytime:
#
#   source func.tcl
#
# Functions:
#
#   pr_blk        <gap> soft|hard|part <blocked_percentage>
#   pr_split_blk   <seq>
#   pr_space_1    <gap>
#   pr_space_2    <pin2pin_gap> <back2back_gap>
#   pr_halo       <L> <B> <R> <T> -type hard|soft|part <pct>
#   pr_help
#
# Function implementations live in:
#
#   pr_halo.tcl       -> pr_halo
#   pr_blk.tcl        -> pr_blk
#   pr_split_blk.tcl  -> pr_split_blk
#   pr_space.tcl      -> pr_space_1, pr_space_2
#
# All functions operate on FC GUI current selection.
# ============================================================


# ============================================================
# Help
# ============================================================

proc pr_help {} {

    puts ""
    puts "============================================================"
    puts " Macro PR Utility"
    puts "============================================================"
    puts ""
    puts " 1. pr_blk <gap> soft|hard|part <blocked_percentage>"
    puts ""
    puts "    Create placement blockages between selected hard macros"
    puts "    whose horizontal edge-to-edge gap is greater than <gap>"
    puts "    (gap > <gap>, macros must overlap in Y)."
    puts ""
    puts "    Only adjacent pairs are blocked (no other selected"
    puts "    macro inside the gap region), so blockages never"
    puts "    overlap or cover an intermediate macro."
    puts ""
    puts "    Existing blockages overlapping the target region are"
    puts "    removed first (replace behavior)."
    puts ""
    puts "    Examples:"
    puts "      pr_blk 3 soft"
    puts "      pr_blk 3 hard"
    puts "      pr_blk 3 part 70"
    puts "      pr_blk 35 part 50"
    puts ""
    puts "------------------------------------------------------------"
    puts ""
    puts " 2. pr_split_blk <seq>"
    puts ""
    puts "    Split selected placement blockage(s) into 10x10 small"
    puts "    blockages. Keeps original type (hard/soft/partial) and"
    puts "    blocked_percentage. Removes original blockage after"
    puts "    splitting (only when created count == expected count)."
    puts ""
    puts "    <seq> is a run number (1/2/3 ...) embedded in the new"
    puts "    blockage names, so each run uses unique names."
    puts ""
    puts "    Examples:"
    puts "      pr_split_blk 1"
    puts "      pr_split_blk 2"
    puts "      pr_split_blk 3"
    puts ""
    puts "------------------------------------------------------------"
    puts ""
    puts " 3. pr_space_1 <gap>"
    puts ""
    puts "    Rightmost macro is anchor (keeps position and orientation)."
    puts "    All other macros follow anchor orientation and are placed"
    puts "    from right to left with uniform gap."
    puts ""
    puts "    Example:"
    puts "      pr_space_1 3"
    puts ""
    puts "------------------------------------------------------------"
    puts ""
    puts " 4. pr_space_2 <pin2pin_gap> <back2back_gap>"
    puts ""
    puts "    Rightmost macro is anchor. Orientation alternates"
    puts "    R180 / MX / R180 / MX ... from right to left."
    puts "    Gap alternates pin-to-pin / back-to-back."
    puts ""
    puts "    Example:"
    puts "      pr_space_2 16 1"
    puts ""
    puts "------------------------------------------------------------"
    puts ""
    puts " 5. pr_halo <L> <B> <R> <T> -type hard|soft|part <pct>"
    puts ""
    puts "    Create halo placement blockages around selected hard"
    puts "    macros. Macros whose gap <= 1.1 um are treated as one"
    puts "    group (no blockage between them)."
    puts ""
    puts "    soft/partial blockages are auto-split into 10x10 tiles."
    puts ""
    puts "    Examples:"
    puts "      pr_halo 4 0 4 0 -type hard"
    puts "      pr_halo 3 3 3 3 -type soft"
    puts "      pr_halo 3 3 3 3 -type part 70"
    puts ""
    puts "============================================================"
    puts ""
}


# ============================================================
# Automatically print help after source
# ============================================================

pr_help


```

