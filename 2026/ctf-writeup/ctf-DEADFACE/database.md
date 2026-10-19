# Admitted Under Another Name

**Category:** Database
**Points:** 15
**Flag:** `deadface{AH-0044719}`

## The Goal
The challenge says DEADFACE dumped Aurora Health's patient registry. One high-profile patient checked in under a fake name, and staff flagged them as a VIP. I need to find their real MRN.

## My Process

First, I had to see what databases were available:

```sql
SHOW DATABASES;
```

<table>
  <tr>
    <td width="55%" valign="top">
      Since the challenge was about a patient registry, <code>aurora_ehr</code> (Electronic Health Records) was the obvious choice. I switched to it and listed the tables to confirm <code>patients</code> was there.<br><br>
      <code>USE aurora_ehr;</code><br>
      <code>SHOW TABLES;</code><br><br>
      Next, I wanted to see the column names for the <code>patients</code> table before guessing anything. I checked the schema:<br><br>
      <code>SELECT column_name FROM information_schema.columns<br>
      WHERE table_schema = 'aurora_ehr' AND table_name = 'patients';</code><br><br>
      The two columns that caught my eye were <code>alias</code> (the fake name) and <code>is_vip</code> (the staff flag). <br><br>
      I ran this query:<br><br>
      <code>SELECT patient_id, mrn, first_name, last_name, alias, is_vip<br>
      FROM patients<br>
      WHERE is_vip = 1;</code><br><br>
      The output gave me one row: Gregory Halloran, alias George Harmon, with MRN <code>AH-0044719</code>.
    </td>
    <td width="45%" valign="top">
      <img src="https://github.com/user-attachments/assets/c64bfd42-590b-42f7-a885-36abca9582bb" width="100%" />
    </td>
  </tr>
</table>
---

# Schedule of Concern

**Category:** Database
**Points:** 20
**Flag:** `deadface{9}`

## The Goal
The challenge wants a baseline for the pharmacy formulary: how many drugs are classified as DEA Schedule II (the most tightly regulated class).

## My Process

Since this was about the pharmacy formulary, I switched to the `aurora_pharmacy` database and checked the tables.

<table>
  <tr>
    <td width="55%" valign="top">
      <code>USE aurora_pharmacy;</code><br>
      <code>SHOW TABLES;</code><br><br>
      I saw a <code>drugs</code> table. Before trying to count anything, I wanted to see the exact column names so I didn't guess wrong. I described the table:<br><br>
      <code>DESCRIBE drugs;</code><br><br>
      I saw the column <code>dea_schedule</code>. But I didn't know what format the data was in (it could be '2', 'II', or 'C-II'). So I checked the unique values first:<br><br>
      <code>SELECT DISTINCT dea_schedule FROM drugs;</code><br><br>
      The output showed <code>II, III, IV, V, N</code>. So the value was exactly <code>'II'</code>. Now I could run the count:<br><br>
      <code>SELECT COUNT(*) FROM drugs WHERE dea_schedule = 'II';</code><br><br>
      The output was <code>9</code>.
    </td>
    <td width="45%" valign="top">
      <img src="https://github.com/user-attachments/assets/5ffc3870-7788-410d-afe8-e8ac4840e225" width="100%" />
    </td>
  </tr>
</table>
---

# The Attending

**Category:** Database
**Points:** 40
**Flag:** `deadface{mreyes}`

## The Goal
Find which provider attended Gregory Halloran's (the VIP patient) November 2025 admission.

## My Process

Since I already found the VIP patient in the first challenge, I knew his `patient_id` was `1007`. I needed to find his encounter in November 2025 first.

<table>
  <tr>
    <td width="55%" valign="top">
      I wrote a query to pull the encounter details for patient 1007 during that specific month:<br><br>
      <code>SELECT encounter_id, patient_id, attending_provider_id, admit_date<br>
      FROM aurora_ehr.encounters<br>
      WHERE patient_id = 1007<br>
      AND admit_date >= '2025-11-01'<br>
      AND admit_date < '2025-12-01';</code><br><br>
      The output showed an encounter on `2025-11-14`, and the <code>attending_provider_id</code> was `5`.<br><br>
      Now that I had the provider's ID, I queried the <code>providers</code> table to get their actual name and username:<br><br>
      <code>SELECT provider_id, first_name, last_name, username, specialty<br>
      FROM aurora_ehr.providers<br>
      WHERE provider_id = 5;</code><br><br>
      The output gave me: Marcus Reyes, username <code>mreyes</code>.
    </td>
    <td width="45%" valign="top">
      <img width="625" height="485" alt="image" src="https://github.com/user-attachments/assets/9a5e8709-838b-476d-83e7-bee0b5586007" />

    </td>
  </tr>
