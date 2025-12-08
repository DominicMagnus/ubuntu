# MANUAL: Send regular Lynis warnings/suggestions to Gmail from Ubuntu using msmtp + user cron (no sudo in script)
# PURPOSE:
# - Configure Gmail SMTP with App Password
# - Use native Linux mail tools (mailx + sendmail via msmtp-mta)
# - Run the reporting script as a regular user
# - Ensure the user can read /var/log/lynis-report.dat using ACL
# - Keep a "live" run log so you can see each cron execution

# =============================================================================
# 1) INSTALL REQUIRED PACKAGES (INCLUDING ACL TOOLS)
# =============================================================================
sudo apt update
sudo apt install -y msmtp msmtp-mta bsd-mailx ca-certificates acl

# Optional verification that ACL tools exist
which setfacl
which getfacl

# =============================================================================
# 2) GOOGLE ACCOUNT PREREQUISITES
# =============================================================================
# In your Google account:
# - Enable 2-Step Verification (2FA)
# - Create an App Password for Mail
# - You will use that 16-character App Password below

# =============================================================================
# 3) CREATE USER msmtp CONFIG
# =============================================================================
# Create or edit the user config (NOT root)
nano ~/.msmtprc

# Paste this block and replace placeholders:
# ----
# defaults
# auth           on
# tls            on
# tls_starttls   on
# tls_trust_file /etc/ssl/certs/ca-certificates.crt
# logfile        ~/.msmtp.log
# account gmail
# host smtp.gmail.com
# port 587
# from YOUR_GMAIL_ADDRESS@gmail.com
# user YOUR_GMAIL_ADDRESS@gmail.com
# password YOUR_16_CHAR_APP_PASSWORD
# account default : gmail
# ----

# Secure permissions
chmod 600 ~/.msmtprc

# =============================================================================
# 4) USER-LEVEL SMTP TESTS (NO SUDO)
# =============================================================================
# These should be executed as your normal user
echo "test from $(hostname)" | mail -s "msmtp test" YOUR_GMAIL_ADDRESS@gmail.com
echo "test" | msmtp -a gmail YOUR_GMAIL_ADDRESS@gmail.com

# If you see:
# - 534-5.7.9 Application-specific password required
# You are using a normal password instead of an App Password

# =============================================================================
# 5) ACL: ALLOW YOUR USER TO READ THE Lynis REPORT
# =============================================================================
# The report file is usually owned by root, so without ACL your user may get Permission denied
# Replace YOUR_LINUX_USER with your actual username (e.g., alex)

sudo setfacl -m u:YOUR_LINUX_USER:r /var/log/lynis-report.dat

# Verify access without sudo
grep -E "warning|suggestion" /var/log/lynis-report.dat | head -n 20

# NOTE:
# If Lynis recreates the file and permissions reset, re-run the setfacl command above

# =============================================================================
# 6) CREATE USER SCRIPT DIRECTORY
# =============================================================================
mkdir -p ~/bin

# =============================================================================
# 7) CREATE THE NO-SUDO REPORT SCRIPT (WITH ALWAYS-ON LOGGING)
# =============================================================================
nano ~/bin/send-lynis-report.sh

# Paste this script and replace placeholders:
# ----
# #!/usr/bin/env bash
# # Send Lynis warnings/suggestions by email via user's msmtp config
# # Logs status messages so you can confirm cron runs even when nothing is sent
# set -euo pipefail
# RECIPIENT="YOUR_GMAIL_ADDRESS@gmail.com"
# REPORT_FILE="/var/log/lynis-report.dat"
# LOG_FILE="/home/YOUR_LINUX_USER/send-lynis-report.log"
# HOST="$(hostname -f 2>/dev/null || hostname)"
# NOW_ISO="$(date -Iseconds)"
# log() { echo "[$NOW_ISO] [$HOST] $*" >> "$LOG_FILE"; }
# log "Run started"
# if [[ ! -f "$REPORT_FILE" ]]; then
#   log "Report file not found: $REPORT_FILE. Exiting."
#   exit 0
# fi
# CONTENT="$(grep -E "warning|suggestion" "$REPORT_FILE" || true)"
# if [[ -z "$CONTENT" ]]; then
#   log "No warnings/suggestions found. Exiting."
#   exit 0
# fi
# {
#   echo "Host: $HOST"
#   echo "Time: $NOW_ISO"
#   echo "Source: $REPORT_FILE"
#   echo
#   echo "$CONTENT"
# } | mail -s "Lynis warnings on ${HOST} ${NOW_ISO}" "$RECIPIENT"
# log "Email sent to $RECIPIENT"
# ----

# Make executable
chmod 755 ~/bin/send-lynis-report.sh

# =============================================================================
# 8) MANUAL TEST (NO SUDO)
# =============================================================================
~/bin/send-lynis-report.sh

# Check script run log
tail -n 200 ~/send-lynis-report.log

# Optional: check msmtp transport log
tail -n 200 ~/.msmtp.log

# =============================================================================
# 9) ADD USER CRON (NO ROOT)
# =============================================================================
crontab -e

# Add one schedule line (replace YOUR_LINUX_USER in the path):

# Daily at 08:00
# 0 8 * * * /home/YOUR_LINUX_USER/bin/send-lynis-report.sh

# Every Monday at 09:00
# 0 9 * * 1 /home/YOUR_LINUX_USER/bin/send-lynis-report.sh

# Verify user cron
crontab -l

# =============================================================================
# 10) QUICK TROUBLESHOOTING
# =============================================================================
# Confirm you are not running as root
whoami
# Ensure msmtp config exists for the user
ls -l ~/.msmtprc
# Confirm sendmail wrapper is present
which sendmail
ls -l "$(which sendmail)"
# Confirm Lynis report is readable without sudo
grep -E "warning|suggestion" /var/log/lynis-report.dat | head -n 20
# If Permission denied returns, re-apply ACL
sudo setfacl -m u:YOUR_LINUX_USER:r /var/log/lynis-report.dat
