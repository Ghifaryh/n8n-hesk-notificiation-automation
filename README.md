# 🧾 HESK → n8n Ticket Automation

This project connects a HESK Helpdesk database (MariaDB) to n8n for automated ticket monitoring and notifications (e.g., Telegram / Discord).
It checks for new or open tickets, sends messages, and marks them as sent to prevent duplicates.

## ⚙️ System Overview

Stack:

- 🧩 n8n (workflow automation)

- 🗄️ MariaDB / MySQL (HESK ticket database)

- 🔔 Telegram / Discord (notification targets)

Purpose:

> Automatically detect new or unsent HESK tickets and notify admins or support teams.

## 🧠 Workflow Logic

**1. Scheduler Trigger**
   - Runs periodically (cron-based).
   - Default: every 1 minutes.
     ```cron
      0 */1 * * * 1-5
     ```
**2. MySQL Query**
   - Fetches tickets that are:
     - Still open (`closedat IS NULL`)
     - Or created today
     - And not yet sent (`custom50 IS NULL OR custom50 = ''`)
   - Includes helpful flags (`is_open`, `is_unreplied`, etc.)
     ```mysql
     SELECT
        t.id,
        t.trackid,
        t.name AS requester_name,
        t.email AS requester_email,
        t.subject,
        t.message,
        t.dt AS created_at,
        t.lastchange AS updated_at,
        u.name AS assigned_name,
        CASE CAST(t.priority AS UNSIGNED)
          WHEN 1 THEN 'Kritis'
          WHEN 2 THEN 'Tinggi'
          WHEN 3 THEN 'Menengah'
          ELSE CONCAT('Unknown (', t.priority, ')')
        END AS priority_name,
        CASE t.status
          WHEN 0 THEN 'Baru'
          WHEN 1 THEN 'Menunggu Balasan'
          WHEN 2 THEN 'Balasan Staff'
          WHEN 3 THEN 'Selesai'
          ELSE CONCAT('Unknown (', t.status, ')')
        END AS status_name,
        (t.closedat IS NULL) AS is_open,
        (t.custom50 IS NULL OR t.custom50 = '') AS is_unsent
      FROM hesk_tickets t
      LEFT JOIN hesk_users u ON u.id = t.owner
      WHERE
        (
          t.closedat IS NULL
          OR (t.dt >= CURDATE() AND t.dt < DATE_ADD(CURDATE(), INTERVAL 1 DAY))
        )
        AND (t.custom50 IS NULL OR t.custom50 = '')
      ORDER BY t.lastchange DESC, t.id DESC
      LIMIT 20;      
     ```
**3. Notification Node**
   - Sends formatted message:
      
        ```markdown
         ```ALERT
         🎫 Tiket Helpdesk
         
         🗓️ Tanggal: 2025-10-16 11:47:16
         🆔 ID Tiket: ET5-ZS6-ZJUZ
         🧾 Subjek: testing FINAL
         👤 Pembuat: Gip-test
         
         📩 Isi:
         TESTING INSTANT NOTIFICATION
         
         
         👨‍💻 Assigned: Admin
         ```
        ```
**4. Update Node**
  - Mark ticket as sent once notification suceeds:
    ```mysql
    UPDATE hesk_tickets
    SET custom50 = '1'
    WHERE id = {{ $json.id }};
    ```
**5. Error Handling**
  - One node to revert the update.

## Database Setup
**Create dedicated n8n user**
```mysql
CREATE USER 'n8nuser'@'%' IDENTIFIED BY 'yourpassword';
GRANT SELECT ON hesk.* TO 'n8nuser'@'%';
GRANT UPDATE (custom50) ON hesk.hesk_tickets TO 'n8nuser'@'%';
FLUSH PRIVILEGES;
```
>This limits n8n to read tickets and update only one column (`custom50`).

## Schedule Options
| Schedule                            | Expression           | Description                          |
|-------------------------------------|----------------------|--------------------------------------|
| Every 3 minute (weekdays)           | `0 */3 * * * 1-5`    | Default safe mode                    |
| Every minute (24/7)                 | `0 */1 * * * *`      | Faster polling **(Currently used)** |
| Every minute (08:00–17:00, Mon–Fri) | `0 */1 8-17 * * 1-5` | Office hours only                    |

## 🚀 Future Improvements
- Add custom49 for Telegram vs Discord tracking.
- Implement “update changed” detection (not just new tickets).
- Optionally move flag tracking to a separate lightweight table.
- Add retry queue for failed sends.
- Add webhook or trigger for instant push (instead of polling).

## 🧾 License / Notes
- This integration is for internal automation with HESK helpdesk.
- It does not modify core HESK logic except custom50 column usage.
- Tested with MariaDB 10.6+, n8n 1.70+.