</table>
---

# Prying Eyes

**Category:** Database
**Points:** 55
**Flag:** `deadface{rborden}`

## The Goal
Find out which user accessed the VIP patient's chart the most. The VIP patient's MRN is `AH-0044719` (from the first challenge).

## My Process

I switched to the `aurora_audit` database because that's where access logs usually live. I listed the tables and saw `access_log` and `users`.

<table>
  <tr>
    <td width="55%" valign="top">
      I needed to join the two tables to get the username, and then filter specifically for the VIP's MRN, the `view` action, and the `chart` resource.<br><br>
      <code>SELECT u.username, COUNT(*) AS views<br>
      FROM access_log al<br>
      JOIN users u ON u.user_id = al.user_id<br>
      WHERE al.patient_mrn = 'AH-0044719'<br>
      AND al.action = 'view'<br>
      AND al.resource = 'chart'<br>
      GROUP BY u.username<br>
      ORDER BY views DESC;</code><br><br>
      The output showed <code>rborden</code> at the top with 13 views. The next highest was `mreyes_ehr` with only 2 views.
    </td>
    <td width="45%" valign="top">
      <img width="650" height="682" alt="image" src="https://github.com/user-attachments/assets/6a3eb981-042f-42e0-917e-c92827e39221" />
      
  </tr>
</table>
---

# The Original Reading

**Category:** Database
**Points:** 70
**Flag:** `deadface{2.3}`

## The Goal
Find the original value of a primary efficacy endpoint that was overwritten. The challenge mentions a `data_changes` audit table, but didn't say which database it was in.

## My Process

<table>
  <tr>
    <td width="55%" valign="top">
      Since I didn't know which database had the <code>data_changes</code> table, I searched <code>information_schema</code> across all databases to find it:<br><br>
      <code>SELECT table_schema, table_name FROM information_schema.tables<br>
      WHERE table_name = 'data_changes';</code><br><br>
      The output told me it was in <code>aurora_trials</code>. I used the fully qualified name to select all the rows so I could look at them:<br><br>
      <code>SELECT * FROM aurora_trials.data_changes;</code><br><br>
      There were 9 rows. Most of them were routine corrections to the <code>subjects</code> table. But row <code>500001</code> stood out. It changed the <code>primary_efficacy_delta</code> for <code>VELT-0137</code> from <code>2.3</code> to <code>6.0</code>. That was the malicious overwrite.
    </td>
    <td width="45%" valign="top">
      <img width="940" height="370" alt="image" src="https://github.com/user-attachments/assets/3427bbd4-cf3c-4188-b1f7-f89f820922ae" />

  </tr>
</table>
---

# Heavy Hand

**Category:** Database
**Points:** 100
**Flag:** `deadface{BR2938471}`

## The Goal
Find the prescriber who dispensed the highest total quantity of Schedule II medication between October 1 and December 31, 2025, and report their DEA number.

## My Process

The challenge mentioned a pharmacy database, so I switched to `aurora_pharmacy` and listed the tables. I saw `dispenses`, `drugs`, `prescribers`, and `prescriptions`. 

<table>
  <tr>
    <td width="55%" valign="top">
      Because I didn't know the exact column names, I ran <code>DESCRIBE</code> on the relevant tables to map out the schema.<br><br>
      <code>DESCRIBE drugs;</code><br>
      <code>DESCRIBE dispenses;</code><br>
      <code>DESCRIBE prescribers;</code><br>
      <code>DESCRIBE prescriptions;</code><br><br>
      I found the real columns:
      <ul>
        <li><code>drugs.dea_schedule</code></li>
        <li><code>dispenses.quantity_dispensed</code> and <code>dispensed_at</code></li>
        <li><code>prescribers.dea_number</code></li>
      </ul>
      "Rx" is just medical shorthand for prescription. So <code>rx_id</code> is the prescription ID that links the <code>dispenses</code> table (what was actually handed out) to the <code>prescriptions</code> table (what was written).
    </td>
    <td width="45%" valign="top">
      <img width="430" height="583" alt="image" src="https://github.com/user-attachments/assets/00f18b74-9554-41e3-b084-57acce67a519" />

  </tr>
</table>

<br>

Now that I knew the columns, I could write the JOIN. I needed to link `dispenses` to `prescriptions` (using `rx_id`), then to `prescribers` and `drugs`.

