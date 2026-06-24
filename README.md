
If you want runtime confirmations, don't pass --snapshot-confirm or --reboot-confirm. The script should prompt interactively when those actions are executed.
Usage
./patch.sh -i inventory/prod.ini -a ping

./patch.sh -i inventory/prod.ini -a disk_cleanup

./patch.sh -i inventory/prod.ini -a pki_backup

./patch.sh -i inventory/prod.ini -a download_patch

./patch.sh -i inventory/prod.ini -a install_patch

./patch.sh -i inventory/prod.ini -a reboot

./patch.sh -i inventory/prod.ini -a validate

./patch.sh -i inventory/prod.ini -a full_patch
Writing
#!/bin/bash
set -e
PLAYBOOK="playbook/site.yaml" VAULT="vaults/users/user.yaml" USER="user"
usage() { cat << EOF
Usage: $0 -i <inventory_file> -a 
Actions: ping disk_cleanup pki_backup download_patch install_patch reboot validate full_patch
Examples: $0 -i inventory/prod.ini -a ping $0 -i inventory/prod.ini -a disk_cleanup $0 -i inventory/prod.ini -a install_patch $0 -i inventory/prod.ini -a full_patch
EOF exit 1 }
while [[ $# -gt 0 ]] do case "$1" in -i|--inventory) INVENTORY="$2" shift 2 ;; -a|--action) ACTION="$2" shift 2 ;; *) usage ;; esac done
[[ -z "$INVENTORY" ]] && usage [[ -z "$ACTION" ]] && usage
if [[ ! -f "$INVENTORY" ]]; then echo "ERROR: Inventory file not found: $INVENTORY" exit 1 fi
HOST_COUNT=$(grep -Ev '^\s*$|^\s*#|^.*' "$INVENTORY" | wc -l)
echo "==========================================" echo "Inventory : $INVENTORY" echo "Hosts      : $HOST_COUNT" echo "Action     : $ACTION" echo "=========================================="
run_ansible() { local TAG=$1
ansible-playbook \
  -i "$INVENTORY" \
  "$PLAYBOOK" \
  -u "$USER" \
  -e @"$VAULT" \
  --tags "$TAG"
}
confirm_snapshot() {
echo
echo "Target Hosts : $HOST_COUNT"
read -p "Have you taken VM snapshots? (yes/no): " SNAP

if [[ "$SNAP" != "yes" ]]; then
    echo "Snapshot confirmation failed."
    exit 1
fi
}
confirm_reboot() {
echo
echo "Target Hosts : $HOST_COUNT"
read -p "Are you sure you want to reboot these servers? (yes/no): " REBOOT

if [[ "$REBOOT" != "yes" ]]; then
    echo "Reboot cancelled."
    exit 1
fi
}
case "$ACTION" in
ping)
    run_ansible ping
    ;;

disk_cleanup)
    run_ansible disk_cleanup
    ;;

pki_backup)
    run_ansible pki_backup
    ;;

download_patch)
    run_ansible download_patch
    ;;

install_patch)

    confirm_snapshot
    run_ansible install_patch
    ;;

reboot)

    confirm_reboot
    run_ansible reboot
    ;;

validate)

    run_ansible validate
    ;;

full_patch)

    run_ansible ping

    run_ansible disk_cleanup

    run_ansible pki_backup

    run_ansible download_patch

    confirm_snapshot

    run_ansible install_patch

    confirm_reboot

    run_ansible reboot

    run_ansible validate
    ;;

*)

    usage
    ;;
esac
Ansible Tags Required
Use these tags in site.yaml:
ping
disk_cleanup
pki_backup
download_patch
install_patch
reboot
validate
Runtime Flow
Install Patch
$ ./patch.sh -i inventory/prod.ini -a install_patch

Target Hosts : 25
Have you taken VM snapshots? (yes/no): yes

--> Install starts
Reboot
$ ./patch.sh -i inventory/prod.ini -a reboot

Target Hosts : 25
Are you sure you want to reboot these servers? (yes/no): yes

--> Reboot starts
Full Patch Cycle
Ping
↓
Disk Validation & Cleanup
↓
PKI Backup
↓
Patch Download
↓
Snapshot Confirmation
↓
Patch Install
↓
Reboot Confirmation
↓
Reboot
↓
Validation (uptime + yum history)
This is typically the safest approach for production patching because confirmations happen immediately before the risky actions rather than being passed as command-line flags.