
#!/bin/bash

set -euo pipefail

PLAYBOOK="playbook/site.yaml"
VAULT="vaults/users/user.yaml"
USER="user"

usage() {
cat <<EOF

Usage:
$0 -i <inventory_file> -a <action>

Actions:
ping
disk_cleanup
pki_backup
download_patch
install_patch
reboot
validate
full_patch

Examples:
$0 -i inventory/prod.ini -a ping

$0 -i inventory/prod.ini -a disk_cleanup

$0 -i inventory/prod.ini -a pki_backup

$0 -i inventory/prod.ini -a download_patch

$0 -i inventory/prod.ini -a install_patch

$0 -i inventory/prod.ini -a reboot

$0 -i inventory/prod.ini -a validate

$0 -i inventory/prod.ini -a full_patch

EOF
exit 1
}

while [[ $# -gt 0 ]]
do
case "$1" in
-i|--inventory)
INVENTORY="$2"
shift 2
;;
-a|--action)
ACTION="$2"
shift 2
;;
-h|--help)
usage
;;
*)
echo "Unknown option: $1"
usage
;;
esac
done

[[ -z "${INVENTORY:-}" ]] && usage
[[ -z "${ACTION:-}" ]] && usage

if [[ ! -f "$INVENTORY" ]]; then
echo "ERROR: Inventory file not found: $INVENTORY"
exit 1
fi

HOST_COUNT=$(grep -Ev '^\s*$|^\s*#|^.*' "$INVENTORY" | wc -l)

echo "=================================================="
echo " Linux Patch Automation"
echo "=================================================="
echo " Inventory : $INVENTORY"
echo " Hosts      : $HOST_COUNT"
echo " Action     : $ACTION"
echo "=================================================="

run_ansible() {

local TAG=$1

ansible-playbook \
    -i "$INVENTORY" \
    "$PLAYBOOK" \
    -u "$USER" \
    -e @"$VAULT" \
    --tags "$TAG"

}

confirm_snapshot() {

echo
echo "**************************************************"
echo " SNAPSHOT CONFIRMATION"
echo "**************************************************"
echo " Target Hosts : $HOST_COUNT"
echo

read -rp "Have you taken VM snapshots for all servers? (yes/no): " SNAP

if [[ "$SNAP" != "yes" ]]; then
    echo "Patch installation aborted."
    exit 1
fi

}

confirm_reboot() {

echo
echo "**************************************************"
echo " REBOOT CONFIRMATION"
echo "**************************************************"
echo " Target Hosts : $HOST_COUNT"
echo

read -rp "Are you sure you want to reboot all servers? (yes/no): " REBOOT

if [[ "$REBOOT" != "yes" ]]; then
    echo "Reboot cancelled."
    exit 1
fi

}

case "$ACTION" in

ping)

    echo "Running connectivity validation..."
    run_ansible ping
    ;;

disk_cleanup)

    echo "Running disk validation and cleanup..."
    run_ansible disk_cleanup
    ;;

pki_backup)

    echo "Running PKI backup..."
    run_ansible pki_backup
    ;;

download_patch)

    echo "Downloading patches..."
    run_ansible download_patch
    ;;

install_patch)

    confirm_snapshot

    echo "Installing patches..."
    run_ansible install_patch
    ;;

reboot)

    confirm_reboot

    echo "Rebooting servers..."
    run_ansible reboot
    ;;

validate)

    echo "Running post-patch validation..."
    run_ansible validate
    ;;

full_patch)

    echo
    echo "Starting Full Patch Cycle..."
    echo

    echo "Step 1/8 - Ping Validation"
    run_ansible ping

    echo
    echo "Step 2/8 - Disk Validation & Cleanup"
    run_ansible disk_cleanup

    echo
    echo "Step 3/8 - Backup PKI Certificates"
    run_ansible pki_backup

    echo
    echo "Step 4/8 - Download Patches"
    run_ansible download_patch

    echo
    echo "Step 5/8 - Snapshot Confirmation"
    confirm_snapshot

    echo
    echo "Step 6/8 - Install Patches"
    run_ansible install_patch

    echo
    echo "Step 7/8 - Reboot Confirmation"
    confirm_reboot

    echo
    echo "Rebooting Servers"
    run_ansible reboot

    echo
    echo "Step 8/8 - Validation"
    run_ansible validate

    echo
    echo "Full Patch Cycle Completed Successfully."
    ;;

*)

    usage
    ;;

esac
