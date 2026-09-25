#!/bin/bash

set -Eeuo pipefail

# =========================================================
# MYSQL AUTO RESTORE TEST
# =========================================================

BASE_DIR="/opt/mysql-backup"
CONFIG_DIR="$BASE_DIR/config"
LOG_DIR="$BASE_DIR/logs"
TMP_DIR="$BASE_DIR/tmp/restore"

ENV_FILE="$CONFIG_DIR/backup.env"
DATABASE_FILE="$CONFIG_DIR/databases.conf"

MYSQL_CONTAINER="mysql-namtech"

RCLONE_CONFIG="$CONFIG_DIR/rclone.conf"
RCLONE_REMOTE="gdrive:Server-Backup/server01/mysql"

# =========================================================
# CHECK ROOT
# =========================================================

if [[ "$EUID" -ne 0 ]]; then
    echo "ERROR: Script harus dijalankan sebagai root."
    echo "Gunakan:"
    echo "sudo $0"
    exit 1
fi

# =========================================================
# CHECK CONFIG
# =========================================================

if [[ ! -f "$ENV_FILE" ]]; then
    echo "ERROR: File tidak ditemukan:"
    echo "$ENV_FILE"
    exit 1
fi

if [[ ! -f "$DATABASE_FILE" ]]; then
    echo "ERROR: File tidak ditemukan:"
    echo "$DATABASE_FILE"
    exit 1
fi

if [[ ! -f "$RCLONE_CONFIG" ]]; then
    echo "ERROR: File rclone tidak ditemukan:"
    echo "$RCLONE_CONFIG"
    exit 1
fi

source "$ENV_FILE"

# =========================================================
# DIRECTORY
# =========================================================

mkdir -p "$LOG_DIR"
mkdir -p "$TMP_DIR"

DATE=$(date '+%Y-%m-%d_%H-%M-%S')
DATE_DISPLAY=$(date '+%d-%m-%Y %H:%M:%S WIB')

LOG_FILE="$LOG_DIR/restore_$DATE.log"

START_TIME=$(date +%s)

exec > >(tee -a "$LOG_FILE") 2>&1

# =========================================================
# VARIABLES
# =========================================================

SUCCESS_COUNT=0
FAILED_COUNT=0

RESULT=""

LATEST_TIMESTAMP=""

# =========================================================
# DISCORD FUNCTION
# =========================================================

send_discord() {

    local MESSAGE="$1"

    if [[ -z "${DISCORD_WEBHOOK:-}" ]]; then
        echo "WARNING: DISCORD_WEBHOOK tidak ditemukan."
        return 0
    fi

    if ! command -v curl >/dev/null 2>&1; then
        echo "WARNING: curl tidak ditemukan."
        return 0
    fi

    if ! command -v jq >/dev/null 2>&1; then
        echo "WARNING: jq tidak ditemukan."
        return 0
    fi

    curl -sS \
        -H "Content-Type: application/json" \
        -X POST \
        -d "$(jq -n \
            --arg content "$MESSAGE" \
            '{content: $content}')" \
        "$DISCORD_WEBHOOK" \
        >/dev/null 2>&1 || true
}

# =========================================================
# DATABASE LIST
# =========================================================

DATABASES=()