<table>
  <tr>
    <td width="55%" valign="top">
      <code>SELECT prescribers.dea_number, prescribers.first_name, prescribers.last_name, SUM(dispenses.quantity_dispensed) AS total<br>
      FROM dispenses<br>
      JOIN prescriptions ON prescriptions.rx_id = dispenses.rx_id<br>
      JOIN prescribers ON prescribers.prescriber_id = prescriptions.prescriber_id<br>
      JOIN drugs ON drugs.drug_id = prescriptions.drug_id<br>
      WHERE drugs.dea_schedule = 'II'<br>
      AND dispenses.dispensed_at BETWEEN '2025-10-01' AND '2025-12-31'<br>
      GROUP BY prescribers.dea_number, prescribers.first_name, prescribers.last_name<br>
      ORDER BY total DESC<br>
      LIMIT 5;</code><br><br>
      The output showed Marcus Reyes at the top with <code>5220</code>. The next closest person was Dorothy Hill with only <code>150</code>. That massive gap made it obvious he was the target.
    </td>
    <td width="45%" valign="top">
      <<img width="493" height="728" alt="image" src="https://github.com/user-attachments/assets/46a617bc-842c-46e5-9622-1e60318fe4da" />

  </tr>
</table>
---

# Upon the Throne

**Category:** Database
**Points:** 120
**Flag:** `deadface{REVENANT-7}`

## The Goal
Find the single DEADFACE handler codename that appears in **both** the audit `users` table and the EHR `providers` table. 

## My Process

I figured I needed to look at both sides of the house. First, I pulled the `handler_ref` column from the audit database:

<table>
  <tr>
    <td width="55%" valign="top">
      <code>SELECT username, handler_ref FROM aurora_audit.users;</code><br><br>
      This returned 66 rows. I could see a mix of <code>NULL</code> values and codenames like <code>EMISSARY-9</code>, <code>REGENT-16</code>, and <code>REVENANT-7</code>. <br><br>
      Next, I pulled the same column from the provider database:<br><br>
      <code>SELECT first_name, last_name, handler_ref FROM aurora_ehr.providers;</code><br><br>
      This returned 44 rows. I saw <code>NULL</code>s again, but also codenames like <code>ORACLE-4</code>, <code>PUPPETEER-7</code>, and <code>REVENANT-7</code>.<br><br>
      Since there were over 100 rows total, scrolling through them by eye to find the match was a bad idea. I needed to do an intersection.
    </td>
    <td width="45%" valign="top">
      <img width="489" height="991" alt="image" src="https://github.com/user-attachments/assets/8f12b9f1-5f4d-43a0-9ab2-a275611a450f" />

  </tr>
</table>

<br>

To find the exact overlap, I wrote a query to join the two tables on the `handler_ref` column:

<table>
  <tr>
    <td width="55%" valign="top">
      <code>SELECT DISTINCT u.handler_ref<br>
      FROM aurora_audit.users u<br>
      JOIN aurora_ehr.providers p ON p.handler_ref = u.handler_ref;</code><br><br>
      I used <code>DISTINCT</code> just in case multiple people shared the same handler. The JOIN naturally ignored all the <code>NULL</code> values on both sides.<br><br>
      The output was exactly one row: <code>REVENANT-7</code>. <br><br>
      I verified this by cross-referencing: on the audit side, it was Rebecca Borden (Health Information Clerk), and on the provider side, it was Marcus Reyes (Pain Management). This fit the challenge story perfectly: an insider clerk and a physician being run by the same handler.
    </td>
    <td width="45%" valign="top">
     <img width="572" height="844" alt="image" src="https://github.com/user-attachments/assets/61d4b85b-65ad-42db-87ba-6ac4c332b282" />

  </tr>
</table>
---

# After the Lock

**Category:** Database
**Points:** 150
**Flag:** `deadface{VELT-0137}`

## The Goal
Find the subject whose primary efficacy result was modified *after* the study's official data-lock date. That's a protocol violation and a sign of tampering.

## My Process

I switched to the `aurora_trials` database. I knew I needed to compare the `modified_at` timestamp on the results against the `data_lock_date` on the studies.

<table>
  <tr>
    <td width="55%" valign="top">
      Since the dates are in different tables, I had to do a 3-table JOIN: <code>results</code> connects to <code>subjects</code> via <code>subject_id</code>, and <code>subjects</code> connects to <code>studies</code> via <code>study_id</code>.<br><br>
      <code>SELECT sub.subject_code, r.value, r.modified_at, st.data_lock_date<br>
      FROM results r<br>
      JOIN subjects sub ON sub.subject_id = r.subject_id<br>
      JOIN studies st ON st.study_id = sub.study_id<br>
      WHERE r.measure = 'primary_efficacy_delta'<br>
      AND r.modified_at > st.data_lock_date;</code><br><br>
      The output gave me exactly one row: <code>VELT-0137</code>, with a value of <code>6.00</code>, modified on <code>2025-12-06</code>. The lock date was <code>2025-12-01</code>, meaning it was changed 5 days after the deadline.
    </td>
    <td width="45%" valign="top">
      <img width="654" height="392" alt="image" src="https://github.com/user-attachments/assets/ecb6e4ec-c27c-4b96-8d4c-fbe7b92bd4f0" />

  </tr>
