#!/bin/bash
# jm - Simplified JAR management tool with stable table alignment

# Color definitions
GREEN="\033[0;32m"
RED="\033[0;31m"
YELLOW="\033[1;33m"
BLUE="\033[0;34m"
CYAN="\033[0;36m"
NC="\033[0m" # No Color

# Table configuration
COLUMN_SEP="│"
HEADER_LINE="─"
CORNER="┼"
REFRESH_INTERVAL=1
PADDING=2
MIN_COL_WIDTH=6

# Initialize variables
SCRIPT_NAME="jm"
PID_DIR="/var/run/jm"
LOG_DIR="/var/log/jm"
DEFAULT_JDK=""
declare -A PID_MAP  # 使用关联数组来存储PID，以应用路径为键
declare -a LAST_PIDS
STATUS_COL_WIDTHS=(0 0 0 0 0 0 0)
JDK_COL_WIDTHS=(0 0 0)
FIRST_DYNAMIC_RUN=true
SORTED_DATA=()
SORT_FIELD="status"
SORT_ORDER="desc"
LAST_DISPLAYED_LINES=0

# Get script directory and config files
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CURRENT_DIR="$(pwd)"
CONF_FILES=("$CURRENT_DIR/jm.conf" "$SCRIPT_DIR/jm.conf" "/etc/jm/jm.conf")
JAR_CONFIGS=()

# Create necessary directories
mkdir -p "$PID_DIR" "$LOG_DIR" 2>/dev/null || { 
    echo -e "${YELLOW}Warning: Failed to create system directories, using local directories${NC}"
    PID_DIR="./jm_pids"
    LOG_DIR="./jm_logs"
    mkdir -p "$PID_DIR" "$LOG_DIR" || { echo -e "${RED}Error: Failed to create local directories $PID_DIR or $LOG_DIR${NC}"; exit 1; }
}

# Trap signals
cleanup_exit() {
    echo -e "\033[?25h${NC}"
    exit 0
}

trap cleanup_exit SIGINT SIGTERM

# Install script to /usr/bin
install_script() {
    if [ "$(id -u)" -ne 0 ]; then
        echo -e "${RED}Error: Installation requires root privileges. Use sudo.${NC}"
        return 1
    fi

    local script_path=$(realpath "$0" 2>/dev/null)
    [ -z "$script_path" ] && script_path="$0"
    
    if [ ! -f "$script_path" ]; then
        echo -e "${RED}Error: Script file not found at $script_path${NC}"
        return 1
    fi

    mkdir -p /etc/jm || { echo -e "${RED}Error: Failed to create /etc/jm directory${NC}"; return 1; }

    if [ ! -f "/etc/jm/jm.conf" ]; then
        cat > /etc/jm/jm.conf << EOF
# JM Configuration File
# Format:
# [application-name]
# path=/absolute/path/to/application.jar  (required)
# jdk=/absolute/path/to/jdk              (optional)
# args=JVM arguments                      (optional)
# daemon=true|false                       (optional, default: false)
# startupTimeout=30                       (optional, default: 60)

# Example:
#[myapp]
#path=/opt/myapp/myapp.jar
#args=-Xms512m -Xmx1g
#jdk=/usr/lib/jvm/jdk-17.0.10
EOF
        echo -e "${GREEN}Created default configuration: /etc/jm/jm.conf${NC}"
    fi

    cp "$script_path" "/usr/bin/$SCRIPT_NAME" || { echo -e "${RED}Error: Failed to copy script to /usr/bin${NC}"; return 1; }
    chmod 755 "/usr/bin/$SCRIPT_NAME" || { echo -e "${RED}Error: Failed to set permissions${NC}"; return 1; }

    echo -e "${GREEN}Successfully installed $SCRIPT_NAME to /usr/bin${NC}"
    echo -e "${CYAN}Usage: $SCRIPT_NAME [command]${NC}"
    return 0
}

# Uninstall script
uninstall_script() {
    if [ "$(id -u)" -ne 0 ]; then
        echo -e "${RED}Error: Uninstallation requires root privileges. Use sudo.${NC}"
        return 1
    fi

    local script_path="/usr/bin/$SCRIPT_NAME"
    
    if [ -f "$script_path" ]; then
        rm -f "$script_path"
        echo -e "${GREEN}Removed $script_path${NC}"
    else
        echo -e "${YELLOW}$SCRIPT_NAME is not installed${NC}"
    fi

    echo -e "${CYAN}Note: Configuration files in /etc/jm/ were not removed${NC}"
    return 0
}

# JDK information functions
get_jdk_install_path() {
    local java_path="$1"
    local real_path=$(readlink -f "$java_path" 2>/dev/null)
    [ -z "$real_path" ] && real_path="$java_path"
    echo "$real_path" | sed -E 's/\/bin\/java$//'
}

get_jdk_version() {
    local java_path="$1"
    if [ -x "$java_path" ]; then
        local version_output=$("$java_path" -version 2>&1)
        echo "$version_output" | awk -F '"' '/version/ {print $2}' | head -1
    else
        echo "Not executable"
    fi
}

