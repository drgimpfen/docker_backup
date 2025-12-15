# Docker Stack Backup Script

## Description

This Python script automates the backup process for multiple Docker Compose stacks. It implements a **sequential Stop-Archive-Start cycle** for each stack individually, which guarantees data consistency while minimizing overall service downtime. Archives are saved as **uncompressed .tar files** as part of a strategy specifically optimized for external deduplication tools like BorgBackup.

---

## Prerequisites

The only requirement is that the following commands are available on the system executing the script: `docker compose` (or `docker-compose`, depending on the system), `tar`, `find`, and `df`.

---

## Features

* **Static Stack Configuration:** Stacks to be backed up are defined in a simple list (`STACKS = [...]`) within the script. **This list must be maintained manually.**
* **Zero-Downtime Strategy (Per-Stack):** The script executes the backup cycle (stop, archive, start) for **one stack at a time**. This limits the downtime to a single service for the shortest possible duration, maximizing the overall availability of your environment.
* **Deduplication-Optimized Archives:** Creates uncompressed .tar archives, intentionally avoiding internal compression, which is ideal for subsequent processing by tools like BorgBackup.
* **Local Retention:** Automatically deletes backups older than the configured value (default 28 days) and calculates the total disk space freed.
* **Detailed Reporting:** Sends a comprehensive email report with status, a list of created archives, storage usage, and retention cleanup details.

---

## Rationale for Uncompressed .tar Archives

The script specifically uses uncompressed .tar archives (instead of .tar.gz, .tar.tar.bz2, or similar) because this format is crucial for maximizing the efficiency of external deduplication tools like **BorgBackup**.

* **Deduplication Efficiency:** Tools like Borg perform best when operating on raw, uncompressed data blocks. If compressed archives were used, a minor change inside a container volume would cause the entire compressed archive file to change significantly, defeating block-level deduplication.
* **Performance:** Avoiding internal compression during the archive creation phase saves CPU cycles, allowing external tools to handle the compression/deduplication in a highly optimized manner.

---

## Configuration

### 1. Stack List

You must manually maintain the list of stack names at the beginning of the script:

    STACKS = ["stack1", "stack2", "stack3", "etc..."] 

### 2. Script Variables

Adjust the following variables within the script (`docker-backup.py`) to match your environment.

#### Paths and Retention
| Variable | Description | Default Value |
| :--- | :--- | :--- |
| `STACKS` | **Critical:** List of Docker Compose stack directories to backup (must match directory names). | `["stack1", ...]` |
| `BASE_DIR` | Base directory containing most of your Docker Compose stacks. **Required for path construction.** | `/opt/stacks` |
| `EXTRA_STACK_PATH` | Path to a single stack located outside the `BASE_DIR` (optional exception, e.g., Dockge). | `/opt/dockge` |
| `BACKUP_DIR` | Destination directory for the backup archives. | `/var/backups/docker` |
| `DAILY_RETENTION_DAYS` | Number of days to keep local backups before deletion. | `28` |
| `LOG_FILE` | Path to the output log file. | `/var/log/docker-backup.log` |

#### Email Notification
| Variable | Description | Default Value |
| :--- | :--- | :--- |
| `SMTP_SERVER` / `SMTP_PORT` | SMTP server address and port. | `mailserver / 587` |
| `SMTP_USER` / `SMTP_PASS` | Credentials for the SMTP server. | `username / password` |
| `SENDER_EMAIL` / `RECEIVER_EMAIL` | Sender and recipient email addresses. | `sender@domain.com` |
| `SUBJECT_TAG` | Prefix for the email subject line. | `[TAG]` |

---

## Usage

1.  **Permissions:** Ensure the script is executable and has the necessary permissions (e.g., membership in the `docker` group or appropriate `sudo` rights) to access Docker and all relevant directories.
    ```bash
    chmod +x docker-backup.py
    ```

2.  **Set up Cron Job:** Schedule the script to run daily (or as needed) using a cron job.
    ```bash
    # Example: Run daily at 02:00 AM
    0 2 * * * /usr/bin/env python3 /path/to/your/docker-backup.py
    ```
