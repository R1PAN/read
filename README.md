```
LE_prompt=$1
  LE_max=${2--1}
  LE_fill=${3-" "}

  LE_backward() {
    LE_s=$1
    while [ "x$LE_s" != x ]; do
      printf '\b%s' "$2"
      LE_s=${LE_s%?}
    done
  }

  LE_fill() {
    LE_s=$1
    while [ "x$LE_s" != x ]; do
      printf %s "$LE_fill"
      LE_s=${LE_s%?}
    done
  }

  LE_restore='stty "$LE_tty"
              LC_COLLATE='${LC_COLLATE-"; unset LC_COLLATE"}
  LE_ret=1 LE_tty=$(stty -g) LE_px=$4 LE_sx= LC_COLLATE=C

  stty -icanon -echo -isig min 100 time 1 -istrip
  printf '%b%s' "$LE_prompt" "$LE_px"

  while set -- $(dd bs=100 count=1 2> /dev/null | od -vAn -to1); do
    while [ "$#" -gt 0 ]; do
      LE_k=$1
      shift
      if [ "$LE_k" = 033 ]; then
        case "$1$2$3" in
          133103*|117103*) shift 2; LE_k=006;;
          133104*|117104*) shift 2; LE_k=002;;
          133110*|117110*) shift 2; LE_k=001;;
          133120*|117120*) shift 2; LE_k=004;;
          133106*|117106*) shift 2; LE_k=005;;
          133061176) shift 3; LE_k=001;;
          133064176) shift 3; LE_k=005;;
          133063176) shift 3; LE_k=004;;
          133*|117*)
            shift
            while [ "0$1" -ge 060 ] && [ "0$1" -le 071 ] ||
                  [ "0$1" -eq 073 ]; do
              shift
            done;;
        esac
      fi

      case $LE_k in
001) # ^A beginning of line
          LE_backward "$LE_px"
          LE_sx=$LE_px$LE_sx
          LE_px=;;
        002) # ^B backward
          if [ "x$LE_px" = x ]; then
            printf '\a'
          else
            printf '\b'
            LE_tmp=${LE_px%?}
            LE_sx=${LE_px#"$LE_tmp"}$LE_sx
            LE_px=$LE_tmp
          fi;;
        003) # CTRL-C
          break 2;;
        004) # ^D del char
          if [ "x$LE_sx" = x ]; then
            printf '\a'
          else
            LE_sx=${LE_sx#?}
            printf '%s\b' "$LE_sx$LE_fill"
            LE_backward "$LE_sx"
          fi;;
        012|015) # NL or CR
          LE_sx=$LE_sx$'\n'  # Tambahkan karakter newline (enter) ke LE_sx
          printf %s "$LE_sx"
          LE_px=$LE_px$LE_sx
          LE_sx=;;
        006) # ^F forward
          if [ "x$LE_sx" = x ]; then
            printf '\a'
          else
            LE_tmp=${LE_sx#?}
            LE_px=$LE_px${LE_sx%"$LE_tmp"}
            printf %s "${LE_sx%"$LE_tmp"}"
            LE_sx=$LE_tmp
          fi;;
        010|177) # backspace or del
          if [ "x$LE_px" = x ]; then
            printf '\a'
          else
            printf '\b%s\b' "$LE_sx$LE_fill"
            LE_backward "$LE_sx"
            LE_px=${LE_px%?}
          fi;;
        013) # ^K kill to end of line
          LE_fill "$LE_sx"
          LE_backward "$LE_sx"
          LE_sx=;;
        014) # ^L redraw
          printf '\r%b%s' "$LE_prompt" "$LE_px$LE_sx"
          LE_backward "$LE_sx";;
        025) # ^U kill line
          LE_backward "$LE_px"
          LE_fill "$LE_px$LE_sx"
          LE_backward "$LE_px$LE_sx"
          LE_px=
          LE_sx=;;
        027) # ^W kill word
          if [ "x$LE_px" = x ]; then
            printf '\a'
          else
            LE_tmp=${LE_px% *}
            LE_backward "${LE_px#"$LE_tmp"}"
            LE_fill "${LE_px#"$LE_tmp"}"
            LE_backward "${LE_px#"$LE_tmp"}"
            LE_px=$LE_tmp
          fi;;
        033) # Tombol ESC
          break 2;;
        [02][4-7]?|[13]??) # 040 -> 177, 240 -> 377
                           # was assuming iso8859-x at the time
          if [ "$LE_max" -ge 0 ] && LE_tmp=$LE_px$LE_sx \
             && [ "${#LE_tmp}" -eq "$LE_max" ]; then
            printf '\a'
          else
            LE_px=$LE_px$(printf '%b' "\0$LE_k")
            printf '%b%s' "\0$LE_k" "$LE_sx"
            LE_backward "$LE_sx"
          fi;;
        *)
          printf '\a';;
      esac
    done
  done
  eval "$LE_restore"
  REPLY=$LE_px$LE_sx
  echo $REPLY