calculate_jdk_column_widths() {
    JDK_COL_WIDTHS=(0 0 0)
    local headers=("App Name" "JDK Path" "JDK Version")
    
    for i in "${!headers[@]}"; do
        header_len=${#headers[$i]}
        JDK_COL_WIDTHS[$i]=$(( header_len > MIN_COL_WIDTH ? header_len : MIN_COL_WIDTH ))
    done
    
    for config in "${JAR_CONFIGS[@]}"; do
        IFS='|' read -r name _ jdk_home _ _ _ _ <<< "$config"
        local jdk_path="Using default JDK"
        local version="Unknown"
        
        if [ -n "$jdk_home" ] && [ -x "$jdk_home/bin/java" ]; then
            jdk_path=$(get_jdk_install_path "$jdk_home/bin/java")
            version=$(get_jdk_version "$jdk_home/bin/java")
        elif [ -n "$DEFAULT_JDK" ]; then
            jdk_path=$(get_jdk_install_path "$DEFAULT_JDK")
            version=$(get_jdk_version "$DEFAULT_JDK")
        fi
        
        local lengths=(${#name} ${#jdk_path} ${#version})
        for i in "${!lengths[@]}"; do
            if [ ${lengths[$i]} -gt ${JDK_COL_WIDTHS[$i]} ]; then
                JDK_COL_WIDTHS[$i]=${lengths[$i]}
            fi
        done
    done
    
    for i in "${!JDK_COL_WIDTHS[@]}"; do
        JDK_COL_WIDTHS[$i]=$((JDK_COL_WIDTHS[$i] + PADDING))
    done
}

create_jdk_table_line() {
    local cols=("$@")
    local line="$COLUMN_SEP"
    
    for i in "${!cols[@]}"; do
        local width=${JDK_COL_WIDTHS[$i]}
        local content="${cols[$i]}"
        
        if [ ${#content} -gt $((width - PADDING)) ]; then
            content="${content:0:$((width - PADDING - 3))}..."
        fi
        
        printf -v padded "%-${width}s" "$content"
        line+="$padded$COLUMN_SEP"
    done
    
    echo "$line"
}

create_jdk_header_sep() {
    local sep="$CORNER"
    
    for width in "${JDK_COL_WIDTHS[@]}"; do
        sep+=$(printf "%0.s$HEADER_LINE" $(seq 1 $width))
        sep+="$CORNER"
    done
    
    echo "$sep"
}

show_jdk_info() {
    local app_name="$1"
    
    if [ ${#JAR_CONFIGS[@]} -eq 0 ]; then
        echo -e "${YELLOW}No applications configured${NC}"
        return 0
    fi
    
    if [ -n "$app_name" ]; then
        local found=0
        for config in "${JAR_CONFIGS[@]}"; do
            IFS='|' read -r name path jdk_home _ _ _ _ <<< "$config"
            if [ "$name" = "$app_name" ]; then
                found=1
                echo -e "${BLUE}=== JDK Information for '$name' ===${NC}"
                calculate_jdk_column_widths
                
                echo -e "${BLUE}$(create_jdk_table_line "App Name" "JDK Path" "JDK Version")${NC}"
                echo -e "${BLUE}$(create_jdk_header_sep)${NC}"
                
                local jdk_path="Not configured"
                local version="Unknown"
                local color=$YELLOW
                
                if [ -n "$jdk_home" ] && [ -x "$jdk_home/bin/java" ]; then
                    jdk_path=$(get_jdk_install_path "$jdk_home/bin/java")
                    version=$(get_jdk_version "$jdk_home/bin/java")
                    color=$GREEN
                elif [ -n "$DEFAULT_JDK" ]; then
                    jdk_path="$(get_jdk_install_path "$DEFAULT_JDK") (default)"
                    version=$(get_jdk_version "$DEFAULT_JDK")
                    color=$CYAN
                fi
                
                local line=$(create_jdk_table_line "$name" "$jdk_path" "$version")
                echo -e "$color$line$NC"
                break
            fi
        done
        
        if [ $found -eq 0 ]; then
            echo -e "${RED}Error: Application '$app_name' not found${NC}"
            echo -e "${CYAN}Available applications:${NC}"
            for config in "${JAR_CONFIGS[@]}"; do
                IFS='|' read -r name _ <<< "$config"
                echo "  - $name"
            done
        fi
        return 0
    fi
    
    echo -e "${BLUE}=== JDK Information for All Applications ===${NC}"
    calculate_jdk_column_widths
    
    echo -e "${BLUE}$(create_jdk_table_line "App Name" "JDK Path" "JDK Version")${NC}"
    echo -e "${BLUE}$(create_jdk_header_sep)${NC}"
    
    for config in "${JAR_CONFIGS[@]}"; do
        IFS='|' read -r name path jdk_home _ _ _ _ <<< "$config"
        local jdk_path="Not configured"
        local version="Unknown"
        local color=$YELLOW
        
        if [ -n "$jdk_home" ] && [ -x "$jdk_home/bin/java" ]; then
            jdk_path=$(get_jdk_install_path "$jdk_home/bin/java")
            version=$(get_jdk_version "$jdk_home/bin/java")
            color=$GREEN
        elif [ -n "$DEFAULT_JDK" ]; then
            jdk_path="$(get_jdk_install_path "$DEFAULT_JDK") (default)"
            version=$(get_jdk_version "$DEFAULT_JDK")
            color=$CYAN
        fi
        
        local line=$(create_jdk_table_line "$name" "$jdk_path" "$version")
        echo -e "$color$line$NC"
    done
}

# Find default JDK
find_default_jdk() {
    if command -v which >/dev/null 2>&1; then
        local java_path=$(which java 2>/dev/null)
        if [ -n "$java_path" ] && [ -x "$java_path" ]; then
            DEFAULT_JDK="$java_path"
            return 0
        fi
    fi

    if [ -n "$JAVA_HOME" ] && [ -x "$JAVA_HOME/bin/java" ]; then
        DEFAULT_JDK="$JAVA_HOME/bin/java"
        return 0
    fi

    local jdk_paths=(
        "/usr/lib/jvm/java-*/bin/java"
        "/Library/Java/JavaVirtualMachines/*/Contents/Home/bin/java"
        "/mnt/c/Program Files/Java/*/bin/java.exe"
    )

    shopt -s nullglob
    for pattern in "${jdk_paths[@]}"; do
        for exp_path in $pattern; do
            if [ -x "$exp_path" ]; then
                DEFAULT_JDK="$exp_path"
                shopt -u nullglob
                return 0
            fi
        done
    done
    shopt -u nullglob 2>/dev/null

    DEFAULT_JDK=""
    return 1
}

# 完全重写的配置加载函数
load_config() {
    JAR_CONFIGS=()
    local conf_file=""
    local found_conf=false
    
    for file in "${CONF_FILES[@]}"; do
        if [ -f "$file" ]; then
            conf_file="$file"
            found_conf=true
            break
        fi
    done

    if [ "$found_conf" = "false" ]; then
        return 1
    fi
    
    local current_section=""
    local current_name=""
    local current_path=""
    local current_jdk=""
    local current_args=""
    local current_daemon="false"
    local current_timeout=60
    local config_count=0

    while IFS= read -r line || [ -n "$line" ]; do
        line=$(echo "$line" | sed 's/#.*$//' | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
        [ -z "$line" ] && continue
        
        if [[ "$line" =~ ^\[([a-zA-Z0-9_-]+)\]$ ]]; then
            if [ -n "$current_section" ] && [ -n "$current_path" ]; then
                JAR_CONFIGS+=("$current_name|$current_path|$current_jdk|$current_args|$current_daemon|$current_timeout")
                ((config_count++))
            fi
            
            current_section="${BASH_REMATCH[1]}"
            current_name="$current_section"
            current_path=""
            current_jdk=""
            current_args=""
            current_daemon="false"
            current_timeout=60
            
        elif [[ "$line" =~ ^([a-zA-Z]+)[[:space:]]*=[[:space:]]*(.*)$ ]]; then
            local key="${BASH_REMATCH[1]}"
            local value="${BASH_REMATCH[2]}"
            
            case "$key" in
                path) current_path="$value" ;;
                jdk) current_jdk="$value" ;;
                args) current_args="$value" ;;
                daemon) current_daemon="$value" ;;
                startupTimeout) 
                    if [[ "$value" =~ ^[0-9]+$ ]]; then
                        current_timeout="$value"
                    fi ;;
            esac
        fi
    done < "$conf_file"

    if [ -n "$current_section" ] && [ -n "$current_path" ]; then
        JAR_CONFIGS+=("$current_name|$current_path|$current_jdk|$current_args|$current_daemon|$current_timeout")
        ((config_count++))
    fi

    # 初始化PID_MAP
    for config in "${JAR_CONFIGS[@]}"; do
        IFS='|' read -r name path _ _ _ _ <<< "$config"
        PID_MAP["$path"]=$(get_pid_by_path "$path")
    done

    if [ $config_count -eq 0 ]; then
        return 1
    else
        return 0
    fi
}

# Process status functions
is_running() {
    local pid="$1"
    [ -z "$pid" ] && return 1
    kill -0 "$pid" 2>/dev/null
}

# 通过路径获取PID，解决应用名更改问题
get_pid_by_path() {
    local path="$1"
    # 查找所有PID文件
    for pid_file in "$PID_DIR"/*.pid; do
        [ -f "$pid_file" ] || continue
        local pid=$(cat "$pid_file" 2>/dev/null | grep -E '^[0-9]+$' | head -1)
        if [ -n "$pid" ] && is_running "$pid"; then
            # 检查进程是否匹配此JAR路径
            local cmd_line=$(cat /proc/"$pid"/cmdline 2>/dev/null | tr '\0' ' ')
            if [[ "$cmd_line" == *"$path"* ]]; then
                echo "$pid"
                return 0
            fi
        fi
    done
    echo ""
}

get_pid() {
    local name="$1"
    local pid_file="$PID_DIR/$name.pid"
    if [ -f "$pid_file" ]; then
        local pid=$(cat "$pid_file" 2>/dev/null | grep -E '^[0-9]+$' | head -1)
        if [ -n "$pid" ] && is_running "$pid"; then
            echo "$pid"
            return 0
        else
            rm -f "$pid_file"
        fi
    fi
    echo ""
}

# 修复时间获取函数
get_start_time() {
    local pid="$1"
    
    if [ -z "$pid" ] || [ "$pid" = "0" ] || [ "$pid" = "--" ] || ! is_running "$pid"; then
        echo "--"
        return 0
    fi

    if [ -f /proc/"$pid"/stat ]; then
        local start_time=$(cat /proc/"$pid"/stat | awk '{print $22}')
        local uptime=$(cat /proc/uptime | awk '{print $1}')
        local start_seconds=$(awk "BEGIN {print int($uptime - $start_time / 100)}")
        
        if [ $start_seconds -lt 60 ]; then
            echo "${start_seconds}s"
        elif [ $start_seconds -lt 3600 ]; then
            echo "$((start_seconds / 60))m"
        else
            echo "$((start_seconds / 3600))h"
        fi
    else
        echo "--"
    fi
}

# 恢复显示所有端口的函数
get_listening_ports() {
    local pid="$1"
    local ports="--"
    
    if [ -z "$pid" ] || [ "$pid" = "0" ] || [ "$pid" = "--" ] || ! is_running "$pid"; then
        echo "$ports"
        return 0
    fi
    
    if command -v ss >/dev/null 2>&1; then
        ports=$(ss -tulpn 2>/dev/null | awk -v pid="$pid" '$7 ~ "pid=" pid "," {print $5}' | grep -oE ':[0-9]+' | cut -d: -f2 | sort -n | uniq | tr '\n' ',' | sed 's/,$//')
    elif command -v netstat >/dev/null 2>&1; then
        ports=$(netstat -tulpn 2>/dev/null | awk -v pid="$pid" '$7 ~ pid "/java" {print $4}' | grep -oE ':[0-9]+' | cut -d: -f2 | sort -n | uniq | tr '\n' ',' | sed 's/,$//')
    fi
    
    [ -z "$ports" ] && ports="--"
    echo "$ports"
}

# 修复CPU使用函数
get_cpu_usage() {
    local pid="$1"
    
    if [ -z "$pid" ] || [ "$pid" = "0" ] || [ "$pid" = "--" ] || ! is_running "$pid"; then
        echo "--"
        return 0
    fi

    if command -v ps >/dev/null 2>&1; then
        local cpu=$(ps -o %cpu= -p "$pid" 2>/dev/null | awk '{printf "%.1f%%", $1}')
        echo "${cpu:---}"
    else
        echo "--"
    fi
}

# 修复内存使用函数
get_memory_usage() {
    local pid="$1"
    
    if [ -z "$pid" ] || [ "$pid" = "0" ] || [ "$pid" = "--" ] || ! is_running "$pid"; then
        echo "--"
        return 0
    fi

    if command -v ps >/dev/null 2>&1; then
        local memory_kb=$(ps -o rss= -p "$pid" 2>/dev/null)
        if [ -n "$memory_kb" ]; then
            local memory_mb=$(awk "BEGIN {printf \"%.1fM\", $memory_kb/1024}")
            echo "$memory_mb"
        else
            echo "--"
        fi
    else
        echo "--"
    fi
}

is_crashed() {
    local name="$1"
    local current_pid="$2"
    
    for i in "${!LAST_PIDS[@]}"; do
        if echo "${LAST_PIDS[$i]}" | grep -q "^$name:[0-9]\+$"; then
            local prev_pid=$(echo "${LAST_PIDS[$i]}" | cut -d':' -f2)
            LAST_PIDS[$i]="$name:$current_pid"
            if [ -n "$prev_pid" ] && [ "$prev_pid" != "0" ] && [ "$prev_pid" != "$current_pid" ] && ! is_running "$prev_pid"; then
                return 0
            fi
            break
        fi
    done
    return 1
}

# Start/stop functions
start_jar() {
    local config="$1"
    IFS='|' read -r name path jdk_home args daemon startup_timeout <<< "$config"

    [ -z "$name" ] && { echo -e "${RED}Error: Name not configured${NC}"; return 1; }
    [ -z "$path" ] && { echo -e "${RED}Error: Path not configured for $name${NC}"; return 1; }
    [ ! -f "$path" ] && { echo -e "${RED}Error: JAR file not found: $path${NC}"; return 1; }

    # 首先通过路径检查是否已经在运行
    local existing_pid=$(get_pid_by_path "$path")
    if [ -n "$existing_pid" ] && is_running "$existing_pid"; then
        echo -e "${YELLOW}$name is already running (PID: $existing_pid)${NC}"
        return 0
    fi

    # 然后通过名称检查
    local pid=$(get_pid "$name")
    if [ -n "$pid" ] && is_running "$pid"; then
        echo -e "${YELLOW}$name is already running (PID: $pid)${NC}"
        return 0
    fi

    local java_cmd=""
    if [ -n "$jdk_home" ] && [ -x "$jdk_home/bin/java" ]; then
        java_cmd="$jdk_home/bin/java"
    elif [ -n "$DEFAULT_JDK" ]; then
        java_cmd="$DEFAULT_JDK"
    else
        echo -e "${RED}Error: No JDK found for $name${NC}"; return 1;
    fi

    local log_file="$LOG_DIR/$name.log"
    echo -e "Starting ${BLUE}$name${NC}..."
    
    # 直接启动，不检查超时
    $java_cmd $args -jar "$path" >> "$log_file" 2>&1 &
    local new_pid=$!
    
    # 等待进程真正启动
    sleep 2
    if is_running "$new_pid"; then
        echo "$new_pid" > "$PID_DIR/$name.pid"
        PID_MAP["$path"]="$new_pid"
        echo -e "${GREEN}$name started successfully (PID: $new_pid)${NC}"
        for i in "${!LAST_PIDS[@]}"; do
            if echo "${LAST_PIDS[$i]}" | grep -q "^$name:"; then
                LAST_PIDS[$i]="$name:$new_pid"
                break
            fi
        done
        return 0
    else
        echo -e "${RED}Error: $name failed to start. Check log: $log_file${NC}"
        return 1
    fi
}

# 改进的停止函数，更可靠地停止进程
stop_jar() {
    local name="$1"
    local pid=$(get_pid "$name")

    if [ -z "$pid" ] || ! is_running "$pid"; then
        echo -e "${YELLOW}$name is not running${NC}"
        return 0
    fi

    echo -e "Stopping ${BLUE}$name${NC} (PID: $pid) ..."
    
    # 首先尝试优雅停止
    kill "$pid" >/dev/null 2>&1
    
    # 等待进程停止，增加等待时间和检查频率
    local count=0
    local max_wait=30
    while is_running "$pid" && [ $count -lt $max_wait ]; do
        sleep 1
        count=$((count + 1))
        # 每5秒显示一次等待信息
        if [ $((count % 5)) -eq 0 ]; then
            echo -e "${YELLOW}Waiting for $name to stop... ($count/$max_wait seconds)${NC}"
        fi
    done

    # 如果进程还在运行，使用更强力的停止方式
    if is_running "$pid"; then
        echo -e "${YELLOW}Process still running, sending SIGTERM...${NC}"
        kill -TERM "$pid" >/dev/null 2>&1
        sleep 3
        
        if is_running "$pid"; then
            echo -e "${RED}Force terminating ${BLUE}$name${NC} ..."
            kill -9 "$pid" >/dev/null 2>&1
            sleep 2
        fi
    fi

    # 最终检查进程是否真的停止了
    if is_running "$pid"; then
        echo -e "${RED}Error: Failed to stop $name (PID: $pid)${NC}"
        return 1
    fi

    local log_file="$LOG_DIR/$name.log"
    echo "$(date +"%Y-%m-%d %H:%M:%S"): $name stopped manually" >> "$log_file" 2>/dev/null
    
    rm -f "$PID_DIR/$name.pid"
    # 从PID_MAP中移除
    for path in "${!PID_MAP[@]}"; do
        if [ "${PID_MAP[$path]}" = "$pid" ]; then
            unset PID_MAP["$path"]
            break
        fi
    done
    
    for i in "${!LAST_PIDS[@]}"; do
        if echo "${LAST_PIDS[$i]}" | grep -q "^$name:"; then
            LAST_PIDS[$i]="$name:0"
            break
        fi
    done
    echo -e "${GREEN}$name has been stopped${NC}"
    return 0
}

# 修复gen命令，支持指定应用名，并修改日志指向为/dev/null
generate_nohup() {
    local config="$1"
    IFS='|' read -r name path jdk_home args daemon startup_timeout <<< "$config"

    [ -z "$name" ] && { echo -e "${RED}Error: Name not configured${NC}"; return 1; }
    [ -z "$path" ] && { echo -e "${RED}Error: Path not configured for $name${NC}"; return 1; }
    
    if [ ! -f "$path" ]; then
        echo -e "${YELLOW}Warning: JAR file not found for $name: $path${NC}"
    fi

    local java_cmd=""
    if [ -n "$jdk_home" ] && [ -x "$jdk_home/bin/java" ]; then
        java_cmd="$jdk_home/bin/java"
    elif [ -n "$DEFAULT_JDK" ]; then
        java_cmd="$DEFAULT_JDK"
    else
        java_cmd="java"
        echo -e "${YELLOW}Warning: No explicit JDK found for $name, using system default${NC}"
    fi

    # 修改：日志指向默认改为/dev/null
    local log_file="/dev/null"
    local nohup_cmd="nohup $java_cmd $args -jar \"$path\" > \"$log_file\" 2>&1 &"
    
    echo -e "\n${BLUE}Application: $name${NC}"
    echo -e "${CYAN}Generated nohup command:${NC}"
    echo -e "$nohup_cmd"
    echo -e "${CYAN}Log file location:${NC} $log_file"
}

# Simplified status table functions
prepare_sorted_data() {
    SORTED_DATA=()
    for config in "${JAR_CONFIGS[@]}"; do
        IFS='|' read -r name path jdk_home args daemon startup_timeout <<< "$config"
        
        # 优先使用路径查找PID，解决应用名更改问题
        local pid="${PID_MAP[$path]}"
        if [ -z "$pid" ]; then
            pid=$(get_pid "$name")
        fi
        
        local status="Stopped"
        
        local status_weight=1
        if [ -n "$pid" ] && is_running "$pid"; then
            status="Running"
            status_weight=3
        elif is_crashed "$name" "$pid"; then
            status="Crashed"
            status_weight=2
        fi

        local ports=$(get_listening_ports "$pid")
        local memory=$(get_memory_usage "$pid")
        local cpu=$(get_cpu_usage "$pid")
        
        SORTED_DATA+=("$status_weight|$name|$pid|$path|$ports|$memory|$cpu|$status")
    done
}

# 修复排序函数，确保从大到小排序
sort_data() {
    prepare_sorted_data
    
    if [ ${#SORTED_DATA[@]} -eq 0 ]; then
        return 0
    fi
    
    case "$SORT_FIELD" in
        status)
            # 状态排序：Running > Crashed > Stopped
            SORTED_DATA=($(printf "%s\n" "${SORTED_DATA[@]}" | sort -t'|' -k1,1nr))
            ;;
        memory)
            # 内存排序：从大到小，处理 "--" 值
            SORTED_DATA=($(printf "%s\n" "${SORTED_DATA[@]}" | while IFS='|' read -r weight name pid path ports memory cpu status; do
                if [[ "$memory" == "--" ]] || [[ "$memory" == *"M" ]]; then
                    # 将 "--" 转换为 0，提取数字部分
                    memory_num=$(echo "$memory" | sed 's/M//' | awk '{if ($0 == "--") print 0; else print $1}')
                    printf "%s|%s|%s|%s|%s|%s|%s|%s\n" "$memory_num" "$name" "$pid" "$path" "$ports" "$memory" "$cpu" "$status"
                else
                    printf "0|%s|%s|%s|%s|%s|%s|%s\n" "$name" "$pid" "$path" "$ports" "$memory" "$cpu" "$status"
                fi
            done | sort -t'|' -k1,1nr | cut -d'|' -f2-))
            ;;
        cpu)
            # CPU排序：从大到小，处理 "--" 值
            SORTED_DATA=($(printf "%s\n" "${SORTED_DATA[@]}" | while IFS='|' read -r weight name pid path ports memory cpu status; do
                if [[ "$cpu" == "--" ]] || [[ "$cpu" == *"%" ]]; then
                    # 将 "--" 转换为 0，提取数字部分
                    cpu_num=$(echo "$cpu" | sed 's/%//' | awk '{if ($0 == "--") print 0; else print $1}')
                    printf "%s|%s|%s|%s|%s|%s|%s|%s\n" "$cpu_num" "$name" "$pid" "$path" "$ports" "$memory" "$cpu" "$status"
                else
                    printf "0|%s|%s|%s|%s|%s|%s|%s\n" "$name" "$pid" "$path" "$ports" "$memory" "$cpu" "$status"
                fi
            done | sort -t'|' -k1,1nr | cut -d'|' -f2-))
            ;;
        *)
            SORTED_DATA=($(printf "%s\n" "${SORTED_DATA[@]}" | sort -t'|' -k1,1nr))
            ;;
    esac
}

calculate_status_column_widths() {
    STATUS_COL_WIDTHS=(0 0 0 0 0 0 0)
    local headers=("Process Name" "PID" "JAR Path" "Ports" "Memory" "CPU" "Status")
    
    for i in "${!headers[@]}"; do
        header_len=${#headers[$i]}
        STATUS_COL_WIDTHS[$i]=$(( header_len > MIN_COL_WIDTH ? header_len : MIN_COL_WIDTH ))
    done
    
    for item in "${SORTED_DATA[@]}"; do
        IFS='|' read -r _ name pid path ports memory cpu status <<< "$item"
        
        local row=("$name" "$pid" "$path" "$ports" "$memory" "$cpu" "$status")
        
        for i in "${!row[@]}"; do
            len=${#row[$i]}
            if [ $len -gt ${STATUS_COL_WIDTHS[$i]} ]; then
                STATUS_COL_WIDTHS[$i]=$len
            fi
        done
    done
    
    for i in "${!STATUS_COL_WIDTHS[@]}"; do
        STATUS_COL_WIDTHS[$i]=$((STATUS_COL_WIDTHS[$i] + PADDING))
    done
}

create_status_table_line() {
    local cols=("$@")
    local line="$COLUMN_SEP"
    
    for i in "${!cols[@]}"; do
        local width=${STATUS_COL_WIDTHS[$i]}
        local content="${cols[$i]}"
        
        if [ ${#content} -gt $((width - PADDING)) ]; then
            content="${content:0:$((width - PADDING - 3))}..."
        fi
        
        printf -v padded "%-${width}s" "$content"
        line+="$padded$COLUMN_SEP"
    done
    
    echo "$line"
}

create_status_header_sep() {
    local sep="$CORNER"
    
    for width in "${STATUS_COL_WIDTHS[@]}"; do
        sep+=$(printf "%0.s$HEADER_LINE" $(seq 1 $width))
        sep+="$CORNER"
    done
    
    echo "$sep"
}

# 修复动态刷新显示问题，并支持Ctrl+C和q键退出
show_status() {
    local dynamic=$1
    sort_data
    
    local current_lines=$(( ${#SORTED_DATA[@]} + 4 ))
    
    if [ "$dynamic" = "true" ] && [ "$FIRST_DYNAMIC_RUN" = "false" ]; then
        # 使用更稳定的清除方法
        for ((i=0; i<LAST_DISPLAYED_LINES; i++)); do
            echo -ne "\033[1A\033[2K"
        done
    fi
    FIRST_DYNAMIC_RUN=false
    
    calculate_status_column_widths
    echo -e "${BLUE}$(create_status_table_line "Process Name" "PID" "JAR Path" "Ports" "Memory" "CPU" "Status")${NC}"
    echo -e "${BLUE}$(create_status_header_sep)${NC}"
    
    if [ ${#SORTED_DATA[@]} -gt 0 ]; then
        for item in "${SORTED_DATA[@]}"; do
            IFS='|' read -r status_weight name pid path ports memory cpu status <<< "$item"
            
            # 根据状态设置状态列的颜色
            local status_color=$NC
            if [ "$status" = "Running" ]; then
                status_color=$GREEN
            elif [ "$status" = "Crashed" ]; then
                status_color=$RED
            else
                status_color=$YELLOW
            fi

            # 创建表格行，状态列单独着色
            local name_col=$(printf "%-${STATUS_COL_WIDTHS[0]}s" "$name")
            local pid_col=$(printf "%-${STATUS_COL_WIDTHS[1]}s" "$pid")
            local path_col=$(printf "%-${STATUS_COL_WIDTHS[2]}s" "$path")
            local ports_col=$(printf "%-${STATUS_COL_WIDTHS[3]}s" "$ports")
            local memory_col=$(printf "%-${STATUS_COL_WIDTHS[4]}s" "$memory")
            local cpu_col=$(printf "%-${STATUS_COL_WIDTHS[5]}s" "$cpu")
            local status_col=$(printf "%-${STATUS_COL_WIDTHS[6]}s" "$status")
            
            # 构建完整的行
            local line="${COLUMN_SEP}${name_col}${COLUMN_SEP}${pid_col}${COLUMN_SEP}${path_col}${COLUMN_SEP}${ports_col}${COLUMN_SEP}${memory_col}${COLUMN_SEP}${cpu_col}${COLUMN_SEP}${status_color}${status_col}${NC}${COLUMN_SEP}"
            
            echo -e "$line"
        done
    else
        local empty_msg="No applications configured"
        local empty_line="$COLUMN_SEP"
        local total_width=0
        for width in "${STATUS_COL_WIDTHS[@]}"; do
            total_width=$((total_width + width + 1))
        done
        
        if [ $total_width -gt ${#empty_msg} ]; then
            local padding_left=$(( (total_width - ${#empty_msg} - 2) / 2 ))
            printf -v padded "%*s%s%*s" $padding_left "" "$empty_msg" $padding_left ""
            empty_line+="$padded$COLUMN_SEP"
        else
            empty_line+=" $empty_msg $COLUMN_SEP"
        fi
        
        echo -e "${YELLOW}$empty_line${NC}"
    fi

    local sort_info=""
    case "$SORT_FIELD" in
        status) sort_info="Status (Running first)" ;;
        memory) sort_info="Memory usage (Highest to lowest)" ;;
        cpu) sort_info="CPU usage (Highest to lowest)" ;;
    esac
    
    if [ "$dynamic" = "true" ]; then
        echo -e "\n${CYAN}Dynamic mode - refreshing every $REFRESH_INTERVAL seconds | Sorting: $sort_info | Press Ctrl+C or 'q' to exit${NC}"
    else
        echo -e "\n${CYAN}Sorting: $sort_info${NC}"
    fi
    
    LAST_DISPLAYED_LINES=$current_lines
}

# Show help information
show_help() {
    cat << EOF
Usage: $SCRIPT_NAME [command] [options]

Commands:
  start [name]      Start all or specified JAR packages
  stop [name]       Stop all or specified JAR packages
  restart [name]    Restart all or specified JAR packages
  status            Display status in simplified table format
  status -f         Continuously refresh status (press Ctrl+C or 'q' to exit)
  status -s -m      Sort by memory usage (highest to lowest)
  status -s -c      Sort by CPU usage (highest to lowest)
  status -s -m -f   Dynamically refresh sorted by memory
  status -s -c -f   Dynamically refresh sorted by CPU
  gen [name]        Generate nohup command for all or specified JAR (logs to /dev/null)
  install           Install $SCRIPT_NAME to /usr/bin (requires root)
  uninstall         Remove $SCRIPT_NAME from /usr/bin (requires root)
  jdk [name]        Display JDK information
  -j [name]         Same as jdk command
  help              Display this help information

Configuration:
  Configuration file is searched in (in order):
  1. Current directory: ./jm.conf
  2. Script directory: $SCRIPT_DIR/jm.conf  
  3. System directory: /etc/jm/jm.conf
  Current directory has highest priority.
EOF
}

# Daemon mode
daemon_mode() {
    echo "Entering daemon mode (monitoring daemon-enabled applications)..."
    while true; do
        for config in "${JAR_CONFIGS[@]}"; do
            IFS='|' read -r name path jdk_home args daemon startup_timeout <<< "$config"
            [ "$daemon" != "true" ] && continue
            
            local pid=$(get_pid "$name")
            if ! is_running "$pid"; then
                echo "$(date): $name stopped unexpectedly, attempting restart..."
                start_jar "$config"
            fi
        done
        sleep 10
    done
}

# Parse sort options
parse_sort_options() {
    local args=("$@")
    local i=0
    while [ $i -lt ${#args[@]} ]; do
        case "${args[$i]}" in
            -s)
                if [ $((i+1)) -lt ${#args[@]} ]; then
                    case "${args[$((i+1))]}" in
                        -m)
                            SORT_FIELD="memory"
                            i=$((i+1))
                            ;;
                        -c)
                            SORT_FIELD="cpu"
                            i=$((i+1))
                            ;;
                    esac
                fi
                ;;
        esac
        i=$((i+1))
    done
}

# 动态刷新状态函数，支持Ctrl+C和q键退出
dynamic_status() {
    echo -e "\033[?25l"
    # 设置陷阱捕获Ctrl+C
    trap 'echo -e "\033[?25h${NC}"; exit 0' SIGINT
    
    while true; do
        load_config
        show_status "true"
        
        # 使用read -t来等待并检查按键输入
        if read -t $REFRESH_INTERVAL -n1 -r key; then
            if [ "$key" = "q" ] || [ "$key" = "Q" ]; then
                echo -e "\033[?25h${NC}"
                exit 0
            fi
        fi
    done
}

# Main logic
main() {
    if [ $# -eq 0 ]; then
        show_help
        return 0
    fi

    if [ "$1" = "jdk" ] || [ "$1" = "-j" ]; then
        find_default_jdk
        load_config
        show_jdk_info "$2"
        return 0
    fi

    if [ "$1" = "install" ]; then
        install_script
        return 0
    elif [ "$1" = "uninstall" ]; then
        uninstall_script
        return 0
    fi

    find_default_jdk || echo -e "${YELLOW}Warning: No system default JDK found${NC}"
    
    # Load configuration
    if ! load_config; then
        if [ "$1" = "status" ]; then
            # For status command, we can continue but show empty table
            echo -e "${YELLOW}No configuration found or no valid applications configured${NC}"
        elif [ "$1" != "help" ]; then
            echo -e "${RED}Error: No configuration found or no valid applications configured${NC}"
            return 1
        fi
    fi

    if [ "$1" = "status" ]; then
        parse_sort_options "$@"
        
        local dynamic=false
        for arg in "$@"; do
            if [ "$arg" = "-f" ]; then
                dynamic=true
                break
            fi
        done
        
        if [ "$dynamic" = "true" ]; then
            dynamic_status
        else
            show_status "false"
        fi
        return 0
    fi

    if [ "$1" = "gen" ]; then
        if [ ${#JAR_CONFIGS[@]} -eq 0 ]; then
            echo -e "${YELLOW}Warning: No configurations found${NC}"
            return 1
        fi

        if [ $# -eq 2 ]; then
            local target_name="$2"
            local found=0
            for config in "${JAR_CONFIGS[@]}"; do
                IFS='|' read -r name _ <<< "$config"
                if [ "$name" = "$target_name" ]; then
                    generate_nohup "$config"
                    found=1
                    break
                fi
            done
            if [ $found -eq 0 ]; then
                echo -e "${RED}Error: Configuration for '$target_name' not found${NC}"
                if [ ${#JAR_CONFIGS[@]} -gt 0 ]; then
                    echo -e "${CYAN}Available applications:${NC}"
                    for config in "${JAR_CONFIGS[@]}"; do
                        IFS='|' read -r name _ <<< "$config"
                        echo "  - $name"
                    done
                fi
            fi
        else
            echo -e "${CYAN}Generating nohup commands for all applications:${NC}"
            for config in "${JAR_CONFIGS[@]}"; do
                generate_nohup "$config"
            done
        fi
        return 0
    fi

    case "$1" in
        start|stop|restart|daemon)
            if [ ${#JAR_CONFIGS[@]} -eq 0 ]; then
                echo -e "${YELLOW}Warning: No configurations found${NC}"
                if [ "$1" != "stop" ]; then
                    return 1
                fi
            fi
            ;;
    esac

    case "$1" in
        start)
            if [ $# -eq 2 ]; then
                local found=0
                for config in "${JAR_CONFIGS[@]}"; do
                    IFS='|' read -r name _ <<< "$config"
                    if [ "$name" = "$2" ]; then
                        start_jar "$config"
                        found=1
                        break
                    fi
                done
                [ $found -eq 0 ] && echo -e "${RED}Error: Configuration for $2 not found${NC}"
            else
                echo -e "${CYAN}Starting all applications...${NC}"
                for config in "${JAR_CONFIGS[@]}"; do
                    start_jar "$config"
                done
            fi
            ;;
        stop)
            if [ $# -eq 2 ]; then
                stop_jar "$2"
            else
                echo -e "${CYAN}Stopping all applications...${NC}"
                for config in "${JAR_CONFIGS[@]}"; do
                    IFS='|' read -r name _ <<< "$config"
                    stop_jar "$name"
                done
            fi
            ;;
        restart)
            if [ $# -eq 2 ]; then
                stop_jar "$2"
                local found=0
                for config in "${JAR_CONFIGS[@]}"; do
                    IFS='|' read -r name _ <<< "$config"
                    if [ "$name" = "$2" ]; then
                        start_jar "$config"
                        found=1
                        break
                    fi
                done
                [ $found -eq 0 ] && echo -e "${RED}Error: Configuration for $2 not found${NC}"
            else
                echo -e "${CYAN}Restarting all applications...${NC}"
                for config in "${JAR_CONFIGS[@]}"; do
                    IFS='|' read -r name _ <<< "$config"
                    stop_jar "$name"
                    start_jar "$config"
                done
            fi
            ;;
        help)
            show_help
            ;;
        daemon)
            daemon_mode
            ;;
        *)
            echo -e "${RED}Error: Unknown command '$1'${NC}"
            show_help
            exit 1
            ;;
    esac
}

if [ "$1" = "systemd" ]; then
    main daemon &
    echo $! > "$PID_DIR/daemon.pid"
    exit 0
fi

main "$@"