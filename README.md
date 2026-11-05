# NIS2 auf Linux-Systemen — Begleit-Playbooks

Ansible-Automatisierung zum eBook **„Vom Paragrafen zum Playbook: NIS2 auf RHEL umsetzen"**.
Dieses ZIP wächst kapitelweise. Alle Rollen unterstützen **RHEL 9/10** und
**Ubuntu 24.04/26.04 LTS** (Auswahl über `ansible_os_family`, ein gemeinsames `site.yml`).
Aktuell enthalten:

- `roles/hardening` — Kapitel 4: System-Hardening
- `roles/patch_management` — Kapitel 5: Patch- & Schwachstellenmanagement
- `roles/iam` — Kapitel 6: Identity & Access Management
- `roles/logging_monitoring` — Kapitel 7: Logging, Monitoring & Vorfallerkennung
- `roles/backup` — Kapitel 8: Backup, Restore & Business Continuity
- `roles/encryption` — Kapitel 9: Verschlüsselung (Krypto-Policy, NBDE-Tooling)

### RHEL → Ubuntu: Gegenstücke im Überblick

| Thema | RHEL 9/10 | Ubuntu 24.04/26.04 |
|---|---|---|
| Mandatory Access Control | SELinux (enforcing/targeted) | AppArmor (aktiv halten, complain-Profile melden) |
| Host-Firewall | firewalld | ufw |
| Automatische Updates | dnf-automatic | unattended-upgrades (+ needrestart) |
| Schwachstellen-Scan | OVAL (RHEL ≤ 9) / updateinfo (RHEL 10) | OVAL (Canonical USN) + `apt list --upgradable` |
| PAM-Konfiguration | authselect | pam-auth-update (+ eigene faillock-Profile) |
| Audit-Regeln | Beispielregeln aus `/usr/share/audit/sample-rules` | rolleneigener Regelsatz |
| Restic | EPEL | universe |
| Krypto-Policy | update-crypto-policies | crypto-config (ab 26.04) + OpenSSL-/OpenSSH-Drop-in |

Distributionsspezifische Paketnamen stehen in `roles/<rolle>/vars/<OS-Familie>.yml`
(interne `__`-Variablen); überschrieben wird wie gewohnt über die Variablen aus `defaults/`.

## Voraussetzungen

- Ansible (`ansible-core` ≥ 2.15) auf dem Control-Node
- Collections: `ansible.posix`, `community.general`
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```
- Zielsysteme: RHEL 9 oder 10 (bzw. kompatible) oder Ubuntu 24.04/26.04 LTS, SSH-Zugang, sudo/become

## Schnellstart

1. Hosts in `inventory.example.ini` eintragen (Gruppe `hardening_targets`).
2. Trockenlauf (verändert nichts):
   ```bash
   ansible-playbook --check --diff site.yml
   ```
3. Schrittweise anwenden, z. B. nur sysctl oder nur Patching:
   ```bash
   ansible-playbook --tags sysctl site.yml
   ansible-playbook --tags patching site.yml
   ansible-playbook --tags iam site.yml
   ansible-playbook --tags logging,monitoring site.yml
   ansible-playbook --tags backup site.yml
   ansible-playbook --tags encryption site.yml
   ```
4. Vollständig anwenden:
   ```bash
   ansible-playbook site.yml
   ```

## Rolle: hardening (Kapitel 4)

Tags: `packages`, `sshd`, `sysctl`, `selinux`/`apparmor` (gemeinsam: `mac`), `services`, `firewalld`/`ufw` (gemeinsam: `firewall`)

- **SSH:** Drop-in wird vor dem Reload mit `sshd -t` validiert (kein Aussperren).
- **Firewall:** Standardzonenwechsel ist opt-in (Aussperr-Risiko); SSH wird stets zuerst freigegeben.
- **SELinux:** bleibt `enforcing`/`targeted`.
- **Ubuntu:**
  - SSH-Drop-in heißt `10-nis2-hardening.conf`, damit es vor `50-cloud-init.conf` gelesen wird
    (sshd übernimmt den *ersten* Wert; Cloud-Images setzen dort ggf. `PasswordAuthentication yes`).
  - **ufw** ist auf Ubuntu standardmäßig *inaktiv*. Die Rolle setzt zuerst die Freigaben aus
    `hardening_ufw_rules` (Standard: App-Profil `OpenSSH`) und aktiviert ufw danach — alle anderen
    eingehenden Verbindungen werden ab dann abgewiesen. Weitere Dienste (z. B. Webserver) vorher eintragen.
  - **AppArmor** wird aktiv gehalten; Profile im complain-Modus werden gemeldet, nicht umgeschaltet.

## Rolle: patch_management (Kapitel 5)

- Konfiguriert `dnf-automatic` (Standard: nur Sicherheitsupdates, Kernel ausgenommen)
  und aktiviert `dnf-automatic.timer`.
- Installiert `dnf-utils` für `needs-restarting`.
- Richtet einen wöchentlichen Schwachstellen-Scan ein (/var/log/compliance): auf RHEL 8/9 per OVAL, auf RHEL 10 als updateinfo-Report — Red Hat publiziert für RHEL 10 keine OVAL-Daten mehr (Nachfolger CSAF/VEX; auswertbar z. B. mit Grype/Trivy, siehe Kapitel 5.4/11).
- Die OVAL-URL wird automatisch aus der RHEL-Hauptversion abgeleitet.

**Ubuntu:**
- Konfiguriert `unattended-upgrades` über `/etc/apt/apt.conf.d/52nis2-unattended-upgrades`
  (per `#clear` *nur* `-security` inkl. ESM bzw. bei `default` zusätzlich `-updates`;
  Kernel-Pakete über `patch_management_unattended_blacklist` ausgenommen; kein automatischer Reboot)
  und aktiviert `apt-daily.timer`/`apt-daily-upgrade.timer`.