</table>
---

# Too Good to be True

**Category:** Database
**Points:** 310
**Flag:** `deadface{VELT-0148}`

## The Goal
Find the subject whose primary efficacy result is more than 3 standard deviations above the study-wide mean. A result that high is statistically impossible, which means it was likely fabricated.

## My Process

I didn't want to write one huge, complicated query with a CTE for this. I decided to break the math down into smaller, simple steps so I could actually see the numbers.

<table>
<tr>
<td width="55%" valign="top">
<b>Step 1: Get the mean</b><br>
<code>SELECT AVG(value) FROM results WHERE measure = 'primary_efficacy_delta';</code><br>
Output: <code>2.965</code><br><br>
<b>Step 2: Get the standard deviation</b><br>
<code>SELECT STDDEV_POP(value) FROM results WHERE measure = 'primary_efficacy_delta';</code><br>
Output: <code>2.8066...</code><br><br>
<b>Step 3: Do the math on a calculator</b><br>
Mean + (3 * StdDev) = 2.965 + (3 * 2.8066) = <b>11.3851</b>.<br>
Anything above 11.3851 is our outlier.<br><br>
<b>Step 4: Find the value</b><br>
<code>SELECT subject_id, value FROM results<br>
WHERE measure = 'primary_efficacy_delta'<br>
AND value > 11.3851;</code><br>
Output showed one row: <code>subject_id 148</code> with a value of <code>22.00</code>.<br><br>
<b>Step 5: Get the subject code</b><br>
<code>SELECT subject_code FROM subjects WHERE subject_id = 148;</code><br>
Output: <code>VELT-0148</code>.
</td>
<td width="45%" valign="top">
<img src="https://github.com/user-attachments/assets/fe9978a8-3e21-4e67-96fc-efbd25e0b4bd" width="100%" />
</td>
</tr>
</table>
---

# Two Places at Once

**Category:** Database
**Points:** 450
**Flag:** `deadface{kbhatt}`

## The Goal
Find the account that accessed the system from two different workstations at the exact same timestamp. 

## My Process

I had to switch to `USE aurora_audit;`.

I needed to group the logs by user and timestamp, and then check if the same user hit multiple workstations. I used `GROUP_CONCAT` because I actually wanted to read the names of the workstations and the IPs to see how different they were.

<table>
  <tr>
    <td width="55%" valign="top">
      <code>SELECT u.username, al.accessed_at,<br>
      COUNT(DISTINCT al.workstation) AS distinct_workstations,<br>
      GROUP_CONCAT(DISTINCT al.workstation) AS workstations,<br>
      GROUP_CONCAT(DISTINCT al.ip_address) AS ips<br>
      FROM access_log al<br>
      JOIN users u ON u.user_id = al.user_id<br>
      GROUP BY u.username, al.accessed_at<br>
      HAVING COUNT(DISTINCT al.workstation) > 1;</code><br><br>
      This query returned 8 rows, and every single one was for the username <code>kbhatt</code>. The timestamps were all late at night, and the IP addresses were from completely different subnets (e.g., 10.20.3.x vs 10.20.4.x).<br><br>
      To confirm they were the only one doing this, I ran a subquery to rank the users by how many times it happened. <code>kbhatt</code> had a count of 8.
    </td>
    <td width="45%" valign="top">
      <img width="886" height="918" alt="image" src="https://github.com/user-attachments/assets/70496849-6997-4840-b94d-891d4e8f1ddd" />

  </tr>
</table>

<br>

Finally, I pulled the full log for `kbhatt` to see the pattern of what they were doing.

<table>
  <tr>
    <td width="55%" valign="top">
      <code>SELECT al.accessed_at, al.workstation, al.ip_address, al.action, al.resource<br>
      FROM access_log al<br>
      JOIN users u ON u.user_id = al.user_id<br>
      WHERE u.username = 'kbhatt'<br>
      ORDER BY al.accessed_at;</code><br><br>
      Looking at the full log, you can clearly see normal working-hour access at single workstations. But then there are those 8 specific late-night events (between 23:54 and 03:57) where they were hitting two workstations at the exact same second.<br><br>
      Even worse, some of these simultaneous sessions were performing <code>export</code> actions on charts and labs. That's a stolen session being used to exfiltrate data while the real person is still logged in elsewhere. Yikes.
    </td>
    <td width="45%" valign="top">
      <img width="867" height="849" alt="image" src="https://github.com/user-attachments/assets/b1ecbde2-a090-4f6d-aa0c-3a7ab81594d7" />

  </tr>
</table>
