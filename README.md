SNMP template for Lenovo ThinkSystem SR630, SR650, SR655, SR665, HX650 and VX650 (all generations) via the XClarity Controller (XCC) SNMP agent, LENOVO-XCC-MIB (1.3.6.1.4.1.19046.11.1).
      Autor: Frank Aguilieri
      Versão: 2.1
      Data: 15/07/2026
      GitHub: https://github.com/frankaguilieri/Zabbix-Lenovo-ThinkSystem-SR630-SR650-SR655-SR665-HX650-VX650-by-XCC-SNMP

      Coverage: memory, power supplies (PSU) + redundancy, disks (basic health plus per-drive stats: predictive failure, media errors, SSD remaining life, temperature), battery (RAID controller status/voltage/temperature and system battery via the voltage table), processor, fans (status + numeric speed), network interfaces, Fibre Channel HBA, power consumption (total + CPU/memory/other + available/remaining), temperature, voltage, GPU (inventory), firmware, LEDs, SNMP trap and availability (ICMP + SNMP).

      =======================================================================
      SERVER-SIDE CONFIGURATION (XCC) - required before the host will poll
      =======================================================================
      These ThinkSystem servers (XCC / XCC2) only provide an SNMPv3 agent for polling; SNMPv1/v2c is available for traps only. The management port is out-of-band and stays up even when the server is powered off.

      1) GLOBAL SNMP AGENT - Web: BMC Configuration > Network > SNMP.
         - Enable the SNMPv3 Agent.
         - Fill in Contact and Location (mandatory - the agent will NOT persist without them).
         - CLI equivalent (SSH into the XCC): snmp -a3 enabled -l <location> -cn <contact>
         - Verify with 'snmp' (no args); it must show '-a3 enabled'.

      2) GLOBAL SECURITY - disable forced password change at first login.
         A freshly created/default account is in a 'must change password at first login' state and will NOT authenticate over SNMP until that is cleared. Disable it so the monitoring account is usable non-interactively:
         - Web: BMC Configuration > Security > (Password/Account Security Level or Global Login Settings) > turn OFF 'Force user to change password on first access' (a.k.a. 'Change password at first login').
         - Also make sure the monitoring account's password is set and not expired.

      3) MONITORING USER - create a read-only account with SNMP access and SNMPv3 auth/priv.
         On this firmware family the SNMPv3 authentication passphrase is the account login password (-p) and only the privacy passphrase is set separately (-spw). Configure via CLI (adjust the account index shown by 'users'):
           users -N <index> -p "<LOGIN/AUTH_PASS>" -ai web|ssh|snmp -sauth HMAC-SHA -spriv AES -spw "<PRIV_PASS>" -sacc Get
         - '-ai ...|snmp' grants SNMP to the account; '-sacc Get' makes it read-only.
         - -sauth HMAC-SHA  => authentication protocol SHA-1.
         - -spriv AES       => privacy protocol AES-128.
         - Both passphrases must be >= 8 characters.
         - The web 'SNMP Settings' toggle on the user often reverts if auth/priv are incomplete; the CLI form above persists reliably.
         - Verify with 'users' (the account must show the 'snmp' attribute).

      4) TRAPS (optional) - BMC Configuration > Notification > SNMP Trap; point the destination to the Zabbix server/proxy IP (UDP 162).

      5) VALIDATE from the Zabbix server/proxy before creating the host:
           SNMPCONFPATH=/dev/null snmpget -v3 -l authPriv -u <user> -a SHA -A "<AUTH_PASS>" -x AES -X "<PRIV_PASS>" <XCC_IP> 1.3.6.1.2.1.1.1.0

      =======================================================================
      ZABBIX-SIDE CONFIGURATION - Host SNMP interface (SNMPv3)
      =======================================================================
      Create the host pointing to the XCC management IP (NOT the OS IP) and add an SNMP interface. In the interface, set SNMP version = SNMPv3 and fill the security fields to match EXACTLY what worked in the snmpget above:

         Field (Zabbix)            Value
         -----------------------  -------------------------------------------
         SNMP version             SNMPv3
         Context name             (leave empty)
         Security name            <monitoring user, e.g. zabbix_mon>
         Security level           authPriv
         Authentication protocol  SHA1        (XCC uses HMAC-SHA = SHA-1; do NOT use SHA224/256/384/512)
         Authentication passphrase <the account login password / -A value>
         Privacy protocol         AES128      (XCC uses AES; do NOT use AES192/256 or DES)
         Privacy passphrase       <the -X / -spw value>

      IMPORTANT: these ThinkSystem XCCs only support SHA-1 for authentication and AES-128 for privacy on the user profile. Selecting SHA256/AES256 in Zabbix will fail authentication/decryption even though the credentials are 'correct'. Any mismatch here reproduces errors like 'Unknown user name', 'authentication failure' or 'decryption error'.

      You can hard-code these values on the interface or reference macros and set them per host, e.g. Security name = {$SNMPV3_USER}, Auth passphrase = {$SNMPV3_AUTHPASS}, Priv passphrase = {$SNMPV3_PRIVPASS} (define them on the host, ideally as Secret macros). Then link this template and adjust the tuning macros ({$POWER_MAX}, temperature thresholds, {$HBA_FC.NAME_MATCHES} / {$HBA_FC.NAME_NOT_MATCHES}, {$GPU.NAME_MATCHES}, {$DRIVE_LIFE_WARN}, etc.).

      NOTES:
      - Fuel-gauge and drive-stat OIDs come from the LENOVO-XCC-MIB but vary by model/firmware (e.g. SR665 does not expose fuelGaugePowerRemaining, SR655 does); validate via snmpwalk.
      - Disks/RAID require raidOOBCapable enabled (a dedicated item monitors this).
      - HBA FC discovery reuses the adapter-port table filtered by name: a port must match {$HBA_FC.NAME_MATCHES} AND must not match {$HBA_FC.NAME_NOT_MATCHES}. This keeps Ethernet NICs from Broadcom/Emulex/QLogic (which also make FC HBAs) out of the HBA rule. On some firmwares the HBA is not exposed over SNMP and needs Redfish/REST.
      - GPU: the MIB exposes inventory only (name/memory). GPU health/temperature are not available over SNMP; they require Redfish/REST.
      - Network interface here = adapter/link status (not traffic/errors). For bps/errors use the IF-MIB on the OS or the switch.
      - Component alerts (fan, overall health, memory, CPU, disk, voltage) are suppressed while the server is powered off (dependency on the 'Server is powered off' trigger).
      - PSU redundancy: the calculated item psu.failed.count sums the supply health codes; the redundancy trigger fires with 2+ supplies installed and at least one degraded.