- Installiert `needrestart` (Gegenstück zu `needs-restarting`).
- Wöchentlicher OVAL-Scan gegen die Canonical-USN-Daten des Releases (`openscap-scanner`)
  plus Liste offener Aktualisierungen, ebenfalls nach `/var/log/compliance`.

Wichtige Variablen (siehe `roles/patch_management/defaults/main.yml`):
`patch_management_automatic_upgrade_type` (security|default),
`patch_management_automatic_excludepkgs`, `patch_management_automatic_timer`,
`patch_management_oval_scan_enabled`, `patch_management_oval_oncalendar`,
`patch_management_unattended_blacklist`, `patch_management_unattended_mail` (Ubuntu).

## Rolle: iam (Kapitel 6)

- Installiert SSSD, authselect und oddjob-mkhomedir.
- Aktiviert das authselect-Profil `sssd` mit `with-mkhomedir with-faillock with-sudo`.
- Konfiguriert Kontosperre (faillock) und Mindest-Passwortqualität (pwquality).
- Bringt ein mit `visudo` validiertes sudo-Drop-in für die Admin-Gruppe aus.
- **Nicht** enthalten: der Domänenbeitritt (`ipa-client-install` bzw. LDAP-Anbindung) — bewusst umgebungsspezifisch.
Hinweis: Mitglieder von ssh-admins (sudo) müssen zusätzlich in ssh-users (SSH-Zugang, Rolle hardening) sein — zwei getrennte Gruppen sind Absicht (Login ≠ Privilegien)

**Ubuntu:**
- Statt authselect: `pam-auth-update` mit den Profilen aus `iam_pam_profiles`
  (`mkhomedir`, `faillock`, `faillock_notify`). Die faillock-Profile bringt die Rolle selbst mit,
  da Ubuntu zwar `pam_faillock.so`, aber kein Profil dafür liefert.
  `pam-auth-update` läuft bewusst ohne `--force` — bei lokal geänderten `common-*`-Dateien bricht es ab.
- `sss` (PAM/NSS/sudo) aktivieren die Pakete `libpam-sss`, `libnss-sss` und `libsss-sudo` selbst.
- `pwquality.conf.d` wird angelegt (unter Ubuntu nicht vorhanden, von libpwquality aber ausgewertet).
- Ab 26.04 ist **sudo-rs** Standard: `Defaults logfile` wird dort weggelassen (sudo-rs kennt es nicht,
  `visudo` würde die Datei ablehnen); sudo-rs protokolliert über syslog/journald.
- Domänenbeitritt (z. B. `realm join`) ist auch hier nicht enthalten.

Wichtige Variablen (siehe `roles/iam/defaults/main.yml`):
`iam_authselect_features`, `iam_sudo_admin_group`, `iam_faillock_deny`,
`iam_pwquality_minlen`.

## Rolle: logging_monitoring (Kapitel 7)

Tags: `logging`, `monitoring`

- Stellt journald auf persistente Speicherung mit Aufbewahrung um.
- Übernimmt die auditd-STIG-Beispielregeln nach `/etc/audit/rules.d/` und lädt sie.
  Immutable (`-e 2`) ist über `logging_monitoring_auditd_immutable` opt-in (erfordert Reboot).
- Installiert und aktiviert den `node_exporter` (Prometheus). **EPEL** ist Voraussetzung
  (`logging_monitoring_node_exporter_install_epel: true` bootstrappt es optional).
- Optionale rsyslog-Weiterleitung an einen zentralen Collector
  (`logging_monitoring_rsyslog_forward_enabled`) — produktiv auf TLS/RELP umstellen.
- **Ubuntu:** Die Audit-Beispielregeln liegen dort nur unter `/usr/share/doc` (auf minimierten Systemen
  gar nicht, Pfad wechselt zwischen 24.04 und 26.04). Die Rolle bringt daher einen eigenen,
  an `30-stig.rules`/`31-privileged.rules` angelehnten Regelsatz (`30-nis2.rules`) aus; die
  setuid/setgid-Programme werden beim Ausbringen ermittelt. x86_64 und aarch64 werden berücksichtigt.

## Rolle: backup (Kapitel 8)

Tags: `backup`

