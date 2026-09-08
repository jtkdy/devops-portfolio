---
date: 2026-06-02
tags:
  - security
  - kisa
  - al2023
  - script
---

```bash
#!/usr/bin/env bash
# =============================================================================
# Amazon Linux 2023 KISA 보안조치 자동화 스크립트
# 기준: KISA 주요정보통신기반시설 기술적 취약점 분석·평가 방법 상세가이드 2026
# 항목: Unix 서버 U-01 ~ U-67 전체 (67개)
#
# 사용법:
#   sudo ./al2023_kisa_hardening.sh [옵션]
#
# 옵션:
#   --check-only        점검만 수행 (조치/검증 없음)
#   --dry-run           조치 내용 출력만 (실제 변경 없음)
#   --fix-only          check 생략, 전 항목 조치 실행
#   --no-verify         조치 후 검증 단계 생략
#   --item U-01,U-03    특정 항목만 실행 (쉼표 구분)
#   --category 1        특정 대분류만 실행 (1=계정관리, 2=파일디렉토리, 3=서비스, 4=패치, 5=로그)
#
# 항목 처리 유형:
#   [AUTO]   자동 점검 + 자동 조치 가능
#   [MANUAL] 자동 점검 + 수동 조치 필요 (서비스 중단/데이터 손실 위험)
#   [INFO]   점검만 수행 (환경 의존적, 조치 기준 없음)
#   [N/A]    AL2023 환경에서 해당 서비스 미존재
#
# 실행 흐름:
#   [기본]          check → (FAIL이면) fix → verify
#   [--check-only]  check only
#   [--dry-run]     check → dryrun 출력
#   [--fix-only]    fix → verify
# =============================================================================

set -uo pipefail

# =============================================================================
# 전역 변수
# =============================================================================
SCRIPT_VERSION="3.0.0"
SCRIPT_NAME=$(basename "$0")
HOSTNAME_SHORT=$(hostname -s 2>/dev/null || echo "unknown")
LOG_DIR="/var/log/security-hardening"
LOG_FILE="${LOG_DIR}/kisa_${HOSTNAME_SHORT}_$(date +%Y%m%d_%H%M%S).log"
BACKUP_DIR="${LOG_DIR}/backup_$(date +%Y%m%d_%H%M%S)"

DRY_RUN=false
CHECK_ONLY=false
FIX_ONLY=false
NO_VERIFY=false
SELECTED_ITEMS=""
SELECTED_CATEGORY=""

# 카운터
CNT_PASS=0
CNT_FIXED=0
CNT_FAILED=0
CNT_MANUAL=0
CNT_NA=0
CNT_SKIP=0

# 컬러
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
MAGENTA='\033[0;35m'
BOLD='\033[1m'
DIM='\033[2m'
NC='\033[0m'

# =============================================================================
# 공통 출력/로그 함수
# =============================================================================
log()          { echo "$(date '+%Y-%m-%d %H:%M:%S') [${HOSTNAME_SHORT}] [${1}] ${*:2}" >> "${LOG_FILE}"; }
_pass()        { echo -e "  ${GREEN}✔ PASS    ${NC} $1"; log "PASS"        "$1"; }
_fail()        { echo -e "  ${RED}✘ FAIL    ${NC} $1"; log "FAIL"        "$1"; }
_fixed()       { echo -e "  ${GREEN}✔ FIXED   ${NC} $1"; log "FIXED"       "$1"; }
_verify_ok()   { echo -e "  ${GREEN}✔ VERIFIED${NC} $1"; log "VERIFIED"    "$1"; }
_verify_fail() { echo -e "  ${RED}✘ VRF_FAIL${NC} $1"; log "VERIFY_FAIL" "$1"; }
_skip()        { echo -e "  ${YELLOW}⊘ SKIP    ${NC} $1"; log "SKIP"        "$1"; }
_warn()        { echo -e "  ${YELLOW}⚠ WARN    ${NC} $1"; log "WARN"        "$1"; }
_dryrun()      { echo -e "  ${MAGENTA}~ DRYRUN  ${NC} would: $1"; log "DRYRUN"      "$1"; }
_manual()      { echo -e "  ${YELLOW}⚙ MANUAL  ${NC} $1"; log "MANUAL"      "$1"; }
_na()          { echo -e "  ${DIM}– N/A     ${NC} $1"; log "NA"           "$1"; }
_info()        { echo -e "  ${CYAN}ℹ INFO    ${NC} $1"; log "INFO"        "$1"; }

print_header() {
    echo -e "\n${BOLD}${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo -e "${BOLD}${BLUE}  $1${NC}"
    echo -e "${BOLD}${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
}

# =============================================================================
# 항목 헤더 출력
#   $1=항목ID  $2=항목명  $3=중요도  $4=유형(AUTO/MANUAL/INFO/N/A)
# =============================================================================
print_item_header() {
    local id="$1" name="$2" grade="$3" type="$4"
    local grade_color="${NC}"
    local type_label=""
    case "$grade" in
        상) grade_color="${RED}" ;;
        중) grade_color="${YELLOW}" ;;
        하) grade_color="${CYAN}" ;;
    esac
    case "$type" in
        AUTO)   type_label="${GREEN}[AUTO]  ${NC}" ;;
        MANUAL) type_label="${YELLOW}[MANUAL]${NC}" ;;
        INFO)   type_label="${CYAN}[INFO]  ${NC}" ;;
        "N/A")  type_label="${DIM}[N/A]   ${NC}" ;;
    esac
    echo -e "\n${BOLD}┌─[${CYAN}${id}${NC}${BOLD}] ${name}${NC}  ${grade_color}(${grade})${NC}  ${type_label}"
    log "INFO" "=== ${id}: ${name} (${grade}) [${type}] ==="
}

# =============================================================================
# 파일 백업
# =============================================================================
backup_file() {
    local file="$1"
    [[ ! -f "$file" ]] && return
    local dest="${BACKUP_DIR}$(dirname "$file")"
    mkdir -p "$dest"
    cp -p "$file" "${dest}/$(basename "$file").bak"
    log "INFO" "Backup: $file"
}

# =============================================================================
# 조치 실행 래퍼
# =============================================================================
do_fix() {
    local desc="$1" cmd="$2"
    if $DRY_RUN; then
        _dryrun "$desc"
        return 0
    fi
    if eval "$cmd"; then
        _fixed "$desc"
        return 0
    else
        _fail "조치 실패: $desc"
        return 1
    fi
}

# =============================================================================
# 항목 실행 여부 판단
# =============================================================================
should_run() {
    local id="$1" cat="$2"
    # 카테고리 필터
    if [[ -n "$SELECTED_CATEGORY" && "$cat" != "$SELECTED_CATEGORY" ]]; then
        return 1
    fi
    # 항목 필터
    if [[ -n "$SELECTED_ITEMS" ]]; then
        echo "$SELECTED_ITEMS" | tr ',' '\n' | grep -qix "$id" || return 1
    fi
    return 0
}

# =============================================================================
# AUTO 항목 실행 프레임워크 (check → fix → verify)
# =============================================================================
run_auto() {
    local item_id="$1" item_name="$2" grade="$3" check_fn="$4" fix_fn="$5"
    local verify_fn="${6:-$check_fn}"

    print_item_header "$item_id" "$item_name" "$grade" "AUTO"

    # [1] CHECK
    local check_result=0
    if ! $FIX_ONLY; then
        echo -e "  ${BOLD}[1/3] CHECK${NC}"
        $check_fn && check_result=0 || check_result=1
    else
        check_result=1
        echo -e "  ${YELLOW}[1/3] CHECK SKIPPED (fix-only)${NC}"
    fi

    if [[ $check_result -eq 0 ]]; then
        echo -e "  ${BOLD}[2/3] FIX   ${NC}: ${GREEN}불필요 (이미 양호)${NC}"
        echo -e "  ${BOLD}[3/3] VERIFY${NC}: ${GREEN}불필요${NC}"
        ((CNT_PASS++))
        log "RESULT" "$item_id: PASS"
        return 0
    fi

    # [2] FIX
    echo -e "  ${BOLD}[2/3] FIX${NC}"
    if $CHECK_ONLY; then
        echo -e "       ${YELLOW}→ CHECK-ONLY 모드: 조치 생략${NC}"
        ((CNT_FAILED++))
        log "RESULT" "$item_id: FAIL (check-only)"
        return 1
    fi

    local fix_result=0
    $fix_fn || fix_result=1
    if [[ $fix_result -ne 0 ]] && ! $DRY_RUN; then
        echo -e "       ${RED}→ 조치 실패 (수동 확인 필요)${NC}"
        ((CNT_FAILED++))
        log "RESULT" "$item_id: FAILED"
        return 1
    fi

    # [3] VERIFY
    echo -e "  ${BOLD}[3/3] VERIFY${NC}"
    if $DRY_RUN || $NO_VERIFY; then
        echo -e "       ${YELLOW}→ 검증 생략${NC}"
        ((CNT_FIXED++))
        log "RESULT" "$item_id: FIXED (verify skipped)"
        return 0
    fi

    local verify_result=0
    $verify_fn && verify_result=0 || verify_result=1
    if [[ $verify_result -eq 0 ]]; then
        _verify_ok "조치 완료 및 검증됨"
        ((CNT_FIXED++))
        log "RESULT" "$item_id: FIXED+VERIFIED"
    else
        _verify_fail "조치 후에도 미흡 - 수동 확인 필요"
        ((CNT_FAILED++))
        log "RESULT" "$item_id: VERIFY_FAILED"
    fi
}

# =============================================================================
# MANUAL 항목 프레임워크 (check만, fix는 안내 출력)
# =============================================================================
run_manual() {
    local item_id="$1" item_name="$2" grade="$3" check_fn="$4" manual_msg="$5"

    print_item_header "$item_id" "$item_name" "$grade" "MANUAL"
    echo -e "  ${BOLD}[1/2] CHECK${NC}"
    local check_result=0
    $check_fn && check_result=0 || check_result=1

    echo -e "  ${BOLD}[2/2] FIX${NC}"
    if [[ $check_result -eq 0 ]]; then
        echo -e "       ${GREEN}→ 양호 (조치 불필요)${NC}"
        ((CNT_PASS++))
        log "RESULT" "$item_id: PASS"
    else
        _manual "수동 조치 필요: ${manual_msg}"
        ((CNT_MANUAL++))
        log "RESULT" "$item_id: MANUAL_REQUIRED"
    fi
}

# =============================================================================
# INFO 항목 프레임워크 (점검 결과만 출력, PASS/FAIL 없음)
# =============================================================================
run_info() {
    local item_id="$1" item_name="$2" grade="$3" check_fn="$4"
    print_item_header "$item_id" "$item_name" "$grade" "INFO"
    $check_fn
    log "RESULT" "$item_id: INFO"
}

# =============================================================================
# N/A 항목 프레임워크
# =============================================================================
run_na() {
    local item_id="$1" item_name="$2" grade="$3" reason="$4"
    print_item_header "$item_id" "$item_name" "$grade" "N/A"
    _na "$reason"
    ((CNT_NA++))
    log "RESULT" "$item_id: N/A - $reason"
}

# =============================================================================
# ============================================================================
# 1. 계정 관리 (U-01 ~ U-13)
# =============================================================================
# =============================================================================

# ── U-01: root 계정 원격 접속 제한 [AUTO] ────────────────────────────────────
_u01_check() {
    local f="/etc/ssh/sshd_config"
    local val
    val=$(grep -iE "^PermitRootLogin" "$f" 2>/dev/null | awk '{print $2}' | head -1)
    [[ "${val,,}" == "no" ]] \
        && { _pass "[${f}] PermitRootLogin=no"; return 0; } \
        || { _fail "[${f}] PermitRootLogin='${val:-미설정}' (no 필요)"; return 1; }
}
_u01_fix() {
    local f="/etc/ssh/sshd_config"; backup_file "$f"
    _warn "사전 확인: root SSH 접속에 의존하는 배포 도구/자동화 스크립트가 없는지 확인 후 진행"
    grep -qiE "^PermitRootLogin" "$f" \
        && do_fix "[${f}] PermitRootLogin no 수정" "sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' '$f'" \
        || do_fix "[${f}] PermitRootLogin no 추가" "echo 'PermitRootLogin no' >> '$f'"
    $DRY_RUN || do_fix "sshd reload" "systemctl reload sshd 2>/dev/null || systemctl reload sshd"
}

# ── U-02: 비밀번호 관리정책 설정 [AUTO] ──────────────────────────────────────
# 복잡성(pwquality) + 최대/최소 사용기간(login.defs) 통합
_u02_check() {
    local ok=true
    local lf="/etc/login.defs" pq="/etc/security/pwquality.conf"

    local max min
    max=$(awk '/^PASS_MAX_DAYS/{print $2}' "$lf" 2>/dev/null)
    min=$(awk '/^PASS_MIN_DAYS/{print $2}' "$lf" 2>/dev/null)
    [[ "${max:-999}" -le 90 ]] \
        && _pass "[${lf}] PASS_MAX_DAYS=${max}" \
        || { _fail "[${lf}] PASS_MAX_DAYS=${max:-없음} (≤90 필요)"; ok=false; }
    [[ "${min:-0}" -ge 1 ]] \
        && _pass "[${lf}] PASS_MIN_DAYS=${min}" \
        || { _fail "[${lf}] PASS_MIN_DAYS=${min:-없음} (≥1 필요)"; ok=false; }

    if [[ -f "$pq" ]]; then
        local minlen dcredit ucredit lcredit ocredit
        minlen=$(awk  -F= '/^\s*minlen/{gsub(/ /,"");print $2}' "$pq")
        dcredit=$(awk -F= '/^\s*dcredit/{gsub(/ /,"");print $2}' "$pq")
        ucredit=$(awk -F= '/^\s*ucredit/{gsub(/ /,"");print $2}' "$pq")
        lcredit=$(awk -F= '/^\s*lcredit/{gsub(/ /,"");print $2}' "$pq")
        ocredit=$(awk -F= '/^\s*ocredit/{gsub(/ /,"");print $2}' "$pq")
        [[ "${minlen:-0}"  -ge 8  ]] && _pass "[${pq}] minlen=${minlen}"   || { _fail "[${pq}] minlen=${minlen:-없음} (≥8)";   ok=false; }
        [[ "${dcredit:-0}" -le -1 ]] && _pass "[${pq}] dcredit=${dcredit}" || { _fail "[${pq}] dcredit=${dcredit:-없음} (≤-1)"; ok=false; }
        [[ "${ucredit:-0}" -le -1 ]] && _pass "[${pq}] ucredit=${ucredit}" || { _fail "[${pq}] ucredit=${ucredit:-없음} (≤-1)"; ok=false; }
        [[ "${lcredit:-0}" -le -1 ]] && _pass "[${pq}] lcredit=${lcredit}" || { _fail "[${pq}] lcredit=${lcredit:-없음} (≤-1)"; ok=false; }
        [[ "${ocredit:-0}" -le -1 ]] && _pass "[${pq}] ocredit=${ocredit}" || { _fail "[${pq}] ocredit=${ocredit:-없음} (≤-1)"; ok=false; }
    else
        _fail "[${pq}] 파일 없음"; ok=false
    fi
    $ok && return 0 || return 1
}
_u02_fix() {
    local lf="/etc/login.defs" pq="/etc/security/pwquality.conf"
    backup_file "$lf"
    grep -q "^PASS_MAX_DAYS" "$lf" \
        && do_fix "[${lf}] PASS_MAX_DAYS=90 수정" "sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' '$lf'" \
        || do_fix "[${lf}] PASS_MAX_DAYS=90 추가" "echo 'PASS_MAX_DAYS   90' >> '$lf'"
    grep -q "^PASS_MIN_DAYS" "$lf" \
        && do_fix "[${lf}] PASS_MIN_DAYS=1 수정"  "sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/'  '$lf'" \
        || do_fix "[${lf}] PASS_MIN_DAYS=1 추가"  "echo 'PASS_MIN_DAYS   1' >> '$lf'"
    backup_file "$pq"
    do_fix "[${pq}] pwquality 정책 적용" "cat > '$pq' << 'EOF'
# KISA U-02
minlen = 8
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
EOF"
}

# ── U-03: 계정 잠금 임계값 설정 [AUTO] ───────────────────────────────────────
_u03_check() {
    local ok=true f="/etc/security/faillock.conf"
    [[ ! -f "$f" ]] && { _fail "[${f}] 파일 없음"; return 1; }
    local deny unlock
    deny=$(awk    -F= '/^\s*deny/{gsub(/ /,"");print $2}' "$f")
    unlock=$(awk  -F= '/^\s*unlock_time/{gsub(/ /,"");print $2}' "$f")
    [[ "${deny:-99}" -le 5 && "${deny:-0}" -gt 0 ]] \
        && _pass "[${f}] deny=${deny}" \
        || { _fail "[${f}] deny=${deny:-없음} (1~5 필요)"; ok=false; }
    [[ "${unlock:-0}" -ge 1800 ]] \
        && _pass "[${f}] unlock_time=${unlock}s" \
        || { _fail "[${f}] unlock_time=${unlock:-없음} (≥1800s 필요)"; ok=false; }
    $ok && return 0 || return 1
}
_u03_fix() {
    local f="/etc/security/faillock.conf"; backup_file "$f"
    _warn "사전 확인: /etc/pam.d/system-auth 수동 수정 내용이 있다면 authselect 활성화 시 초기화될 수 있음"
    [[ ! -f "$f" ]] && do_fix "[${f}] 파일 생성" "touch '$f'"
    grep -q "^deny" "$f" \
        && do_fix "[${f}] deny=5 수정" "sed -i 's/^deny.*/deny = 5/' '$f'" \
        || do_fix "[${f}] deny=5 추가" "echo 'deny = 5' >> '$f'"
    grep -q "^unlock_time" "$f" \
        && do_fix "[${f}] unlock_time=1800 수정" "sed -i 's/^unlock_time.*/unlock_time = 1800/' '$f'" \
        || do_fix "[${f}] unlock_time=1800 추가" "echo 'unlock_time = 1800' >> '$f'"
    command -v authselect &>/dev/null && \
        authselect current 2>/dev/null | grep -q "with-faillock" || \
        do_fix "authselect with-faillock 활성화" "authselect enable-feature with-faillock 2>/dev/null || true"
}

# ── U-04: 비밀번호 파일 보호 [AUTO] ──────────────────────────────────────────
_u04_check() {
    local ok=true s="/etc/shadow" p="/etc/passwd"
    if [[ -f "$s" ]]; then
        local sp so; sp=$(stat -c '%a' "$s"); so=$(stat -c '%U' "$s")
        [[ "$so" == "root" ]] && _pass "[${s}] 소유자=root" || { _fail "[${s}] 소유자=${so}"; ok=false; }
        [[ "$sp" -le 400 ]]   && _pass "[${s}] 권한=${sp}"  || { _fail "[${s}] 권한=${sp} (≤400 필요)"; ok=false; }
    else
        _fail "[${s}] 파일 없음 (shadow 미사용)"; ok=false
    fi
    local plain
    plain=$(awk -F: '$2 != "x" && $2 != "!" && $2 != "*" && $2 != "" {print $1}' "$p" 2>/dev/null)
    [[ -z "$plain" ]] \
        && _pass "[${p}] 모든 계정 shadow 처리됨" \
        || { _fail "[${p}] 평문 패스워드 계정: ${plain}"; ok=false; }
    $ok && return 0 || return 1
}
_u04_fix() {
    local s="/etc/shadow"
    [[ -f "$s" ]] && {
        do_fix "[${s}] chown root:root" "chown root:root '$s'"
        do_fix "[${s}] chmod 400"       "chmod 400 '$s'"
    }
    command -v pwconv &>/dev/null && do_fix "pwconv 실행" "pwconv"
}

# ── U-05: root 이외 UID=0 금지 [AUTO] ────────────────────────────────────────
_u05_check() {
    local bad
    bad=$(awk -F: '$3 == 0 && $1 != "root" {print $1}' /etc/passwd)
    [[ -z "$bad" ]] \
        && { _pass "[/etc/passwd] root 외 UID=0 계정 없음"; return 0; } \
        || { _fail "[/etc/passwd] UID=0 비root 계정: ${bad}"; return 1; }
}
_u05_fix() {
    # UID 0 계정은 삭제 대신 UID 변경 — 실제 UID는 환경에 따라 다르므로 안내
    local bad
    bad=$(awk -F: '$3 == 0 && $1 != "root" {print $1}' /etc/passwd)
    for acct in $bad; do
        do_fix "[/etc/passwd] ${acct} UID 변경 (500+)" \
            "usermod -u \$(awk -F: 'BEGIN{m=500} \$3>=m{m=\$3+1} END{print m}' /etc/passwd) '$acct'"
    done
}

# ── U-06: 사용자 계정 su 기능 제한 [AUTO] ────────────────────────────────────
_u06_check() {
    local ok=true pf="/etc/pam.d/su" sb="/usr/bin/su"
    grep -qE "^auth\s+required\s+pam_wheel.so" "$pf" 2>/dev/null \
        && _pass "[${pf}] pam_wheel.so 설정됨" \
        || { _fail "[${pf}] pam_wheel.so 미설정"; ok=false; }
    [[ "$(stat -c '%a' "$sb" 2>/dev/null)" == "4750" ]] \
        && _pass "[${sb}] 권한=4750" \
        || { _fail "[${sb}] 권한=$(stat -c '%a' "$sb" 2>/dev/null) (4750 필요)"; ok=false; }
    $ok && return 0 || return 1
}
_u06_fix() {
    local f="/etc/pam.d/su"; backup_file "$f"
    _warn "wheel 그룹 구성원 확인: $(getent group wheel | awk -F: '{print ($4=="" ? "없음(su 전면 차단 위험)" : $4)}')"
    if grep -qE "^#.*pam_wheel.so" "$f"; then
        do_fix "[${f}] pam_wheel.so 주석 해제" "sed -i 's/^#\s*\(auth\s*required\s*pam_wheel.so.*\)/\1/' '$f'"
    elif ! grep -qE "^auth\s+required\s+pam_wheel.so" "$f"; then
        do_fix "[${f}] pam_wheel.so 추가" "echo 'auth  required  pam_wheel.so use_uid' >> '$f'"
    fi
    do_fix "[/usr/bin/su] chgrp wheel" "chgrp wheel /usr/bin/su"
    do_fix "[/usr/bin/su] chmod 4750"  "chmod 4750  /usr/bin/su"
}

# ── U-07: 불필요한 계정 제거 [MANUAL] ────────────────────────────────────────
# 어떤 계정이 "불필요"한지는 운영 환경에 따라 다름 → 수동 판단 필요
_u07_check() {
    local sys_accounts="games ftp halt shutdown sync operator lp mail news uucp gopher"
    local found=false
    for acct in $sys_accounts; do
        if id "$acct" &>/dev/null; then
            _warn "[/etc/passwd] 잠재적 불필요 계정 존재: ${acct} (UID=$(id -u "$acct"))"
            found=true
        fi
    done
    $found || _pass "[/etc/passwd] 표준 불필요 계정 없음"
    return 1  # 항상 수동 확인 유도
}

# ── U-08: 관리자 그룹 최소 계정 [MANUAL] ─────────────────────────────────────
_u08_check() {
    local grp_members
    grp_members=$(getent group wheel sudo 2>/dev/null | awk -F: '{print $1": "$4}')
    _info "[/etc/group] 관리자 그룹 구성원:"
    echo "$grp_members" | while read -r line; do _info "  ${line}"; done
    _warn "관리자 그룹 구성원이 최소화되어 있는지 수동 검토 필요"
    return 1
}

# ── U-09: 계정 없는 GID 금지 [AUTO] ──────────────────────────────────────────
_u09_check() {
    local ok=true orphan=false
    # AL2023/Rocky 시스템 하드웨어·OS 그룹 — 멤버 없어도 정상
    local system_groups="kmem dialout fax floppy tape audio operator video kvm render sgx utmp utempter screen man games lock input ssh_keys stapusr stapsys stapdev systemd-journal docker tty disk lp adm sys shadow cdrom wheel"
    while IFS=: read -r gname _ gid members; do
        [[ -z "$members" ]] || continue
        echo "$system_groups" | grep -qw "$gname" && continue
        grep -q ":${gid}:" /etc/passwd 2>/dev/null || {
            _fail "[/etc/group] orphan GID: ${gid} (${gname})"
            ok=false; orphan=true
        }
    done < /etc/group
    $orphan || _pass "[/etc/group] orphan GID 없음"
    $ok && return 0 || return 1
}

_u09_fix() {
    _manual "orphan GID 확인 후 groupdel <그룹명> 으로 수동 제거"
}

# ── U-10: 동일 UID 금지 [AUTO] ───────────────────────────────────────────────
_u10_check() {
    local dup
    dup=$(awk -F: '{print $3}' /etc/passwd | sort | uniq -d)
    [[ -z "$dup" ]] \
        && { _pass "[/etc/passwd] 중복 UID 없음"; return 0; } \
        || { _fail "[/etc/passwd] 중복 UID: ${dup}"; return 1; }
}
_u10_fix() {
    _manual "중복 UID 계정을 식별하여 usermod -u <새UID> <계정명> 으로 수동 변경 필요"
}

# ── U-11: 사용자 Shell 점검 [MANUAL] ─────────────────────────────────────────
_u11_check() {
    local ok=true valid_shells
    valid_shells=$(cat /etc/shells 2>/dev/null)
    while IFS=: read -r user _ uid _ _ _ shell; do
        [[ "$uid" -lt 1000 && "$uid" -ne 0 ]] || continue
        [[ "$shell" == "/sbin/nologin" || "$shell" == "/bin/false" || "$shell" == "/usr/sbin/nologin" ]] && continue
        echo "$valid_shells" | grep -qx "$shell" || {
            _fail "[/etc/passwd] ${user}: 유효하지 않은 shell=${shell}"
            ok=false
        }
    done < /etc/passwd
    $ok && { _pass "[/etc/passwd] 모든 시스템 계정 shell 정상"; return 0; } || return 1
}

# ── U-12: 세션 종료 시간 설정 [AUTO] ─────────────────────────────────────────
_u12_check() {
    local tmout src
    tmout=$(grep -rE "^TMOUT[[:space:]]*=" /etc/profile /etc/profile.d/ 2>/dev/null | head -1 | awk -F= '{print $2}' | tr -d ' ')
    src=$(grep -rlE "^TMOUT[[:space:]]*=" /etc/profile /etc/profile.d/ 2>/dev/null | head -1)
    src="${src:-미설정}"
    [[ -n "$tmout" && "$tmout" -le 600 ]] \
        && { _pass "[${src}] TMOUT=${tmout}s"; return 0; } \
        || { _fail "[${src}] TMOUT=${tmout:-없음} (≤600s 필요)"; return 1; }
}
_u12_fix() {
    local t="/etc/profile.d/kisa-session.sh"
    do_fix "[${t}] TMOUT=600 설정" "cat > '$t' << 'EOF'
# KISA U-12
TMOUT=600
export TMOUT
readonly TMOUT
EOF"
}

# ── U-13: 안전한 비밀번호 암호화 알고리즘 사용 [AUTO] ────────────────────────
_u13_check() {
    local ok=true
    local algo
    # login.defs ENCRYPT_METHOD
    algo=$(awk '/^ENCRYPT_METHOD/{print $2}' /etc/login.defs 2>/dev/null)
    [[ "$algo" == "SHA512" || "$algo" == "YESCRYPT" || "$algo" == "SHA256" ]] \
        && _pass "[/etc/login.defs] ENCRYPT_METHOD=${algo}" \
        || { _fail "[/etc/login.defs] ENCRYPT_METHOD=${algo:-없음} (SHA512/YESCRYPT 권장)"; ok=false; }
    # /etc/shadow 실제 해시 형식 확인 ($6=sha512, $y=yescrypt)
    local bad_hash
    bad_hash=$(awk -F: '$2 ~ /^\$/ && $2 !~ /^\$6\$|^\$y\$|^\$5\$/ {print $1": "$2}' /etc/shadow 2>/dev/null)
    [[ -z "$bad_hash" ]] \
        && _pass "[/etc/shadow] 모든 계정 강력한 해시 사용" \
        || { _fail "[/etc/shadow] 취약한 해시 알고리즘 계정: ${bad_hash}"; ok=false; }
    $ok && return 0 || return 1
}
_u13_fix() {
    local lf="/etc/login.defs"; backup_file "$lf"
    grep -q "^ENCRYPT_METHOD" "$lf" \
        && do_fix "[${lf}] ENCRYPT_METHOD SHA512 수정" "sed -i 's/^ENCRYPT_METHOD.*/ENCRYPT_METHOD SHA512/' '$lf'" \
        || do_fix "[${lf}] ENCRYPT_METHOD SHA512 추가" "echo 'ENCRYPT_METHOD SHA512' >> '$lf'"
    _warn "기존 계정의 해시 변경은 다음 패스워드 변경 시 적용됨 (강제 적용: chage -d 0 <user>)"
}

# =============================================================================
# 2. 파일 및 디렉토리 관리 (U-14 ~ U-33)
# =============================================================================

# ── U-14: root 홈·PATH 권한 및 경로 설정 [AUTO] ───────────────────────────────
_u14_check() {
    local ok=true
    # root 홈 디렉토리 권한
    local rhome; rhome=$(eval echo "~root")
    local rp; rp=$(stat -c '%a' "$rhome" 2>/dev/null)
    [[ "$rp" -le 700 ]] \
        && _pass "[${rhome}] 권한=${rp}" \
        || { _fail "[${rhome}] 권한=${rp} (≤700 필요)"; ok=false; }
    # PATH에 '.' 또는 빈 항목 포함 여부
    local root_path
    root_path=$(su -c 'echo $PATH' root 2>/dev/null || echo "$PATH")
    echo "$root_path" | tr ':' '\n' | grep -qE "^\.$|^$" \
        && { _fail "[PATH] '.' 또는 빈 경로 포함: ${root_path}"; ok=false; } \
        || _pass "[PATH] 현재 경로(.)/빈경로 없음"
    $ok && return 0 || return 1
}
_u14_fix() {
    local rhome; rhome=$(eval echo "~root")
    local rp; rp=$(stat -c '%a' "$rhome" 2>/dev/null)
    [[ "$rp" -le 700 ]] || do_fix "[${rhome}] chmod 700" "chmod 700 '$rhome'"
    _warn "[PATH] root .profile/.bash_profile에서 PATH의 '.' 및 빈 항목 수동 제거 필요"
}

# ── U-15: 파일·디렉토리 소유자 설정 [MANUAL] ─────────────────────────────────
# 소유자 없는 파일은 존재만 알리고 수동 처리 필요 (자동 chown 위험)
_u15_check() {
    _info "[/] 소유자 없는 파일 검색 중 (최대 50개 출력)..."
    local noowner
    noowner=$(find / -xdev \( -nouser -o -nogroup \) \
        -not -path "/var/lib/docker/*" \
        -not -path "/run/containerd/*" \
        -not -path "/var/lib/containerd/*" \
        -not -path "/home/*/.local/share/buildkit/*" \
        -not -path "/home/*/.local/share/containerd/*" \
        -not -path "/home/*/.local/share/containers/*" \
        -not -path "/var/lib/containers/*" \
        -print 2>/dev/null | head -50)
    if [[ -z "$noowner" ]]; then
        _pass "[/] 소유자 없는 파일/디렉토리 없음"
        return 0
    else
        echo "$noowner" | while read -r f; do _fail "[${f}] 소유자 없음"; done
        return 1
    fi
}

# ── U-16: /etc/passwd 소유자·권한 [AUTO] ────────────────────────────────────
_u16_check() {
    local ok=true f="/etc/passwd"
    local o p; o=$(stat -c '%U' "$f"); p=$(stat -c '%a' "$f")
    [[ "$o" == "root" ]] && _pass "[${f}] 소유자=root" || { _fail "[${f}] 소유자=${o}"; ok=false; }
    [[ "$p" -le 644 ]]   && _pass "[${f}] 권한=${p}"   || { _fail "[${f}] 권한=${p} (≤644 필요)"; ok=false; }
    $ok && return 0 || return 1
}
_u16_fix() {
    do_fix "[/etc/passwd] chown root:root" "chown root:root /etc/passwd"
    do_fix "[/etc/passwd] chmod 644"       "chmod 644 /etc/passwd"
}

# ── U-17: 시스템 시작 스크립트 권한 설정 [AUTO] ──────────────────────────────
_u17_check() {
    local ok=true f o p checked=false
    for target in /etc/rc.d/rc.local /etc/rc.local /etc/init.d; do
        [[ ! -e "$target" ]] && continue
        while IFS= read -r f; do
            checked=true
            o=$(stat -c '%U' "$f" 2>/dev/null)
            p=$(stat -c '%a' "$f" 2>/dev/null)
            [[ "$o" == "root" ]] \
                && _pass "[${f}] 소유자=root" \
                || { _fail "[${f}] 소유자=${o} (root 필요)"; ok=false; }
            [[ "$p" -le 755 ]] \
                && _pass "[${f}] 권한=${p}" \
                || { _fail "[${f}] 권한=${p} (≤755 필요)"; ok=false; }
        done < <(find "$target" -maxdepth 1 -type f 2>/dev/null)
    done
    $checked || _pass "[시작 스크립트] 점검 대상 파일 없음 (양호)"
    $ok && return 0 || return 1
}

_u17_fix() {
    local f o p
    for target in /etc/rc.d/rc.local /etc/rc.local /etc/init.d; do
        [[ ! -e "$target" ]] && continue
        while IFS= read -r f; do
            o=$(stat -c '%U' "$f" 2>/dev/null)
            p=$(stat -c '%a' "$f" 2>/dev/null)
            [[ "$o" == "root" ]] || do_fix "[${f}] chown root" "chown root:root '$f'"
            [[ "$p" -le 755 ]]   || do_fix "[${f}] chmod 755"  "chmod 755 '$f'"
        done < <(find "$target" -maxdepth 1 -type f 2>/dev/null)
    done
}

# ── U-18: /etc/shadow 소유자·권한 [AUTO] ────────────────────────────────────
_u18_check() {
    local ok=true f="/etc/shadow"
    [[ ! -f "$f" ]] && { _fail "[${f}] 파일 없음"; return 1; }
    local o p; o=$(stat -c '%U' "$f"); p=$(stat -c '%a' "$f")
    [[ "$o" == "root" ]] && _pass "[${f}] 소유자=root" || { _fail "[${f}] 소유자=${o}"; ok=false; }
    [[ "$p" -le 400 ]]   && _pass "[${f}] 권한=${p}"   || { _fail "[${f}] 권한=${p} (≤400 필요)"; ok=false; }
    $ok && return 0 || return 1
}
_u18_fix() {
    do_fix "[/etc/shadow] chown root:root" "chown root:root /etc/shadow"
    do_fix "[/etc/shadow] chmod 400"       "chmod 400 /etc/shadow"
}

# ── U-19: /etc/hosts 소유자·권한 [AUTO] ─────────────────────────────────────
_u19_check() {
    local ok=true f="/etc/hosts"
    local o p; o=$(stat -c '%U' "$f" 2>/dev/null); p=$(stat -c '%a' "$f" 2>/dev/null)
    [[ "$o" == "root" ]] && _pass "[${f}] 소유자=root" || { _fail "[${f}] 소유자=${o}"; ok=false; }
    [[ "$p" -le 644 ]]   && _pass "[${f}] 권한=${p}"   || { _fail "[${f}] 권한=${p} (≤644 필요)"; ok=false; }
    $ok && return 0 || return 1
}
_u19_fix() {
    do_fix "[/etc/hosts] chown root:root" "chown root:root /etc/hosts"
    do_fix "[/etc/hosts] chmod 644"       "chmod 644 /etc/hosts"
}

# ── U-20: /etc/(x)inetd.conf 소유자·권한 [AUTO] ──────────────────────────────
_u20_check() {
    local ok=true found=false
    for f in /etc/inetd.conf /etc/xinetd.conf /etc/xinetd.d/*; do
        [[ ! -e "$f" ]] && continue
        found=true
        local o p; o=$(stat -c '%U' "$f"); p=$(stat -c '%a' "$f")
        [[ "$o" == "root" ]] && _pass "[${f}] 소유자=root" || { _fail "[${f}] 소유자=${o}"; ok=false; }
        [[ "$p" -le 600 ]]   && _pass "[${f}] 권한=${p}"   || { _fail "[${f}] 권한=${p} (≤600 필요)"; ok=false; }
    done
    $found || _na "[inetd/xinetd] AL2023에 inetd/xinetd 미설치 — 해당없음"
    $ok && return 0 || return 1
}
_u20_fix() {
    for f in /etc/inetd.conf /etc/xinetd.conf /etc/xinetd.d/*; do
        [[ ! -f "$f" ]] && continue
        do_fix "[${f}] chown root, chmod 600" "chown root:root '$f' && chmod 600 '$f'"
    done
}

# ── U-21: /etc/(r)syslog.conf 소유자·권한 [AUTO] ─────────────────────────────
_u21_check() {
    local ok=true found=false
    for f in /etc/rsyslog.conf /etc/syslog.conf; do
        [[ ! -f "$f" ]] && continue
        found=true
        local o p; o=$(stat -c '%U' "$f"); p=$(stat -c '%a' "$f")
        [[ "$o" == "root" ]] && _pass "[${f}] 소유자=root" || { _fail "[${f}] 소유자=${o}"; ok=false; }
        [[ "$p" -le 640 ]]   && _pass "[${f}] 권한=${p}"   || { _fail "[${f}] 권한=${p} (≤640 필요)"; ok=false; }
    done
    $found || { _na "[rsyslog/syslog.conf] 파일 없음 (rsyslog 미설치 — 해당없음)"; return 0; }
    $ok && return 0 || return 1
}

_u21_fix() {
    for f in /etc/rsyslog.conf /etc/syslog.conf; do
        [[ ! -f "$f" ]] && continue
        do_fix "[${f}] chown root:root" "chown root:root '$f'"
        do_fix "[${f}] chmod 640"       "chmod 640 '$f'"
    done
}

# ── U-22: /etc/services 소유자·권한 [AUTO] ───────────────────────────────────
_u22_check() {
    local ok=true f="/etc/services"
    local o p; o=$(stat -c '%U' "$f" 2>/dev/null); p=$(stat -c '%a' "$f" 2>/dev/null)
    [[ "$o" == "root" ]] && _pass "[${f}] 소유자=root" || { _fail "[${f}] 소유자=${o}"; ok=false; }
    [[ "$p" -le 644 ]]   && _pass "[${f}] 권한=${p}"   || { _fail "[${f}] 권한=${p} (≤644 필요)"; ok=false; }
    $ok && return 0 || return 1
}
_u22_fix() {
    do_fix "[/etc/services] chown root:root" "chown root:root /etc/services"
    do_fix "[/etc/services] chmod 644"       "chmod 644 /etc/services"
}

# ── U-23: SUID·SGID·Sticky bit 파일 점검 [MANUAL] ────────────────────────────
# 자동 제거 시 시스템 기능 손상 위험 → 목록만 출력
_u23_check() {
    _info "[/] SUID/SGID 설정 파일 검색 중..."
    local list
    list=$(find / -xdev \( -perm -4000 -o -perm -2000 \) -type f \
        -not -path "/var/lib/docker/*" \
        -not -path "/run/containerd/*" \
        -not -path "/var/lib/containerd/*" \
        -print 2>/dev/null)
    local count; count=$(echo "$list" | grep -c . 2>/dev/null || echo 0)
    if [[ "$count" -eq 0 ]]; then
        _pass "[/] SUID/SGID 파일 없음"
        return 0
    fi
    _warn "[/] SUID/SGID 파일 ${count}개 발견 (불필요한 항목 수동 제거 필요):"
    echo "$list" | while read -r f; do
        local perm; perm=$(stat -c '%a %U %n' "$f" 2>/dev/null)
        _info "  ${perm}"
    done
    return 1
}

# ── U-24: 사용자·시스템 환경변수 파일 소유자·권한 [AUTO] ─────────────────────
_u24_check() {
    local ok=true
    local env_files=("/etc/profile" "/etc/bashrc" "/etc/environment")
    # 각 일반사용자 홈의 환경파일도 확인
    for f in "${env_files[@]}" /etc/profile.d/*.sh; do
        [[ ! -f "$f" ]] && continue
        local o p; o=$(stat -c '%U' "$f"); p=$(stat -c '%a' "$f")
        [[ "$o" == "root" ]] && _pass "[${f}] 소유자=root" || { _fail "[${f}] 소유자=${o}"; ok=false; }
        [[ "$p" -le 644 ]]   && _pass "[${f}] 권한=${p}"   || { _fail "[${f}] 권한=${p} (≤644 필요)"; ok=false; }
    done
    $ok && return 0 || return 1
}
_u24_fix() {
    for f in /etc/profile /etc/bashrc /etc/environment /etc/profile.d/*.sh; do
        [[ ! -f "$f" ]] && continue
        do_fix "[${f}] chown root:root" "chown root:root '$f'"
        [[ "$(stat -c '%a' "$f")" -le 644 ]] || do_fix "[${f}] chmod 644" "chmod 644 '$f'"
    done
}

# ── U-25: world writable 파일 점검 [MANUAL] ──────────────────────────────────
_u25_check() {
    _info "[/] world writable 파일 검색 중 (최대 50개)..."
    local list
    list=$(find / -xdev -perm -o+w -not -type l \
           -not -path "/proc/*" \
           -not -path "/sys/*" \
           -not -path "/tmp" \
           -not -path "/var/tmp" \
           -not -path "/var/tmp/*" \
           -not -path "/var/lib/docker/*" \
           -not -path "/run/*" \
           -print 2>/dev/null | head -50)
    [[ -z "$list" ]] && { _pass "[/] world writable 파일 없음"; return 0; }
    echo "$list" | while read -r f; do _fail "[${f}] world writable"; done
    return 1
}

# ── U-26: /dev 비정상 파일 점검 [INFO] ───────────────────────────────────────
_u26_check() {
    _info "[/dev] 일반 파일(non-device) 검색 중..."
    local list
    list=$(find /dev -not -type d -not -type l -not -type c -not -type b -not -type p -not -type s 2>/dev/null)
    [[ -z "$list" ]] \
        && _pass "[/dev] 비정상 파일 없음" \
        || { echo "$list" | while read -r f; do _warn "[${f}] /dev 내 일반파일 존재"; done; }
}

# ── U-27: $HOME/.rhosts, hosts.equiv 사용 금지 [AUTO] ────────────────────────
_u27_check() {
    local ok=true
    # /etc/hosts.equiv
    if [[ -f "/etc/hosts.equiv" ]]; then
        _fail "[/etc/hosts.equiv] 파일 존재 — 즉시 제거 필요"; ok=false
    else
        _pass "[/etc/hosts.equiv] 없음"
    fi
    # 각 사용자 홈 .rhosts
    local found_rhosts=false
    while IFS=: read -r user _ _ _ _ home _; do
        [[ -f "${home}/.rhosts" ]] && {
            _fail "[${home}/.rhosts] 존재 (계정: ${user})"; ok=false; found_rhosts=true
        }
    done < /etc/passwd
    $found_rhosts || _pass "[~/.rhosts] 모든 홈 디렉토리에 .rhosts 없음"
    $ok && return 0 || return 1
}
_u27_fix() {
    [[ -f "/etc/hosts.equiv" ]] && do_fix "[/etc/hosts.equiv] 제거" "rm -f /etc/hosts.equiv"
    while IFS=: read -r _ _ _ _ _ home _; do
        [[ -f "${home}/.rhosts" ]] && do_fix "[${home}/.rhosts] 제거" "rm -f '${home}/.rhosts'"
    done < /etc/passwd
}

# ── U-28: 접속 IP 및 포트 제한 [MANUAL] ──────────────────────────────────────
_u28_check() {
    local ok=true
    # firewalld 상태
    if systemctl is-active firewalld &>/dev/null; then
        _pass "[firewalld] 활성화됨"
        _info "[firewalld] 현재 zone 규칙:"
        firewall-cmd --list-all 2>/dev/null | head -20 | while read -r line; do _info "  ${line}"; done
    else
        _warn "[firewalld] 비활성화 — IP/포트 제한 방화벽 미동작"
        ok=false
    fi
    # TCP Wrappers (hosts.allow/deny)
    for f in /etc/hosts.allow /etc/hosts.deny; do
        [[ -f "$f" ]] \
            && _info "[${f}] 존재: $(grep -vc '^\s*#\|^\s*$' "$f" 2>/dev/null || echo 0)개 규칙" \
            || _warn "[${f}] 파일 없음"
    done
    $ok && return 0 || return 1
}

# ── U-29: hosts.lpd 소유자·권한 [AUTO] ──────────────────────────────────────
_u29_check() {
    local f="/etc/hosts.lpd"
    [[ ! -f "$f" ]] && { _na "[${f}] 파일 없음 (lpd 미사용 — 양호)"; return 0; }
    local o p; o=$(stat -c '%U' "$f"); p=$(stat -c '%a' "$f")
    local ok=true
    [[ "$o" == "root" ]] && _pass "[${f}] 소유자=root" || { _fail "[${f}] 소유자=${o}"; ok=false; }
    [[ "$p" -le 600 ]]   && _pass "[${f}] 권한=${p}"   || { _fail "[${f}] 권한=${p} (≤600 필요)"; ok=false; }
    $ok && return 0 || return 1
}
_u29_fix() {
    local f="/etc/hosts.lpd"
    [[ -f "$f" ]] && {
        do_fix "[${f}] chown root:root" "chown root:root '$f'"
        do_fix "[${f}] chmod 600"       "chmod 600 '$f'"
    }
}

# ── U-30: UMASK 설정 관리 [AUTO] ─────────────────────────────────────────────
_u30_check() {
    local umask_val src
    umask_val=$(grep -rE "^umask[[:space:]]+" /etc/profile /etc/profile.d/ 2>/dev/null | head -1 | awk '{print $2}')
    src=$(grep -rlE "^umask[[:space:]]+" /etc/profile /etc/profile.d/ 2>/dev/null | head -1)
    src="${src:-미설정}"
    [[ "$umask_val" == "022" || "$umask_val" == "027" ]] \
        && { _pass "[${src}] umask=${umask_val}"; return 0; } \
        || { _fail "[${src}] umask=${umask_val:-없음} (022 또는 027 필요)"; return 1; }
}
_u30_fix() {
    local t="/etc/profile.d/kisa-umask.sh"
    do_fix "[${t}] umask=022 설정" "echo 'umask 022' > '$t'"
}

# ── U-31: 홈 디렉토리 소유자·권한 [AUTO] ─────────────────────────────────────
_u31_check() {
    local ok=true
    while IFS=: read -r user _ uid _ _ home _; do
        [[ "$uid" -lt 500 ]] && continue
        [[ ! -d "$home" ]] && continue
        [[ "$home" == "/" ]] && continue
        [[ "$home" == "/nonexistent" ]] && continue
        [[ "$home" =~ ^/var/lib/ && "$uid" -lt 1000 ]] && continue
        local o p; o=$(stat -c '%U' "$home"); p=$(stat -c '%a' "$home")
        [[ "$o" == "$user" ]] \
            && _pass "[${home}] 소유자=${user}" \
            || { _fail "[${home}] 소유자=${o} (${user} 필요)"; ok=false; }
        [[ "$p" -le 755 ]] \
            && _pass "[${home}] 권한=${p}" \
            || { _fail "[${home}] 권한=${p} (≤755 필요)"; ok=false; }
    done < /etc/passwd
    $ok && return 0 || return 1
}

_u31_fix() {
    while IFS=: read -r user _ uid _ _ home _; do
        [[ "$uid" -lt 500 ]] && continue
        [[ ! -d "$home" ]] && continue
        [[ "$home" == "/" ]] && continue
        [[ "$home" == "/nonexistent" ]] && continue
        [[ "$home" =~ ^/var/lib/ && "$uid" -lt 1000 ]] && continue
        local o p; o=$(stat -c '%U' "$home"); p=$(stat -c '%a' "$home")
        [[ "$o" == "$user" ]] || do_fix "[${home}] chown ${user}" "chown '$user' '$home'"
        [[ "$p" -le 755 ]]    || do_fix "[${home}] chmod 755"     "chmod 755 '$home'"
    done < /etc/passwd
}

# ── U-32: 홈 디렉토리 존재 관리 [INFO] ───────────────────────────────────────
_u32_check() {
    local found=false
    while IFS=: read -r user _ uid _ _ home _; do
        [[ "$uid" -lt 500 ]] && continue
        [[ -d "$home" ]] || {
            _warn "[/etc/passwd] ${user}: 홈 디렉토리 ${home} 없음"
            found=true
        }
    done < /etc/passwd
    $found || _pass "[/etc/passwd] 모든 사용자 홈 디렉토리 존재"
}

# ── U-33: 숨겨진 파일·디렉토리 검색 및 제거 [MANUAL] ───────────────────────────
_u33_check() {
    _info "[/] 숨겨진 파일·디렉토리 검색 중 (최대 30개)..."
    local list
    list=$(find / -xdev -name ".*" \
           -not -path "/proc/*" \
           -not -path "/sys/*" \
           -not -path "/root/.*" \
           -not -path "/home/*/.*" \
           -not -path "/boot/*" \
           -not -path "/etc/*" \
           -not -path "/var/*" \
           -not -path "/run/*" \
           -not -path "/usr/*" \
           -not -path "/app/*" \
           2>/dev/null | head -30)
    if [[ -z "$list" ]]; then
        _pass "[/] 비정상 위치의 숨김 파일 없음"
        return 0
    fi
    echo "$list" | while read -r f; do _fail "[${f}] 숨김 파일 존재"; done
    return 1
}

# =============================================================================
# 3. 서비스 관리 (U-34 ~ U-63)
# =============================================================================

# ── U-34: Finger 서비스 비활성화 [AUTO] ──────────────────────────────────────
_u34_check() {
    local active; active=$(systemctl is-active finger 2>/dev/null || echo "inactive")
    [[ "$active" != "active" ]] \
        && { _pass "[systemctl] finger: 비활성/미설치"; return 0; } \
        || { _fail "[systemctl] finger: active (비활성화 필요)"; return 1; }
}

_u34_fix() {
    do_fix "[systemctl] finger stop+disable" "systemctl stop finger 2>/dev/null; systemctl disable finger 2>/dev/null || true"
}

# ── U-35: 공유 서비스 익명 접근 제한 (Anonymous FTP) [AUTO] ──────────────────
_u35_check() {
    local ok=true
    # vsftpd
    if systemctl is-active vsftpd &>/dev/null; then
        local anon
        anon=$(grep -E "^anonymous_enable" /etc/vsftpd/vsftpd.conf 2>/dev/null | awk -F= '{print $2}' | tr -d ' ')
        [[ "${anon,,}" == "no" || "${anon,,}" == "no" ]] \
            && _pass "[/etc/vsftpd/vsftpd.conf] anonymous_enable=NO" \
            || { _fail "[/etc/vsftpd/vsftpd.conf] anonymous_enable=${anon:-미설정} (NO 필요)"; ok=false; }
    else
        _pass "[systemctl] vsftpd: 비활성/미설치"
    fi
    # Samba anonymous
    if systemctl is-active smb &>/dev/null || systemctl is-active nmb &>/dev/null; then
        _warn "[smb] Samba 활성 — smb.conf의 guest/anonymous 접근 수동 확인 필요"
        ok=false
    else
        _pass "[systemctl] smb/nmb: 비활성/미설치"
    fi
    $ok && return 0 || return 1
}
_u35_fix() {
    local vc="/etc/vsftpd/vsftpd.conf"
    if [[ -f "$vc" ]]; then
        backup_file "$vc"
        grep -q "^anonymous_enable" "$vc" \
            && do_fix "[${vc}] anonymous_enable=NO 수정" "sed -i 's/^anonymous_enable.*/anonymous_enable=NO/' '$vc'" \
            || do_fix "[${vc}] anonymous_enable=NO 추가" "echo 'anonymous_enable=NO' >> '$vc'"
        do_fix "vsftpd restart" "systemctl restart vsftpd 2>/dev/null || true"
    fi
}

# ── U-36: r 계열 서비스 비활성화 [AUTO] ──────────────────────────────────────
_u36_check() {
    local ok=true
    for svc in rsh rlogin rexec rsh.socket rlogin.socket rexec.socket; do
        local active; active=$(systemctl is-active "$svc" 2>/dev/null || echo "inactive")
        [[ "$active" == "active" ]] \
            && { _fail "[systemctl] ${svc}: active"; ok=false; } \
            || _pass "[systemctl] ${svc}: 비활성/미설치"
    done
    $ok && return 0 || return 1
}
_u36_fix() {
    for svc in rsh rlogin rexec rsh.socket rlogin.socket rexec.socket; do
        [[ "$(systemctl is-active "$svc" 2>/dev/null || echo inactive)" == "active" ]] && {
            do_fix "[systemctl] ${svc} stop"    "systemctl stop    '$svc' 2>/dev/null || true"
            do_fix "[systemctl] ${svc} disable" "systemctl disable '$svc' 2>/dev/null || true"
        }
    done
}

# ── U-37: crontab 파일 권한 설정 [AUTO] ──────────────────────────────────────
_u37_check() {
    local ok=true
    local -A expected=(["/etc/crontab"]=640 ["/etc/cron.deny"]=640
                       ["/etc/at.deny"]=640 ["/etc/at.allow"]=640
                       ["/var/spool/cron"]=700)
    for f in "${!expected[@]}"; do
        [[ ! -e "$f" ]] && continue
        local p; p=$(stat -c '%a' "$f")
        [[ "$p" -le "${expected[$f]}" ]] \
            && _pass "[${f}] 권한=${p}" \
            || { _fail "[${f}] 권한=${p} (≤${expected[$f]} 필요)"; ok=false; }
    done
    for d in /etc/cron.d /etc/cron.daily /etc/cron.hourly /etc/cron.monthly /etc/cron.weekly; do
        [[ ! -d "$d" ]] && continue
        local p; p=$(stat -c '%a' "$d")
        [[ "$p" -le 640 ]] && _pass "[${d}] 권한=${p}" || { _fail "[${d}] 권한=${p} (≤640 필요)"; ok=false; }
    done
    $ok && return 0 || return 1
}
_u37_fix() {
    local -A fix=(["/etc/crontab"]=640 ["/etc/cron.deny"]=640
                  ["/etc/at.deny"]=640 ["/etc/at.allow"]=640
                  ["/var/spool/cron"]=700)
    for f in "${!fix[@]}"; do
        [[ ! -e "$f" ]] && continue
        [[ "$(stat -c '%a' "$f")" -le "${fix[$f]}" ]] || do_fix "[${f}] chmod ${fix[$f]}" "chmod ${fix[$f]} '$f'"
    done
    for d in /etc/cron.d /etc/cron.daily /etc/cron.hourly /etc/cron.monthly /etc/cron.weekly; do
        [[ -d "$d" && "$(stat -c '%a' "$d")" -gt 640 ]] && do_fix "[${d}] chmod 640" "chmod 640 '$d'"
    done
}

# ── U-38: DoS 취약 서비스 비활성화 [AUTO] ────────────────────────────────────
_u38_check() {
    local ok=true
    for svc in chargen daytime discard echo time chargen.socket daytime.socket discard.socket echo.socket time.socket; do
        local active; active=$(systemctl is-active "$svc" 2>/dev/null || echo "inactive")
        [[ "$active" == "active" ]] \
            && { _fail "[systemctl] ${svc}: active"; ok=false; } \
            || _pass "[systemctl] ${svc}: 비활성/미설치"
    done
    $ok && return 0 || return 1
}
_u38_fix() {
    for svc in chargen daytime discard echo time chargen.socket daytime.socket discard.socket echo.socket time.socket; do
        [[ "$(systemctl is-active "$svc" 2>/dev/null || echo inactive)" == "active" ]] && {
            do_fix "[systemctl] ${svc} stop+disable" "systemctl stop '$svc' 2>/dev/null; systemctl disable '$svc' 2>/dev/null || true"
        }
    done
}

# ── U-39: NFS 서비스 비활성화 [AUTO] ─────────────────────────────────────────
_u39_check() {
    local ok=true
    for svc in nfs-server nfs-mountd; do
        local active; active=$(systemctl is-active "$svc" 2>/dev/null || echo "inactive")
        [[ "$active" == "active" ]] \
            && { _fail "[systemctl] ${svc}: active"; ok=false; } \
            || _pass "[systemctl] ${svc}: 비활성/미설치"
    done
    $ok && return 0 || return 1
}
_u39_fix() {
    for svc in nfs-server nfs-mountd; do
        [[ "$(systemctl is-active "$svc" 2>/dev/null || echo inactive)" == "active" ]] && {
            do_fix "[systemctl] ${svc} stop+disable" "systemctl stop '$svc'; systemctl disable '$svc' 2>/dev/null || true"
        }
    done
}

# ── U-40: NFS 접근통제 [MANUAL] ──────────────────────────────────────────────
_u40_check() {
    local f="/etc/exports"
    [[ ! -f "$f" ]] && { _pass "[${f}] 파일 없음 (NFS 미사용 — 양호)"; return 0; }
    local bad
    bad=$(grep -v "^\s*#\|^\s*$" "$f" | grep -E "\*|no_root_squash")
    [[ -z "$bad" ]] \
        && { _pass "[${f}] 와일드카드(*)/no_root_squash 없음"; return 0; } \
        || { _fail "[${f}] 위험 설정 발견:"; echo "$bad" | while read -r line; do _fail "  ${line}"; done; return 1; }
}

# ── U-41: automountd 제거 [AUTO] ─────────────────────────────────────────────
_u41_check() {
    local active; active=$(systemctl is-active autofs 2>/dev/null || echo "inactive")
    [[ "$active" != "active" ]] \
        && { _pass "[systemctl] autofs: 비활성/미설치"; return 0; } \
        || { _fail "[systemctl] autofs: active (제거 필요)"; return 1; }
}

_u41_fix() {
    do_fix "[systemctl] autofs stop+disable" "systemctl stop autofs 2>/dev/null; systemctl disable autofs 2>/dev/null || true"
}

# ── U-42: 불필요 RPC 서비스 비활성화 [AUTO] ──────────────────────────────────
_u42_check() {
    local ok=true
    for svc in rpcbind rpcbind.socket; do
        local active; active=$(systemctl is-active "$svc" 2>/dev/null || echo "inactive")
        [[ "$active" == "active" ]] \
            && { _fail "[systemctl] ${svc}: active (비활성화 필요)"; ok=false; } \
            || _pass "[systemctl] ${svc}: 비활성/미설치"
    done
    $ok && return 0 || return 1
}
_u42_fix() {
    for svc in rpcbind rpcbind.socket; do
        [[ "$(systemctl is-active "$svc" 2>/dev/null || echo inactive)" == "active" ]] && {
            do_fix "[systemctl] ${svc} stop+disable" "systemctl stop '$svc' 2>/dev/null; systemctl disable '$svc' 2>/dev/null || true"
        }
    done
}

# ── U-43: NIS/NIS+ 비활성화 [AUTO] ───────────────────────────────────────────
_u43_check() {
    local ok=true
    for svc in ypserv ypbind yppasswdd; do
        local active; active=$(systemctl is-active "$svc" 2>/dev/null || echo "inactive")
        [[ "$active" == "active" ]] \
            && { _fail "[systemctl] ${svc}: active"; ok=false; } \
            || _pass "[systemctl] ${svc}: 비활성/미설치"
    done
    $ok && return 0 || return 1
}
_u43_fix() {
    for svc in ypserv ypbind yppasswdd; do
        [[ "$(systemctl is-active "$svc" 2>/dev/null || echo inactive)" == "active" ]] && {
            do_fix "[systemctl] ${svc} stop+disable" "systemctl stop '$svc' 2>/dev/null; systemctl disable '$svc' 2>/dev/null || true"
        }
    done
}

# ── U-44: tftp·talk 서비스 비활성화 [AUTO] ───────────────────────────────────
_u44_check() {
    local ok=true
    for svc in tftp tftp.socket talk ntalk; do
        local active; active=$(systemctl is-active "$svc" 2>/dev/null || echo "inactive")
        [[ "$active" == "active" ]] \
            && { _fail "[systemctl] ${svc}: active"; ok=false; } \
            || _pass "[systemctl] ${svc}: 비활성/미설치"
    done
    $ok && return 0 || return 1
}
_u44_fix() {
    for svc in tftp tftp.socket talk ntalk; do
        [[ "$(systemctl is-active "$svc" 2>/dev/null || echo inactive)" == "active" ]] && {
            do_fix "[systemctl] ${svc} stop+disable" "systemctl stop '$svc' 2>/dev/null; systemctl disable '$svc' 2>/dev/null || true"
        }
    done
}

# ── U-45: 메일 서비스 버전 점검 [N/A or MANUAL] ──────────────────────────────
_u45_check() {
    if systemctl is-active sendmail postfix &>/dev/null 2>&1; then
        _warn "[systemctl] 메일 서비스 활성 — 버전 점검 및 최신 패치 적용 수동 확인 필요"
        return 1
    fi
    _na "[sendmail/postfix] 메일 서비스 미사용 — 해당없음"
    return 0
}

# ── U-46: 일반사용자 메일 서비스 실행 방지 [N/A] ─────────────────────────────
# AL2023 기본 환경 — sendmail 미설치
_u46_na() { _na "AL2023 기본 환경에서 sendmail 미설치 — 해당없음"; }

# ── U-47: 스팸 메일 릴레이 제한 [N/A] ────────────────────────────────────────
_u47_na() { _na "AL2023 기본 환경에서 메일 서버 미사용 — 해당없음"; }

# ── U-48: expn·vrfy 명령어 제한 [N/A] ────────────────────────────────────────
_u48_na() { _na "AL2023 기본 환경에서 sendmail 미설치 — 해당없음"; }

# ── U-49: DNS 보안 버전 패치 [MANUAL] ────────────────────────────────────────
_u49_check() {
    if ! systemctl is-active named &>/dev/null; then
        _na "[named] BIND 서비스 미사용 — 해당없음"
        return 0
    fi
    local ver
    ver=$(named -v 2>/dev/null | awk '{print $2}')
    _info "[named] 버전: ${ver} — 최신 보안 패치 적용 여부 수동 확인 필요"
    return 1
}

# ── U-50: DNS Zone Transfer 설정 [MANUAL] ────────────────────────────────────
_u50_check() {
    if ! systemctl is-active named &>/dev/null; then
        _na "[named] BIND 서비스 미사용 — 해당없음"
        return 0
    fi
    local conf="/etc/named.conf"
    grep -q "allow-transfer" "$conf" 2>/dev/null \
        && _pass "[${conf}] allow-transfer 설정됨" \
        || { _fail "[${conf}] allow-transfer 미설정 (zone transfer 무제한 가능)"; return 1; }
}

# ── U-51: DNS 동적 업데이트 설정 금지 [MANUAL] ───────────────────────────────
_u51_check() {
    if ! systemctl is-active named &>/dev/null; then
        _na "[named] BIND 서비스 미사용 — 해당없음"
        return 0
    fi
    local conf="/etc/named.conf"
    grep -E "allow-update\s*\{\s*none" "$conf" 2>/dev/null \
        && _pass "[${conf}] allow-update none 설정됨" \
        || { _fail "[${conf}] allow-update 제한 미설정"; return 1; }
}

# ── U-52: Telnet 서비스 비활성화 [AUTO] ──────────────────────────────────────
_u52_check() {
    local ok=true
    for svc in telnet telnet.socket; do
        local active; active=$(systemctl is-active "$svc" 2>/dev/null || echo "inactive")
        [[ "$active" == "active" ]] \
            && { _fail "[systemctl] ${svc}: active (비활성화 필요)"; ok=false; } \
            || _pass "[systemctl] ${svc}: 비활성/미설치"
    done
    $ok && return 0 || return 1
}
_u52_fix() {
    for svc in telnet telnet.socket; do
        [[ "$(systemctl is-active "$svc" 2>/dev/null || echo inactive)" == "active" ]] && {
            do_fix "[systemctl] ${svc} stop+disable" "systemctl stop '$svc' 2>/dev/null; systemctl disable '$svc' 2>/dev/null || true"
        }
    done
}

# ── U-53: FTP 서비스 정보 노출 제한 [AUTO] ───────────────────────────────────
_u53_check() {
    if ! systemctl is-active vsftpd &>/dev/null; then
        _pass "[vsftpd] 비활성/미설치 — 해당없음"
        return 0
    fi
    local f="/etc/vsftpd/vsftpd.conf"
    local banner; banner=$(grep -E "^ftpd_banner" "$f" 2>/dev/null | awk -F= '{print $2}')
    [[ -n "$banner" ]] \
        && _pass "[${f}] ftpd_banner 설정됨" \
        || { _fail "[${f}] ftpd_banner 미설정 (기본 버전 정보 노출 위험)"; return 1; }
}
_u53_fix() {
    local f="/etc/vsftpd/vsftpd.conf"; backup_file "$f"
    grep -q "^ftpd_banner" "$f" \
        && do_fix "[${f}] ftpd_banner 수정" "sed -i 's/^ftpd_banner.*/ftpd_banner=FTP Service/' '$f'" \
        || do_fix "[${f}] ftpd_banner 추가" "echo 'ftpd_banner=FTP Service' >> '$f'"
    do_fix "vsftpd restart" "systemctl restart vsftpd 2>/dev/null || true"
}

# ── U-54: 암호화되지 않는 FTP 비활성화 [AUTO] ────────────────────────────────
_u54_check() {
    if ! systemctl is-active vsftpd &>/dev/null; then
        _pass "[vsftpd] 비활성/미설치"
        return 0
    fi
    local f="/etc/vsftpd/vsftpd.conf"
    local ssl
    ssl=$(grep -E "^ssl_enable" "$f" 2>/dev/null | awk -F= '{print $2}' | tr -d ' ')
    [[ "${ssl^^}" == "YES" ]] \
        && _pass "[${f}] ssl_enable=YES (FTPS 사용)" \
        || { _fail "[${f}] ssl_enable=${ssl:-미설정} (평문 FTP 사용 중)"; return 1; }
}
_u54_fix() {
    _manual "FTP 서비스가 필요한 경우 SFTP(SSH) 또는 FTPS(SSL) 로 전환, vsftpd ssl_enable=YES 수동 설정 필요"
}

# ── U-55: FTP 계정 Shell 제한 [AUTO] ─────────────────────────────────────────
_u55_check() {
    if ! systemctl is-active vsftpd &>/dev/null; then
        _pass "[vsftpd] 비활성/미설치 — 해당없음"
        return 0
    fi
    local ftp_shell
    ftp_shell=$(awk -F: '$1 == "ftp" {print $7}' /etc/passwd 2>/dev/null)
    [[ "$ftp_shell" == "/sbin/nologin" || "$ftp_shell" == "/bin/false" ]] \
        && { _pass "[/etc/passwd] ftp 계정 shell=${ftp_shell}"; return 0; } \
        || { _fail "[/etc/passwd] ftp 계정 shell=${ftp_shell:-없음} (nologin 필요)"; return 1; }
}
_u55_fix() {
    do_fix "[/etc/passwd] ftp shell → /sbin/nologin" "usermod -s /sbin/nologin ftp 2>/dev/null || true"
}

# ── U-56: FTP 서비스 접근 제어 설정 [MANUAL] ─────────────────────────────────
_u56_check() {
    if ! systemctl is-active vsftpd &>/dev/null; then
        _pass "[vsftpd] 비활성/미설치 — 해당없음"
        return 0
    fi
    local f="/etc/vsftpd/vsftpd.conf"
    local tcp_wrap; tcp_wrap=$(grep -E "^tcp_wrappers" "$f" 2>/dev/null | awk -F= '{print $2}' | tr -d ' ')
    [[ "${tcp_wrap^^}" == "YES" ]] \
        && _pass "[${f}] tcp_wrappers=YES" \
        || { _fail "[${f}] tcp_wrappers=${tcp_wrap:-미설정} — IP 접근 제어 설정 필요"; return 1; }
}

# ── U-57: Ftpusers 파일 설정 [AUTO] ──────────────────────────────────────────
_u57_check() {
    if ! systemctl is-active vsftpd &>/dev/null; then
        _pass "[vsftpd] 비활성/미설치 — 해당없음"
        return 0
    fi
    local f="/etc/vsftpd/ftpusers"
    [[ ! -f "$f" ]] && { _fail "[${f}] 파일 없음"; return 1; }
    local ok=true
    for user in root bin daemon adm lp sync shutdown halt mail news uucp operator; do
        grep -qx "$user" "$f" \
            && _pass "[${f}] ${user} 접근 차단됨" \
            || { _fail "[${f}] ${user} 미등록 (차단 필요)"; ok=false; }
    done
    $ok && return 0 || return 1
}
_u57_fix() {
    local f="/etc/vsftpd/ftpusers"; backup_file "$f"
    for user in root bin daemon adm lp sync shutdown halt mail news uucp operator; do
        grep -qx "$user" "$f" 2>/dev/null || do_fix "[${f}] ${user} 추가" "echo '$user' >> '$f'"
    done
}

# ── U-58: 불필요 SNMP 서비스 비활성화 [AUTO] ─────────────────────────────────
_u58_check() {
    local active; active=$(systemctl is-active snmpd 2>/dev/null || echo "inactive")
    [[ "$active" != "active" ]] \
        && { _pass "[systemctl] snmpd: 비활성/미설치"; return 0; } \
        || { _warn "[systemctl] snmpd: active — 필요 여부 수동 확인"; return 1; }
}
_u58_fix() {
    _manual "SNMP 서비스가 불필요한 경우: systemctl stop snmpd && systemctl disable snmpd"
}

# ── U-59: 안전한 SNMP 버전 사용 [MANUAL] ─────────────────────────────────────
_u59_check() {
    if ! systemctl is-active snmpd &>/dev/null; then
        _pass "[snmpd] 비활성/미설치 — 해당없음"
        return 0
    fi
    local ver
    ver=$(snmpd --version 2>&1 | head -1)
    _info "[snmpd] 버전: ${ver}"
    grep -rE "^rocommunity|^rwcommunity" /etc/snmp/snmpd.conf 2>/dev/null \
        && { _fail "[/etc/snmp/snmpd.conf] SNMPv1/v2c community string 사용 중 — SNMPv3으로 전환 필요"; return 1; } \
        || _pass "[/etc/snmp/snmpd.conf] SNMPv1/v2c community 미사용"
}

# ── U-60: SNMP Community String 복잡성 [MANUAL] ──────────────────────────────
_u60_check() {
    if ! systemctl is-active snmpd &>/dev/null; then
        _pass "[snmpd] 비활성/미설치 — 해당없음"
        return 0
    fi
    local f="/etc/snmp/snmpd.conf"
    local bad
    bad=$(grep -E "^rocommunity|^rwcommunity" "$f" 2>/dev/null | grep -E "public|private")
    [[ -z "$bad" ]] \
        && _pass "[${f}] 기본 community string(public/private) 미사용" \
        || { _fail "[${f}] 취약 community string 사용: ${bad}"; return 1; }
}

# ── U-61: SNMP Access Control 설정 [MANUAL] ──────────────────────────────────
_u61_check() {
    if ! systemctl is-active snmpd &>/dev/null; then
        _pass "[snmpd] 비활성/미설치 — 해당없음"
        return 0
    fi
    local f="/etc/snmp/snmpd.conf"
    grep -qE "^agentAddress\s+udp:127\|^com2sec.*localhost\|rocommunity.*127" "$f" 2>/dev/null \
        && _pass "[${f}] SNMP 접근 제어 설정됨" \
        || { _fail "[${f}] SNMP 접근 IP 제한 미설정 (전체 허용 가능)"; return 1; }
}

# ── U-62: 로그인 시 경고 메시지 설정 [AUTO] ───────────────────────────────────
_u62_check() {
    local ok=true
    for f in /etc/issue /etc/issue.net /etc/motd; do
        { [[ -f "$f" ]] && grep -q "Authorized\|Unauthorized\|허가" "$f" 2>/dev/null; } \
            && _pass "[${f}] 경고 배너 설정됨" \
            || { _fail "[${f}] 경고 배너 없음"; ok=false; }
    done
    local sc="/etc/ssh/sshd_config"
    grep -qiE "^Banner\s+" "$sc" \
        && _pass "[${sc}] Banner 설정됨" \
        || { _fail "[${sc}] Banner 미설정"; ok=false; }
    $ok && return 0 || return 1
}
_u62_fix() {
    local banner
    banner='*******************************************************************************
* This system is managed by Telepix Co., Ltd.                                 *
* Authorized Personnel Only.                                                   *
* Unauthorized access is strictly prohibited and may result in legal penalty.  *
*******************************************************************************'
    for f in /etc/issue /etc/issue.net /etc/motd; do
        do_fix "[${f}] 경고 배너 작성" "printf '%s\n' '$banner' > '$f'"
    done
    local sc="/etc/ssh/sshd_config"; backup_file "$sc"
    grep -q "^Banner" "$sc" \
        && do_fix "[${sc}] Banner 경로 수정" "sed -i 's|^Banner.*|Banner /etc/issue.net|' '$sc'" \
        || do_fix "[${sc}] Banner 추가"      "echo 'Banner /etc/issue.net' >> '$sc'"
    $DRY_RUN || do_fix "sshd reload" "systemctl reload sshd"
}

# ── U-63: sudo 명령어 접근 관리 [MANUAL] ─────────────────────────────────────
_u63_check() {
    local ok=true
    local risky
    risky=$(grep -rE "NOPASSWD" /etc/sudoers /etc/sudoers.d/ 2>/dev/null \
        | grep -v ":#")
    [[ -z "$risky" ]] \
        && _pass "[/etc/sudoers] NOPASSWD 설정 없음" \
        || { _warn "[/etc/sudoers] NOPASSWD 설정 발견:"; echo "$risky" | while read -r line; do _warn "  ${line}"; done; ok=false; }
    $ok && return 0 || return 1
}

# =============================================================================
# 4. 패치 관리 (U-64)
# =============================================================================

# ── U-64: 주기적 보안 패치 적용 [INFO] ───────────────────────────────────────
_u64_check() {
    _info "[dnf] 보안 업데이트 현황 확인 중..."
    local updates
    updates=$(dnf check-update --security 2>/dev/null \
        | grep -vE "^$|^Last|^Loaded|^Loading|^Amazon|repository|kB/s|MB/s" \
        | head -20)
    if [[ -z "$updates" ]]; then
        _pass "[dnf] 미적용 보안 패치 없음"
    else
        _warn "[dnf] 미적용 보안 패치 존재:"
        echo "$updates" | while read -r line; do _warn "  ${line}"; done
        _warn "수동 적용: dnf update --security -y"
    fi
}

# =============================================================================
# 5. 로그 관리 (U-65 ~ U-67)
# =============================================================================

# ── U-65: NTP 및 시각 동기화 설정 [AUTO] ─────────────────────────────────────
_u65_check() {
    local ok=true
    # chronyd (AL2023 기본 NTP)
    if systemctl is-active chronyd &>/dev/null; then
        _pass "[systemctl] chronyd 활성화됨"
        local sync
        sync=$(chronyc tracking 2>/dev/null | grep "Reference ID")
        _info "[chronyc] ${sync:-동기화 정보 없음}"
    elif systemctl is-active ntpd &>/dev/null; then
        _pass "[systemctl] ntpd 활성화됨"
    else
        _fail "[systemctl] NTP 서비스(chronyd/ntpd) 비활성화"; ok=false
    fi
    $ok && return 0 || return 1
}
_u65_fix() {
    systemctl is-active chronyd &>/dev/null || \
        do_fix "[systemctl] chronyd 활성화" "systemctl enable --now chronyd"
}

# ── U-66: 정책에 따른 시스템 로깅 설정 [AUTO] ────────────────────────────────
_u66_check() {
    local ok=true

    # AL2023: journald 기본, rsyslog 선택적
    if systemctl is-active rsyslog &>/dev/null; then
        _pass "[systemctl] rsyslog 활성화"
        local conf="/etc/rsyslog.conf"
        grep -qE "authpriv\.\*" "$conf" 2>/dev/null \
            && _pass "[${conf}] authpriv 로그 설정됨" \
            || { _fail "[${conf}] authpriv 미설정"; ok=false; }
        for lf in /var/log/messages /var/log/secure /var/log/cron; do
            [[ -f "$lf" ]] \
                && _pass "[${lf}] 파일 존재" \
                || { _fail "[${lf}] 파일 없음"; ok=false; }
        done
    elif systemctl is-active systemd-journald &>/dev/null; then
        _pass "[systemctl] systemd-journald 활성화 (rsyslog 대체)"
        _info "[journald] journalctl -u sshd 등으로 로그 확인 가능"
    else
        _fail "[systemctl] 로그 데몬(rsyslog/journald) 모두 비활성화"
        ok=false
    fi

    # auditd
    systemctl is-active auditd &>/dev/null \
        && _pass "[systemctl] auditd 활성화" \
        || { _fail "[systemctl] auditd 비활성화"; ok=false; }

    $ok && return 0 || return 1
}

_u66_fix() {
    local conf="/etc/rsyslog.conf"

    # rsyslog가 없으면 설치 (journald만으로는 KISA 기준 미충족 가능)
    if ! systemctl is-active rsyslog &>/dev/null; then
        do_fix "rsyslog 설치+활성화" "dnf install -y rsyslog && systemctl enable --now rsyslog"
    fi

    if ! grep -qE "authpriv\.\*" "$conf" 2>/dev/null; then
        backup_file "$conf"
        do_fix "[${conf}] 로그 정책 추가" "cat >> '$conf' << 'EOF'

# KISA U-66
*.info;mail.none;authpriv.none;cron.none  /var/log/messages
authpriv.*                                 /var/log/secure
cron.*                                     /var/log/cron
EOF
systemctl restart rsyslog"
    fi

    # auditd
    command -v auditd &>/dev/null || do_fix "auditd 설치" "dnf install -y audit"
    systemctl is-active auditd &>/dev/null || \
        do_fix "auditd 활성화" "systemctl enable --now auditd"
    local rules="/etc/audit/rules.d/kisa.rules"
    [[ -f "$rules" ]] || do_fix "[${rules}] KISA 감사규칙 추가" "cat > '$rules' << 'EOF'
-w /etc/passwd  -p wa -k identity
-w /etc/shadow  -p wa -k identity
-w /etc/group   -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-a always,exit -F arch=b64 -S execve -k exec_commands
EOF
augenrules --load 2>/dev/null || true"
}

# ── U-67: 로그 디렉토리 소유자·권한 설정 [AUTO] ──────────────────────────────
_u67_check() {
    local ok=true
    # /var/log 디렉토리
    local p o
    p=$(stat -c '%a' /var/log 2>/dev/null); o=$(stat -c '%U' /var/log 2>/dev/null)
    [[ "$o" == "root" ]] && _pass "[/var/log] 소유자=root" || { _fail "[/var/log] 소유자=${o}"; ok=false; }
    [[ "$p" -le 755 ]]   && _pass "[/var/log] 권한=${p}"   || { _fail "[/var/log] 권한=${p} (≤755 필요)"; ok=false; }
    # 주요 로그파일 권한
    for lf in /var/log/messages /var/log/secure /var/log/cron /var/log/wtmp /var/log/btmp; do
        [[ ! -f "$lf" ]] && continue
        local lp lo; lp=$(stat -c '%a' "$lf"); lo=$(stat -c '%U' "$lf")
        [[ "$lo" == "root" ]] && _pass "[${lf}] 소유자=root" || { _fail "[${lf}] 소유자=${lo}"; ok=false; }
        [[ "$lp" -le 640 ]]   && _pass "[${lf}] 권한=${lp}"  || { _fail "[${lf}] 권한=${lp} (≤640 필요)"; ok=false; }
    done
    $ok && return 0 || return 1
}
_u67_fix() {
    do_fix "[/var/log] chown root, chmod 755" "chown root:root /var/log && chmod 755 /var/log"
    for lf in /var/log/messages /var/log/secure /var/log/cron /var/log/wtmp /var/log/btmp; do
        [[ ! -f "$lf" ]] && continue
        do_fix "[${lf}] chown root:root" "chown root:root '$lf'"
        do_fix "[${lf}] chmod 640"       "chmod 640 '$lf'"
    done
}

# =============================================================================
# 결과 요약
# =============================================================================
print_summary() {
    local total=$((CNT_PASS + CNT_FIXED + CNT_FAILED + CNT_MANUAL + CNT_NA + CNT_SKIP))
    echo ""
    print_header "점검/조치/검증 결과 요약"
    echo ""
    printf "  %-24s : %s\n" "호스트명"          "$HOSTNAME_SHORT"
    printf "  %-24s : %s\n" "점검 일시"          "$(date)"
    printf "  %-24s : %s\n" "KISA 가이드 기준"   "2026 (U-01 ~ U-67)"
    printf "  %-24s : %s\n" "총 실행 항목"       "$total"
    echo ""
    echo -e "  ${GREEN}✔ PASS     (처음부터 양호)${NC}        : ${BOLD}${CNT_PASS}${NC}"
    echo -e "  ${GREEN}✔ FIXED    (자동 조치+검증 완료)${NC}  : ${BOLD}${CNT_FIXED}${NC}"
    echo -e "  ${RED}✘ FAILED   (조치 후에도 미흡)${NC}     : ${BOLD}${CNT_FAILED}${NC}"
    echo -e "  ${YELLOW}⚙ MANUAL   (수동 조치 필요)${NC}       : ${BOLD}${CNT_MANUAL}${NC}"
    echo -e "  ${DIM}– N/A      (해당없음)${NC}              : ${BOLD}${CNT_NA}${NC}"
    echo -e "  ${YELLOW}⊘ SKIP${NC}                              : ${BOLD}${CNT_SKIP}${NC}"
    echo ""
    echo -e "  로그 파일 : ${LOG_FILE}"
    echo -e "  백업 경로 : ${BACKUP_DIR}"
    echo ""
    if [[ $CNT_FAILED -eq 0 && $CNT_MANUAL -eq 0 ]]; then
        echo -e "  ${GREEN}${BOLD}✔ 전체 자동 조치 완료${NC}"
    else
        [[ $CNT_FAILED -gt 0 ]]  && echo -e "  ${RED}${BOLD}✘ ${CNT_FAILED}개 항목 추가 확인 필요${NC}  → grep VERIFY_FAIL ${LOG_FILE}"
        [[ $CNT_MANUAL -gt 0 ]] && echo -e "  ${YELLOW}${BOLD}⚙ ${CNT_MANUAL}개 항목 수동 조치 필요${NC}    → grep MANUAL ${LOG_FILE}"
    fi
    echo ""
    log "SUMMARY" "HOST=${HOSTNAME_SHORT} PASS=${CNT_PASS} FIXED=${CNT_FIXED} FAILED=${CNT_FAILED} MANUAL=${CNT_MANUAL} NA=${CNT_NA} SKIP=${CNT_SKIP}"
}

# =============================================================================
# 인자 파싱
# =============================================================================
parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            --dry-run)      DRY_RUN=true ;;
            --check-only)   CHECK_ONLY=true ;;
            --fix-only)     FIX_ONLY=true ;;
            --no-verify)    NO_VERIFY=true ;;
            --item)         SELECTED_ITEMS="$2"; shift ;;
            --item=*)       SELECTED_ITEMS="${1#*=}" ;;
            --category)     SELECTED_CATEGORY="$2"; shift ;;
            --category=*)   SELECTED_CATEGORY="${1#*=}" ;;
            --help|-h)
                cat << 'EOF'
사용법: al2023_kisa_hardening.sh [옵션]

옵션:
  --check-only        점검만 수행 (조치 없음)
  --dry-run           조치 내용 출력만 (실제 변경 없음)
  --fix-only          check 생략, 전 항목 조치 실행
  --no-verify         조치 후 검증 단계 생략
  --item U-01,U-03    특정 항목만 실행 (쉼표 구분)
  --category 1        특정 대분류만 실행
                        1=계정관리(U-01~U-13)
                        2=파일디렉토리(U-14~U-33)
                        3=서비스관리(U-34~U-63)
                        4=패치관리(U-64)
                        5=로그관리(U-65~U-67)

항목 처리 유형:
  [AUTO]   자동 점검 + 자동 조치
  [MANUAL] 자동 점검 + 수동 조치 필요
  [INFO]   점검 결과만 출력
  [N/A]    해당 환경에서 서비스 미존재
EOF
                exit 0 ;;
            *) echo "알 수 없는 옵션: $1"; exit 1 ;;
        esac
        shift
    done
}

# =============================================================================
# 초기화
# =============================================================================
init() {
    [[ $EUID -ne 0 ]] && { echo -e "${RED}ERROR: root 권한 필요 (sudo 실행)${NC}"; exit 1; }
    mkdir -p "$LOG_DIR" "$BACKUP_DIR"
    grep -qi "Amazon Linux" /etc/os-release 2>/dev/null || _warn "Amazon Linux가 아닐 수 있음"

    echo -e "${BOLD}${CYAN}"
    cat << 'BANNER'
╔══════════════════════════════════════════════════════════════╗
║  KISA 주요정보통신기반시설 기술적 취약점 분석·평가            ║
║  Amazon Linux 2023 보안조치 자동화 스크립트 v3.0.0           ║
║  기준: 2026 상세가이드 U-01 ~ U-67 (67개 항목)               ║
╚══════════════════════════════════════════════════════════════╝
BANNER
    echo -e "${NC}"
    echo -e "  호스트명  : ${HOSTNAME_SHORT}"
    echo -e "  실행 시각 : $(date)"
    echo -e "  로그 파일 : ${LOG_FILE}"
    echo -e "  백업 경로 : ${BACKUP_DIR}"
    $DRY_RUN         && echo -e "  모드      : ${MAGENTA}DRY-RUN${NC}"
    $CHECK_ONLY      && echo -e "  모드      : ${YELLOW}CHECK-ONLY${NC}"
    $FIX_ONLY        && echo -e "  모드      : ${YELLOW}FIX-ONLY${NC}"
    $NO_VERIFY       && echo -e "  검증      : ${YELLOW}SKIP${NC}"
    [[ -n "$SELECTED_ITEMS" ]]    && echo -e "  대상 항목 : ${CYAN}${SELECTED_ITEMS}${NC}"
    [[ -n "$SELECTED_CATEGORY" ]] && echo -e "  대분류    : ${CYAN}${SELECTED_CATEGORY}${NC}"

    log "INFO" "Start v3.0.0 host=${HOSTNAME_SHORT} dry=${DRY_RUN} check=${CHECK_ONLY} fix_only=${FIX_ONLY} no_verify=${NO_VERIFY} items='${SELECTED_ITEMS}' category='${SELECTED_CATEGORY}'"
}

# =============================================================================
# 메인 실행
# =============================================================================
main() {
    parse_args "$@"
    init

    # ── 1. 계정 관리 ──────────────────────────────────────────────────────────
    should_run "U-01" "1" && run_auto   "U-01" "root 계정 원격 접속 제한"          "상" _u01_check _u01_fix
    should_run "U-02" "1" && run_auto   "U-02" "비밀번호 관리정책 설정"             "상" _u02_check _u02_fix
    should_run "U-03" "1" && run_auto   "U-03" "계정 잠금 임계값 설정"             "상" _u03_check _u03_fix
    should_run "U-04" "1" && run_auto   "U-04" "비밀번호 파일 보호"                "상" _u04_check _u04_fix
    should_run "U-05" "1" && run_auto   "U-05" "root 이외 UID=0 금지"              "상" _u05_check _u05_fix
    should_run "U-06" "1" && run_auto   "U-06" "사용자 계정 su 기능 제한"          "상" _u06_check _u06_fix
    should_run "U-07" "1" && run_manual "U-07" "불필요한 계정 제거"                "하" _u07_check \
        "목록 확인 후 userdel <계정명> 으로 수동 제거 (서비스 계정 여부 확인 필수)"
    should_run "U-08" "1" && run_manual "U-08" "관리자 그룹에 최소한의 계정 포함"  "중" _u08_check \
        "wheel/sudo 그룹 구성원을 최소화하고 필요 계정만 유지"
    should_run "U-09" "1" && run_manual "U-09" "계정 없는 GID 금지"               "하" _u09_check \
        "orphan GID 확인 후 groupdel <그룹명> 으로 수동 제거"
    should_run "U-10" "1" && run_manual "U-10" "동일 UID 금지"                    "중" _u10_check \
        "usermod -u <새UID> <계정명> 으로 중복 UID 수동 변경"
    should_run "U-11" "1" && run_manual "U-11" "사용자 Shell 점검" "하" _u11_check \
        "목록 확인 후 서비스 계정 여부 검토하여 usermod -s /sbin/nologin <계정명> 수동 적용"
    should_run "U-12" "1" && run_auto   "U-12" "세션 종료 시간 설정"              "하" _u12_check _u12_fix
    should_run "U-13" "1" && run_auto   "U-13" "안전한 비밀번호 암호화 알고리즘"  "중" _u13_check _u13_fix

    # ── 2. 파일 및 디렉토리 관리 ──────────────────────────────────────────────
    should_run "U-14" "2" && run_auto   "U-14" "root 홈·PATH 권한 및 경로 설정"   "상" _u14_check _u14_fix
    should_run "U-15" "2" && run_manual "U-15" "파일·디렉토리 소유자 설정"        "상" _u15_check \
        "소유자 없는 파일: chown <적절한계정> <파일경로> 로 수동 설정"
    should_run "U-16" "2" && run_auto   "U-16" "/etc/passwd 소유자·권한 설정"     "상" _u16_check _u16_fix
    should_run "U-17" "2" && run_auto   "U-17" "시스템 시작 스크립트 권한 설정"   "상" _u17_check _u17_fix
    should_run "U-18" "2" && run_auto   "U-18" "/etc/shadow 소유자·권한 설정"     "상" _u18_check _u18_fix
    should_run "U-19" "2" && run_auto   "U-19" "/etc/hosts 소유자·권한 설정"      "상" _u19_check _u19_fix
    should_run "U-20" "2" && run_auto   "U-20" "/etc/(x)inetd.conf 권한 설정"    "상" _u20_check _u20_fix
    should_run "U-21" "2" && run_auto   "U-21" "/etc/(r)syslog.conf 권한 설정"   "상" _u21_check _u21_fix
    should_run "U-22" "2" && run_auto   "U-22" "/etc/services 권한 설정"         "상" _u22_check _u22_fix
    should_run "U-23" "2" && run_manual "U-23" "SUID·SGID 파일 점검"             "상" _u23_check \
        "불필요한 SUID/SGID 파일: chmod -s <파일경로> 로 수동 제거"
    should_run "U-24" "2" && run_auto   "U-24" "시스템 환경변수 파일 권한 설정"   "상" _u24_check _u24_fix
    should_run "U-25" "2" && run_manual "U-25" "world writable 파일 점검"         "상" _u25_check \
        "불필요한 파일: chmod o-w <파일경로> / 필요한 경우 sticky bit 설정"
    should_run "U-26" "2" && run_info   "U-26" "/dev 비정상 파일 점검"            "상" _u26_check
    should_run "U-27" "2" && run_auto   "U-27" ".rhosts·hosts.equiv 사용 금지"   "상" _u27_check _u27_fix
    should_run "U-28" "2" && run_manual "U-28" "접속 IP 및 포트 제한"             "상" _u28_check \
        "firewall-cmd --add-rich-rule 또는 /etc/hosts.allow|deny 수동 설정"
    should_run "U-29" "2" && run_auto   "U-29" "hosts.lpd 파일 권한 설정"         "하" _u29_check _u29_fix
    should_run "U-30" "2" && run_auto   "U-30" "UMASK 설정 관리"                  "중" _u30_check _u30_fix
    should_run "U-31" "2" && run_auto   "U-31" "홈 디렉토리 소유자·권한 설정"     "중" _u31_check _u31_fix
    should_run "U-32" "2" && run_info   "U-32" "홈 디렉토리 존재 관리"            "중" _u32_check
    should_run "U-33" "2" && run_manual "U-33" "숨겨진 파일·디렉토리 검색 및 제거" "하" _u33_check \
        "목록 확인 후 불필요한 숨김 파일: rm -f <파일경로> 로 수동 제거 (정상 파일 삭제 주의)"

    # ── 3. 서비스 관리 ────────────────────────────────────────────────────────
    should_run "U-34" "3" && run_auto   "U-34" "Finger 서비스 비활성화"           "상" _u34_check _u34_fix
    should_run "U-35" "3" && run_auto   "U-35" "공유 서비스 익명 접근 제한"       "상" _u35_check _u35_fix
    should_run "U-36" "3" && run_auto   "U-36" "r 계열 서비스 비활성화"           "상" _u36_check _u36_fix
    should_run "U-37" "3" && run_auto   "U-37" "crontab 파일 권한 설정"           "상" _u37_check _u37_fix
    should_run "U-38" "3" && run_auto   "U-38" "DoS 취약 서비스 비활성화"         "상" _u38_check _u38_fix
    should_run "U-39" "3" && run_auto   "U-39" "불필요한 NFS 서비스 비활성화"     "상" _u39_check _u39_fix
    should_run "U-40" "3" && run_manual "U-40" "NFS 접근 통제"                    "상" _u40_check \
        "/etc/exports 수정: 와일드카드(*) 제거, no_root_squash 제거, 특정 IP만 허용"
    should_run "U-41" "3" && run_auto   "U-41" "불필요한 automountd 제거"         "상" _u41_check _u41_fix
    should_run "U-42" "3" && run_auto   "U-42" "불필요한 RPC 서비스 비활성화"     "상" _u42_check _u42_fix
    should_run "U-43" "3" && run_auto   "U-43" "NIS/NIS+ 비활성화"               "상" _u43_check _u43_fix
    should_run "U-44" "3" && run_auto   "U-44" "tftp·talk 서비스 비활성화"        "상" _u44_check _u44_fix
    should_run "U-45" "3" && run_manual "U-45" "메일 서비스 버전 점검"            "상" _u45_check \
        "sendmail/postfix 최신 보안 패치 적용: dnf update sendmail postfix"
    should_run "U-46" "3" && run_na     "U-46" "일반사용자 메일 서비스 실행 방지" "상" "AL2023 기본 환경 sendmail 미설치"
    should_run "U-47" "3" && run_na     "U-47" "스팸 메일 릴레이 제한"            "상" "AL2023 기본 환경 메일 서버 미사용"
    should_run "U-48" "3" && run_na     "U-48" "expn·vrfy 명령어 제한"            "중" "AL2023 기본 환경 sendmail 미설치"
    should_run "U-49" "3" && run_manual "U-49" "DNS 보안 버전 패치"               "상" _u49_check \
        "BIND 사용 시 최신 버전 적용: dnf update bind"
    should_run "U-50" "3" && run_manual "U-50" "DNS Zone Transfer 설정"           "상" _u50_check \
        "/etc/named.conf: allow-transfer { 허용IP; }; 수동 설정"
    should_run "U-51" "3" && run_manual "U-51" "DNS 동적 업데이트 설정 금지"      "중" _u51_check \
        "/etc/named.conf: allow-update { none; }; 수동 설정"
    should_run "U-52" "3" && run_auto   "U-52" "Telnet 서비스 비활성화"           "중" _u52_check _u52_fix
    should_run "U-53" "3" && run_auto   "U-53" "FTP 서비스 정보 노출 제한"        "하" _u53_check _u53_fix
    should_run "U-54" "3" && run_manual "U-54" "암호화되지 않은 FTP 비활성화"     "중" _u54_check \
        "SFTP(SSH) 또는 FTPS 전환: vsftpd ssl_enable=YES 및 인증서 설정"
    should_run "U-55" "3" && run_auto   "U-55" "FTP 계정 Shell 제한"              "중" _u55_check _u55_fix
    should_run "U-56" "3" && run_manual "U-56" "FTP 서비스 접근 제어 설정"        "하" _u56_check \
        "vsftpd.conf: tcp_wrappers=YES 및 /etc/hosts.allow|deny에 FTP 접근 IP 제한"
    should_run "U-57" "3" && run_auto   "U-57" "Ftpusers 파일 설정"               "중" _u57_check _u57_fix
    should_run "U-58" "3" && run_manual "U-58" "불필요한 SNMP 서비스 비활성화"    "중" _u58_check \
        "SNMP 불필요 시: systemctl stop snmpd && systemctl disable snmpd"
    should_run "U-59" "3" && run_manual "U-59" "안전한 SNMP 버전 사용"            "상" _u59_check \
        "/etc/snmp/snmpd.conf: SNMPv3 사용자 기반 인증으로 전환"
    should_run "U-60" "3" && run_manual "U-60" "SNMP Community String 복잡성"     "중" _u60_check \
        "/etc/snmp/snmpd.conf: public/private 대신 복잡한 문자열로 수동 변경"
    should_run "U-61" "3" && run_manual "U-61" "SNMP Access Control 설정"         "상" _u61_check \
        "/etc/snmp/snmpd.conf: 특정 관리 호스트 IP만 허용 수동 설정"
    should_run "U-62" "3" && run_auto   "U-62" "로그인 시 경고 메시지 설정"       "하" _u62_check _u62_fix
    should_run "U-63" "3" && run_manual "U-63" "sudo 명령어 접근 관리"            "중" _u63_check \
        "visudo로 NOPASSWD 제거 및 필요 최소 명령만 허용"

    # ── 4. 패치 관리 ──────────────────────────────────────────────────────────
    should_run "U-64" "4" && run_info   "U-64" "주기적 보안 패치 적용"            "상" _u64_check

    # ── 5. 로그 관리 ──────────────────────────────────────────────────────────
    should_run "U-65" "5" && run_auto   "U-65" "NTP 및 시각 동기화 설정"          "중" _u65_check _u65_fix
    should_run "U-66" "5" && run_auto   "U-66" "정책에 따른 시스템 로깅 설정"     "중" _u66_check _u66_fix
    should_run "U-67" "5" && run_auto   "U-67" "로그 디렉토리 소유자·권한 설정"   "중" _u67_check _u67_fix

    print_summary
}

main "$@"
```