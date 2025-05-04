#!/usr/local/bin/bash

# Defaults
LOG_FILE=""
LOG_MAX_SIZE=1048576
gracefultime=900

# Help function
usage() {
    echo "Usage: $0 -t <PING_TARGET> -s <STATUS_FILE> -d <DAYS_LIMIT> -a <ACTION_COMMAND> [-l <LOG_FILE>]"
    echo ""
    echo "  -t  IP or hostname to ping"
    echo "  -s  Path to status file to track downtime (On its own or in combination with user Home directory, please enter subsirectory in user home directory)"
    echo "  -d  Number of days before triggering action"
    echo "  -a  Action to perform when downtime threshold is reached (ShutdownVBox = also an option)"
    echo "  -l  (Optional) Path to log file  (On its own or in combination with user Home directory, please enter subsirectory in user home directory)"
    echo "  -m  (Optional) Max log size in bytes before rotation (default: 1048576 = 1MB)"
    echo "  -g  (Optional) Gracefull shutdown time, standaard 15 minutes (900 seconds)"
    echo "  -h  (Optional) User User Home Directory, better for multiple automations (Empty = Not home directory / 1/Yy/Yes = Use home directory)"
    exit 1
}

# Parse options
while getopts ":t:s:d:a:l:m:g:h:" opt; do
  case ${opt} in
    t ) PING_TARGET="$OPTARG" ;;
    s ) STATUS_FILE="$OPTARG" ;;
    d ) DAYS_LIMIT="$OPTARG" ;;
    a ) ACTION_COMMAND="$OPTARG" ;;
    l ) LOG_FILE="$OPTARG" ;;
    m ) LOG_MAX_SIZE="$OPTARG" ;;
    g ) gracefultime="$OPTARG" ;;
    h ) HOME_DIR="$OPTARG" ;;
    \? ) echo "Invalid option: -$OPTARG" >&2; usage ;;
    : ) echo "Option -$OPTARG requires an argument." >&2; usage ;;
  esac
done

## Check if we want to use the User home directopry or not
#
if [ "$HOME_DIR" = "Y" ] || [ "$HOME_DIR" = "YES" ] || [ "$HOME_DIR" = "JA" ]  || [ "$HOME_DIR" = "Yes" ] || [ "$HOME_DIR" = "1" ]; then
  STATUS_FILE="/home/${USER}/${STATUS_FILE}"
  LOG_FILE="/home/${USER}/${LOG_FILE}"
fi
# Validate required params
if [ -z "$PING_TARGET" ] || [ -z "$STATUS_FILE" ] || [ -z "$DAYS_LIMIT" ] || [ -z "$ACTION_COMMAND" ]; then
    usage
else
        ## We got all the parameters we need, lets go further
        #

        ## Print the parameters
        #
        echo "Recieved following Parameters:"
        echo ""
        echo "Ping Target                                               : ${PING_TARGET}"
        echo "Status File                                               : ${STATUS_FILE}"
        echo "How many days internet offline before executing command   : ${DAYS_LIMIT}"
        echo "Action Command                                            : ${ACTION_COMMAND}"
        echo "Log file                                          : ${LOG_FILE}"
        echo "LOG File Maximum Size                                     : ${LOG_MAX_SIZE}"
        echo "Gracefull shutdown time (seconds)                 : ${gracefultime}"
        echo "Use Home Directory                                        : ${HOME_DIR}"

        rotate_log() {
            if [ -n "$LOG_FILE" ] && [ -f "$LOG_FILE" ]; then
                if [[ "$OSTYPE" == "darwin"* ]] || [[ "$OSTYPE" == "freebsd"* ]]; then
                    log_size=$(stat -f%z "$LOG_FILE")
                else
                    log_size=$(stat -c%s "$LOG_FILE")
                fi
                if [ "$log_size" -ge "$LOG_MAX_SIZE" ]; then
                    [ -f "${LOG_FILE}.1" ] && rm -f "${LOG_FILE}.1"
                    mv "$LOG_FILE" "${LOG_FILE}.1"
                    echo "[$(date '+%Y-%m-%d %H:%M:%S')] Log rotated: ${LOG_FILE} -> ${LOG_FILE}.1" > "$LOG_FILE"
                fi
            fi
        }

        # Logging function
        log() {
            local message="$1"
            local timestamp=$(date "+%Y-%m-%d %H:%M:%S")
            if [ -n "$LOG_FILE" ]; then
                rotate_log
                echo "[$timestamp] $message" >> "$LOG_FILE"
            fi
            echo "[$timestamp] $message"
        }

        # Check internet status
        ping -c 1 -W 3 "$PING_TARGET" > /dev/null 2>&1
        if [ $? -eq 0 ]; then
            log "Internet is UP"
            [ -f "$STATUS_FILE" ] && rm -f "$STATUS_FILE" && log "Removed $STATUS_FILE"
        else
            log "Internet is DOWN"
            if [ ! -f "$STATUS_FILE" ]; then
                date +%s > "$STATUS_FILE"
                log "Marked downtime start at $(cat $STATUS_FILE)"
            else
                down_since=$(cat "$STATUS_FILE")
                now=$(date +%s)
                let "days_down = ($now - $down_since) / 86400"
                log "Internet has been down for $days_down day(s)"

                if [ "$days_down" -ge "$DAYS_LIMIT" ]; then
                    log "Internet down for more than $DAYS_LIMIT days. Executing action."
                    #eval "$ACTION_COMMAND" >> "$LOG_FILE" 2>&1
                    if [ "$ACTION_COMMAND" -eq "ShutdownVBox" ]; then
                        ## Fill a string to see if there are running Vm's
                        #
                        RunningVMs=$(VBoxManage list runningvms)
                        log "Found the following machines running: $RunningVMs"

                        if [ "$RunningVMs" != "" ]; then
                                ## Yes the VM for this user is running, make it shutdown gracefully
                                #
                                ## First get the machine name the smart way (From the string), we need to split the string on a [space]
                                #
                                IFS=' ' read -ra VMNAME <<< "$RunningVMs"
                                VboxName=${VMNAME[0]}
                                VboxName="${VboxName//\"}"

                                log "Pressing ACPI Powerbutton for Virtual Machine:  $VboxName to gracefully shutdown the machine"
                                ResultPWRButton=$(VBoxManage controlvm $VboxName acpipowerbutton)

                                log "Give time for the Virtual Machine:  $VboxName to gracefully shutdown, sleep for $gracefultime seconds"
                                sleep $gracefultime

                                log "To be fully sure, poweroff the VM : ${VboxName}"
                                ResultPowerOFF=$(VBoxManage controlvm $VboxName poweroff)

                        fi
                    fi
                fi
            fi
        fi
  fi