- Installiert Restic (aus EPEL) und richtet Umgebungs- und Ausschlussdateien ein.
- Bringt ein Backup-Skript mit Wartungsschleife (backup → forget/prune → check) und einen systemd-Timer aus.
- Schreibt nach Erfolg eine Prometheus-Textfile-Metrik (`restic_backup_last_success_timestamp_seconds`) für die Überwachung aus Kapitel 7.
- **Geheimnisse per Ansible Vault setzen:** `backup_password` und ggf. `backup_env` (z. B. `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` für S3/B2).
- Für **Append-only-Repos** `backup_prune_enabled: false` setzen — `forget`/`prune` laufen dann auf dem gesicherten Backup-Host, nicht auf dem Client.
- EPEL wird (opt-in, backup_install_epel: true) mit importiertem GPG-Schlüssel eingerichtet — restic kommt aus EPEL. Das Repository einmalig manuell initialisieren (restic init); das Skript bricht andernfalls mit klarer Meldung ab.
- **Ubuntu:** restic kommt aus `universe`; `backup_install_epel` hat keine Wirkung. Sonst identisch.

Wichtige Variablen (siehe `roles/backup/defaults/main.yml`):
`backup_repository`, `backup_paths`, `backup_keep_*`, `backup_oncalendar`,
`backup_metrics_enabled`.

## Rolle: encryption (Kapitel 9)

Tags: `encryption`, `crypto`

- Setzt die systemweite Krypto-Policy idempotent (`encryption_crypto_policy`, z. B. `FUTURE`, `DEFAULT:NO-SHA1`, `DEFAULT:PQ`, `DEFAULT:NIS2-HARDENED`).
- Kann optional eine eigene Subpolicy (`.pmod`) ausbringen (`encryption_custom_module_enabled`).
- Installiert optional das NBDE-/LUKS-Tooling (`encryption_install_nbde_tools`).
- **Nicht** enthalten: die LUKS-Einrichtung selbst — datenträgerspezifisch und destruktiv, daher manuell (siehe Kapitel 9).
- Nach Policy-Wechsel wird ein **Reboot empfohlen**, damit alle Dienste die neue Policy übernehmen.

**Ubuntu** (kein `update-crypto-policies`):
- **26.04:** `encryption_crypto_policy` wird auf ein `crypto-config`-Profil abgebildet
  (`DEFAULT`→`default`, `FUTURE`→`future`, `LEGACY`→`legacy`; Subpolicy-Teil wird ignoriert).
  `crypto-config` wirkt auf OpenSSL, GnuTLS, APT und nginx — **nicht auf OpenSSH**.
  Achtung bei `future`: nur noch TLS 1.3, und APT akzeptiert nur Ed25519/Ed448-signierte Repositories.
- **24.04:** kein systemweites Profil; die Rolle weist darauf hin.
- `encryption_custom_module_enabled: true` (Gegenstück zu `NIS2-HARDENED`) bringt auf Ubuntu
  zwei Drop-ins aus: `/etc/ssl/openssl.conf.d/90-nis2-hardened.cnf` (TLS ≥ 1.2, AEAD mit PFS,
  SECLEVEL 3) und `/etc/ssh/sshd_config.d/05-nis2-crypto.conf` (entfernt CBC, SHA-1, UMAC-64, ssh-rsa;
  `RequiredRSASize 3072` — **RSA-Benutzerschlüssel < 3072 Bit werden abgewiesen**).
  Unter 24.04 wird `/etc/ssl/openssl.cnf` dafür um `.include /etc/ssl/openssl.conf.d` ergänzt.
- FIPS: nur über Ubuntu Pro (`pro enable fips-updates`), nicht Teil der Rolle.

## Sicherheitshinweise

- Erstanwendung immer zuerst an einem Testsystem.
- Patching konservativ einführen: erst beobachten/Download-only, dann automatische Installation.
- Reboots gehören in ein definiertes Wartungsfenster — die Rollen lösen keine automatischen Neustarts aus.

## Bezug zu § 30 BSIG

- `hardening`: Nr. 1 (Sicherheit der Informationssysteme), Nr. 7 (Cyberhygiene), Nr. 9 (Zugriffskontrolle).
- `patch_management`: Nr. 5 (Schwachstellen), Nr. 7 (Cyberhygiene), Nr. 6 (Wirksamkeit/Nachweis).
- `iam`: Nr. 9 (Zugriffskontrolle), Nr. 10 (MFA-Grundlage), Nr. 7 (Cyberhygiene).
- `logging_monitoring`: Nr. 2 (Vorfallerkennung), Nr. 1, Nr. 6 (Auswertbarkeit/Wirksamkeit).
- `backup`: Nr. 3 (Aufrechterhaltung des Betriebs, Wiederherstellung).
- `encryption`: Nr. 8 (Kryptografie/Verschlüsselung), Nr. 10 (gesicherte Kommunikation).
- Krypto (Nr. 8) erfolgt systemweit über Krypto-Policies (Kapitel 9); unter Ubuntu über crypto-config bzw. Drop-ins.