while IFS= read -r DB || [[ -n "$DB" ]]; do

    [[ -z "$DB" ]] && continue

    if [[ "$DB" =~ ^[[:space:]]*# ]]; then
        continue
    fi

    DB="$(echo "$DB" | xargs)"

    [[ -z "$DB" ]] && continue

    DATABASES+=("$DB")

done < "$DATABASE_FILE"

if [[ "${#DATABASES[@]}" -eq 0 ]]; then
    echo "ERROR: Tidak ada database di:"
    echo "$DATABASE_FILE"
    exit 1
fi

# =========================================================
# RESTORE DATABASE NAME
# =========================================================

get_restore_db() {

    local DB="$1"

    echo "${DB}_restore"
}

# =========================================================
# CLEANUP
# =========================================================

cleanup() {

    echo ""
    echo "Cleaning temporary files..."

    rm -rf "$TMP_DIR"
}

trap cleanup EXIT

# =========================================================
# START
# =========================================================

echo "=========================================="
echo "MYSQL AUTO RESTORE TEST"
echo "=========================================="
echo "Server     : $(hostname)"
echo "Container  : $MYSQL_CONTAINER"
echo "Date       : $DATE_DISPLAY"
echo "Remote     : $RCLONE_REMOTE"
echo "=========================================="

# =========================================================
# CHECK DEPENDENCIES
# =========================================================

echo ""
echo "CHECK DEPENDENCIES"

if ! command -v rclone >/dev/null 2>&1; then
    echo "ERROR: rclone tidak ditemukan."
    exit 1
fi

if ! command -v docker >/dev/null 2>&1; then
    echo "ERROR: docker tidak ditemukan."
    exit 1
fi

if ! command -v gzip >/dev/null 2>&1; then
    echo "ERROR: gzip tidak ditemukan."
    exit 1
fi

if ! command -v curl >/dev/null 2>&1; then
    echo "ERROR: curl tidak ditemukan."
    exit 1
fi

if ! command -v jq >/dev/null 2>&1; then
    echo "ERROR: jq tidak ditemukan."
    exit 1
fi

echo "Dependencies OK."

# =========================================================
# CHECK MYSQL CONTAINER
# =========================================================

echo ""
echo "CHECK MYSQL CONTAINER"

if ! docker inspect "$MYSQL_CONTAINER" >/dev/null 2>&1; then

    echo "ERROR: Container $MYSQL_CONTAINER tidak ditemukan."

    exit 1
fi

CONTAINER_STATUS=$(
    docker inspect \
        --format '{{.State.Status}}' \
        "$MYSQL_CONTAINER"
)

echo "Container status: $CONTAINER_STATUS"

if [[ "$CONTAINER_STATUS" != "running" ]]; then

    echo "ERROR: Container $MYSQL_CONTAINER tidak running."

    exit 1
fi

# =========================================================
# CHECK MYSQL CONNECTION
# =========================================================

echo ""
echo "CHECK MYSQL CONNECTION"

if ! docker exec \
    -e MYSQL_PWD="$MYSQL_PASSWORD" \
    "$MYSQL_CONTAINER" \
    mysql \
    -u"$MYSQL_USER" \
    -e "SELECT 1;" \
    >/dev/null; then

    echo "ERROR: Tidak bisa koneksi ke MySQL."

    exit 1
fi

echo "MySQL connection OK."

# =========================================================
# CHECK MYSQL PRIVILEGES
# =========================================================

echo ""
echo "CHECK MYSQL PRIVILEGES"

GRANTS=$(
    docker exec \
        -e MYSQL_PWD="$MYSQL_PASSWORD" \
        "$MYSQL_CONTAINER" \
        mysql \
        -u"$MYSQL_USER" \
        -N \
        -e "SHOW GRANTS;"
)

echo "$GRANTS"

if ! grep -Eiq 'ALL PRIVILEGES|CREATE' <<< "$GRANTS"; then

    echo ""
    echo "ERROR:"
    echo "User $MYSQL_USER tidak memiliki privilege CREATE."
    echo ""

    exit 1
fi

echo "MySQL privileges OK."

# =========================================================
# CHECK GOOGLE DRIVE
# =========================================================

echo ""
echo "=========================================="
echo "CHECK GOOGLE DRIVE"
echo "=========================================="

REMOTE_FILES=$(
    rclone \
        --config "$RCLONE_CONFIG" \
        lsf "$RCLONE_REMOTE"
)

if [[ -z "$REMOTE_FILES" ]]; then

    echo "ERROR: Tidak ada backup di Google Drive."

    exit 1
fi

echo "Backup ditemukan."

# =========================================================
# FIND LATEST COMPLETE BACKUP
# =========================================================

echo ""
echo "MENCARI BACKUP TERBARU YANG LENGKAP"

while IFS= read -r FILE; do

    if [[ "$FILE" =~ ^namtech_db_([0-9]{4}-[0-9]{2}-[0-9]{2}_[0-9]{2}-[0-9]{2}-[0-9]{2})\.sql\.gz$ ]]; then

        CANDIDATE="${BASH_REMATCH[1]}"

        COMPLETE=true

        for DB in "${DATABASES[@]}"; do

            EXPECTED="${DB}_${CANDIDATE}.sql.gz"

            if ! grep -Fxq "$EXPECTED" <<< "$REMOTE_FILES"; then

                COMPLETE=false

                break

            fi

        done

        if [[ "$COMPLETE" == true ]]; then

            if [[ -z "$LATEST_TIMESTAMP" ]] || \
               [[ "$CANDIDATE" > "$LATEST_TIMESTAMP" ]]; then

                LATEST_TIMESTAMP="$CANDIDATE"

            fi

        fi

    fi

done <<< "$REMOTE_FILES"

if [[ -z "$LATEST_TIMESTAMP" ]]; then

    echo ""
    echo "ERROR: Tidak ditemukan backup lengkap."
    echo ""

    echo "Database yang dicari:"

    for DB in "${DATABASES[@]}"; do
        echo "- $DB"
    done

    exit 1
fi

echo ""
echo "Backup terpilih:"
echo "$LATEST_TIMESTAMP"

# =========================================================
# DOWNLOAD BACKUP
# =========================================================

echo ""
echo "=========================================="
echo "DOWNLOAD BACKUP"
echo "=========================================="

for DB in "${DATABASES[@]}"; do

    FILE="${DB}_${LATEST_TIMESTAMP}.sql.gz"

    echo ""
    echo "Download:"
    echo "$FILE"

    rclone \
        --config "$RCLONE_CONFIG" \
        copyto \
        "$RCLONE_REMOTE/$FILE" \
        "$TMP_DIR/$FILE"

    if [[ ! -f "$TMP_DIR/$FILE" ]]; then

        echo "ERROR: Download gagal."

        exit 1

    fi

    # =====================================================
    # VALIDATE GZIP
    # =====================================================

    if ! gzip -t "$TMP_DIR/$FILE"; then

        echo "ERROR: File backup corrupt:"
        echo "$FILE"

        exit 1

    fi

    echo "Download OK."

done

# =========================================================
# RESTORE DATABASES
# =========================================================

echo ""
echo "=========================================="
echo "RESTORE DATABASE"
echo "=========================================="

for DB in "${DATABASES[@]}"; do

    RESTORE_DB=$(get_restore_db "$DB")

    FILE="${DB}_${LATEST_TIMESTAMP}.sql.gz"

    echo ""
    echo "------------------------------------------"
    echo "Source : $DB"
    echo "Target : $RESTORE_DB"
    echo "File   : $FILE"
    echo "------------------------------------------"

    # =====================================================
    # DROP OLD RESTORE DATABASE
    # =====================================================

    echo "Drop database lama..."

    docker exec \
        -e MYSQL_PWD="$MYSQL_PASSWORD" \
        "$MYSQL_CONTAINER" \
        mysql \
        -u"$MYSQL_USER" \
        -e "DROP DATABASE IF EXISTS \`$RESTORE_DB\`;"

    # =====================================================
    # CREATE RESTORE DATABASE
    # =====================================================

    echo "Create database..."

    docker exec \
        -e MYSQL_PWD="$MYSQL_PASSWORD" \
        "$MYSQL_CONTAINER" \
        mysql \
        -u"$MYSQL_USER" \
        -e "CREATE DATABASE \`$RESTORE_DB\`;"

    # =====================================================
    # RESTORE
    # =====================================================

    echo "Restore sedang berjalan..."

    if gzip -dc "$TMP_DIR/$FILE" | \
        sed "s/^USE \`$DB\`;/USE \`$RESTORE_DB\`;/" | \
        docker exec -i \
        -e MYSQL_PWD="$MYSQL_PASSWORD" \
        "$MYSQL_CONTAINER" \
        mysql \
        -u"$MYSQL_USER" \
        "$RESTORE_DB"; then

        echo "Restore SUCCESS."

    else

        echo "Restore FAILED."

        FAILED_COUNT=$((FAILED_COUNT + 1))

        RESULT="${RESULT}
❌ $DB
   Restore gagal
"

        continue

    fi

    # =====================================================
    # VALIDATION TABLE
    # =====================================================

    TABLE_COUNT=$(
        docker exec \
            -e MYSQL_PWD="$MYSQL_PASSWORD" \
            "$MYSQL_CONTAINER" \
            mysql \
            -u"$MYSQL_USER" \
            -N \
            -e "
                SELECT COUNT(*)
                FROM information_schema.tables
                WHERE table_schema='$RESTORE_DB';
            "
    )

    TABLE_COUNT="$(echo "$TABLE_COUNT" | xargs)"

    echo "Table count : $TABLE_COUNT"

    if [[ "$TABLE_COUNT" -eq 0 ]]; then

        echo "ERROR: Database kosong."

        FAILED_COUNT=$((FAILED_COUNT + 1))

        RESULT="${RESULT}
❌ $DB
   Restore berhasil tetapi database kosong
"

        continue

    fi

    # =====================================================
    # VALIDATION SIZE
    # =====================================================

    DB_SIZE=$(
        docker exec \
            -e MYSQL_PWD="$MYSQL_PASSWORD" \
            "$MYSQL_CONTAINER" \
            mysql \
            -u"$MYSQL_USER" \
            -N \
            -e "
                SELECT ROUND(
                    COALESCE(
                        SUM(data_length + index_length),
                        0
                    ) / 1024 / 1024,
                    2
                )
                FROM information_schema.tables
                WHERE table_schema='$RESTORE_DB';
            "
    )

    DB_SIZE="${DB_SIZE:-0}"
    DB_SIZE="$(echo "$DB_SIZE" | xargs)"

    echo "Database size : ${DB_SIZE} MB"

    # =====================================================
    # SUCCESS
    # =====================================================

    SUCCESS_COUNT=$((SUCCESS_COUNT + 1))

    RESULT="${RESULT}
✅ $DB → $RESTORE_DB
   Tables : $TABLE_COUNT
   Size   : ${DB_SIZE} MB
"

done

# =========================================================
# FINAL RESULT
# =========================================================

END_TIME=$(date +%s)

DURATION=$((END_TIME - START_TIME))

echo ""
echo "=========================================="
echo "RESTORE TEST FINISHED"
echo "=========================================="

echo "Backup   : $LATEST_TIMESTAMP"
echo "Success  : $SUCCESS_COUNT"
echo "Failed   : $FAILED_COUNT"
echo "Duration : ${DURATION}s"

echo ""
echo "RESULT:"
echo "$RESULT"

echo "=========================================="

# =========================================================
# DISCORD SUCCESS
# =========================================================

if [[ "$FAILED_COUNT" -eq 0 ]]; then

    echo ""
    echo "✅ RESTORE TEST SUCCESS"

    DISCORD_MESSAGE="\
\`\`\`text
🟢 DATABASE RESTORE TEST BERHASIL

🖥️ Server
$(hostname)

📦 Docker Container
$MYSQL_CONTAINER

📅 Waktu
$DATE_DISPLAY

📊 HASIL RESTORE TEST
$RESULT
━━━━━━━━━━━━━━━━━━━━
✅ Berhasil : $SUCCESS_COUNT database
❌ Gagal    : $FAILED_COUNT database

☁️ Backup Source
Google Drive

📦 Backup
$LATEST_TIMESTAMP

🧪 Validasi
✅ File backup valid
✅ Restore berhasil
✅ Database tidak kosong

⏱️ Durasi
$DURATION detik
\`\`\`"

    send_discord "$DISCORD_MESSAGE"

    exit 0

fi

# =========================================================
# DISCORD FAILED
# =========================================================

echo ""
echo "❌ RESTORE TEST FAILED"

DISCORD_MESSAGE="\
\`\`\`text
🔴 DATABASE RESTORE TEST GAGAL

🖥️ Server
$(hostname)

📦 Docker Container
$MYSQL_CONTAINER

📅 Waktu
$DATE_DISPLAY

📊 HASIL RESTORE TEST
$RESULT
━━━━━━━━━━━━━━━━━━━━
✅ Berhasil : $SUCCESS_COUNT database
❌ Gagal    : $FAILED_COUNT database

☁️ Backup Source
Google Drive

📦 Backup
${LATEST_TIMESTAMP:-Tidak ditemukan}

🧪 Validasi
❌ Restore test gagal

⏱️ Durasi
$DURATION detik
\`\`\`"

send_discord "$DISCORD_MESSAGE"

exit 1
