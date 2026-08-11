# NIS2 auf Linux-Systemen — Begleit-Playbooks

Ansible-Automatisierung zum eBook **„Vom Paragrafen zum Playbook: NIS2 auf RHEL umsetzen"**.
Dieses ZIP wächst kapitelweise. Aktuell enthalten:

- `roles/hardening` — Kapitel 4: System-Hardening (RHEL 9/10)
- `roles/patch_management` — Kapitel 5: Patch- & Schwachstellenmanagement (RHEL 9/10)
- `roles/iam` — Kapitel 6: Identity & Access Management (RHEL 9/10)
- `roles/logging_monitoring` — Kapitel 7: Logging, Monitoring & Vorfallerkennung (RHEL 9/10)
- `roles/backup` — Kapitel 8: Backup, Restore & Business Continuity (RHEL 9/10)
- `roles/encryption` — Kapitel 9: Verschlüsselung (Krypto-Policy, NBDE-Tooling) (RHEL 9/10)

## Voraussetzungen

- Ansible (`ansible-core` ≥ 2.15) auf dem Control-Node
- Collections: `ansible.posix`, `community.general`
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```
- Zielsysteme: RHEL 9 oder 10 (bzw. kompatible), SSH-Zugang, sudo/become

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

Tags: `packages`, `sshd`, `sysctl`, `selinux`, `services`, `firewalld`

- **SSH:** Drop-in wird vor dem Reload mit `sshd -t` validiert (kein Aussperren).
- **Firewall:** Standardzonenwechsel ist opt-in (Aussperr-Risiko); SSH wird stets zuerst freigegeben.
- **SELinux:** bleibt `enforcing`/`targeted`.

## Rolle: patch_management (Kapitel 5)

- Konfiguriert `dnf-automatic` (Standard: nur Sicherheitsupdates, Kernel ausgenommen)
  und aktiviert `dnf-automatic.timer`.
- Installiert `dnf-utils` für `needs-restarting`.
- Richtet einen wöchentlichen Schwachstellen-Scan ein (/var/log/compliance): auf RHEL 8/9 per OVAL, auf RHEL 10 als updateinfo-Report — Red Hat publiziert für RHEL 10 keine OVAL-Daten mehr (Nachfolger CSAF/VEX; auswertbar z. B. mit Grype/Trivy, siehe Kapitel 5.4/11).
- Die OVAL-URL wird automatisch aus der RHEL-Hauptversion abgeleitet.

Wichtige Variablen (siehe `roles/patch_management/defaults/main.yml`):
`patch_management_automatic_upgrade_type` (security|default),
`patch_management_automatic_excludepkgs`, `patch_management_automatic_timer`,
`patch_management_oval_scan_enabled`, `patch_management_oval_oncalendar`.

## Rolle: iam (Kapitel 6)

- Installiert SSSD, authselect und oddjob-mkhomedir.
- Aktiviert das authselect-Profil `sssd` mit `with-mkhomedir with-faillock with-sudo`.
- Konfiguriert Kontosperre (faillock) und Mindest-Passwortqualität (pwquality).
- Bringt ein mit `visudo` validiertes sudo-Drop-in für die Admin-Gruppe aus.
- **Nicht** enthalten: der Domänenbeitritt (`ipa-client-install` bzw. LDAP-Anbindung) — bewusst umgebungsspezifisch.
Hinweis: Mitglieder von ssh-admins (sudo) müssen zusätzlich in ssh-users (SSH-Zugang, Rolle hardening) sein — zwei getrennte Gruppen sind Absicht (Login ≠ Privilegien)

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

## Rolle: backup (Kapitel 8)

Tags: `backup`

- Installiert Restic (aus EPEL) und richtet Umgebungs- und Ausschlussdateien ein.
- Bringt ein Backup-Skript mit Wartungsschleife (backup → forget/prune → check) und einen systemd-Timer aus.
- Schreibt nach Erfolg eine Prometheus-Textfile-Metrik (`restic_backup_last_success_timestamp_seconds`) für die Überwachung aus Kapitel 7.
- **Geheimnisse per Ansible Vault setzen:** `backup_password` und ggf. `backup_env` (z. B. `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` für S3/B2).
- Für **Append-only-Repos** `backup_prune_enabled: false` setzen — `forget`/`prune` laufen dann auf dem gesicherten Backup-Host, nicht auf dem Client.
- EPEL wird (opt-in, backup_install_epel: true) mit importiertem GPG-Schlüssel eingerichtet — restic kommt aus EPEL. Das Repository einmalig manuell initialisieren (restic init); das Skript bricht andernfalls mit klarer Meldung ab.

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
- Krypto (Nr. 8) erfolgt systemweit über Krypto-Policies (Kapitel 9).
