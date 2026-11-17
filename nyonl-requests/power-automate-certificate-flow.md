# CNE certificate flow (Power Automate, standard connectors only)

When someone submits the CNE evaluation in Microsoft Forms, this flow:
1. reads their answers,
2. fills their name into a certificate,
3. turns it into a PDF,
4. emails it to them,
5. saves a copy and logs it for your records.

Every step uses **standard** connectors, the ones included with basic Power Automate:
Microsoft Forms, OneDrive for Business, SharePoint, Office 365 Outlook and Excel Online (Business).

> **Why not a Word template?** The "Populate a Microsoft Word template" action is a **premium** action,
> so this flow builds the certificate as a simple web page (HTML) instead. OneDrive's free
> **Convert file** action then turns it into a PDF.

Set it up once, then copy it for each new program (see **Reusing for the next program** at the end).

---

## Before you start (about 15 minutes)

### 1. Build the evaluation form in Microsoft Forms
In the form settings, set **Who can fill out this form** to **Anyone can respond**, because attendees
are outside NYONL. Because it is open to anyone, the form must ask for name and email itself.

Include these questions. The exact wording is up to you, but keep them **Required**:

| Question | Type | Why the flow needs it |
|---|---|---|
| Full name and credentials as they should appear on your certificate (e.g. Jane Smith, MSN, RN) | Text | Printed on the certificate |
| Email address to send your certificate to | Text, with **Restrictions → Email** | Where the certificate goes |
| I attended the full program and completed this evaluation | Choice: **Yes** / **No** | Only "Yes" gets a certificate |
| *Your evaluation questions* | Any | Not used by the flow |

Tip: the NYONL Requests dashboard can draft the evaluation questions. Use **Ask Claude** on the task.

### 2. Make three places to keep files
- **OneDrive:** a folder called `CNE Certificates/temp`, used for working files the flow deletes.
- **SharePoint** (the site where you already keep certificates): a folder for this program,
  for example `CNE Certificates/2026-12 Leadership Program`.
- **Excel log:** an Excel workbook in OneDrive or SharePoint with a **table** (select the headers →
  Insert → Table) with these columns:
  `Date sent | Name | Email | Program | Contact hours | Certificate number`

### 3. Have these program details ready
- Program title
- Program date
- Contact hours awarded
- **Your accreditation statement.** Copy the exact wording NYONL is required to print on
  certificates. Don't retype it from memory.
- A short program code for certificate numbers, for example `LP2612`

---

## Build the flow

Go to **make.powerautomate.com → + Create → Automated cloud flow**.
Name it `CNE certificate – [Program title]`, choose the trigger
**When a new response is submitted (Microsoft Forms)**, and click **Create**.

### Step 1: Trigger: *When a new response is submitted* (Microsoft Forms)
- **Form Id:** pick your evaluation form.

### Step 2: *Get response details* (Microsoft Forms)
- **Form Id:** the same form.
- **Response Id:** pick **Response Id** from the trigger's dynamic content.

> If Power Automate wraps this step in an **Apply to each** box, that's normal. Put all the
> following steps inside that box.

### Step 3: Program details: four *Initialize variable* actions
Add **Initialize variable** four times (type **String** for all):

| Name | Value |
|---|---|
| `ProgramTitle` | e.g. *Leading Through Change: Nurse Leader Workshop* |
| `ProgramDate` | e.g. *December 4, 2026* |
| `ContactHours` | e.g. *3.0* |
| `AccreditationStatement` | paste NYONL's exact required wording |

> Power Automate only allows **Initialize variable** at the top level, not inside an **Apply to each**.
> If Step 2 created an Apply to each, move these four actions **above** it (drag them, or add them
> right after the trigger).

### Step 4: *Condition*: did they attest?
- Left box: the dynamic content for your **"I attended the full program…"** question.
- Middle: **is equal to**
- Right box: `Yes`

Put Steps 5–11 in the **True** branch. Leave **False** empty, or add an Outlook
**Send an email (V2)** to yourself saying someone answered "No".

### Step 5: *Compose*: the certificate
- **Inputs:** paste the contents of [`certificate-template.html`](certificate-template.html).
- Then replace each `{{…}}` placeholder: delete the placeholder text, and with the cursor there,
  insert the value from the dynamic content / variables list:

| Placeholder | Insert |
|---|---|
| `{{AttendeeName}}` | your **full name** question (from Get response details) |
| `{{ProgramTitle}}` | variable `ProgramTitle` |
| `{{ProgramDate}}` | variable `ProgramDate` |
| `{{ContactHours}}` | variable `ContactHours` |
| `{{AccreditationStatement}}` | variable `AccreditationStatement` |
| `{{CertificateNumber}}` | type your program code, a dash, then **Response Id** (e.g. `LP2612-` + Response Id) |
| `{{IssueDate}}` | **Expression** tab: `formatDateTime(utcNow(), 'MMMM d, yyyy')` |

### Step 6: *Create file* (OneDrive for Business): the working copy
- **Folder Path:** `/CNE Certificates/temp`
- **File Name:** `cert-` + **Response Id** + `.html`
- **File Content:** **Outputs** of Compose (Step 5)

### Step 7: *Convert file* (OneDrive for Business): make the PDF
- **File:** **Id** from Create file (Step 6)
- **Target type:** `PDF`

### Step 8: *Create file* (SharePoint): keep a copy for your records
- **Site Address:** your SharePoint site
- **Folder Path:** this program's certificate folder
- **File Name:** `Certificate - ` + **full name** answer + ` - ` + **Response Id** + `.pdf`
- **File Content:** **File content** from Convert file (Step 7)

### Step 9: *Send an email (V2)* (Office 365 Outlook): send it
- **To:** your **email address** question
- **Subject:** `Your CNE certificate: ` + variable `ProgramTitle`
- **Body** (edit to taste):
  > Thank you for attending [ProgramTitle] and completing the evaluation.
  > Your certificate for [ContactHours] contact hours is attached. Please keep it for your records.
  > New York Organization for Nursing Leadership
- **Attachments** (click **Show all** / advanced parameters):
  - **Attachments Name:** `NYONL CNE Certificate.pdf`
  - **Attachments Content:** **File content** from Convert file (Step 7)

> To send from **admin@nyonl.org** instead of your own account, use
> **Send an email from a shared mailbox (V2)** instead (also standard). You need "Send As"
> permission on that mailbox in Microsoft 365.

### Step 10: *Add a row into a table* (Excel Online (Business)): log it
Pick your log workbook and table, then fill the columns:
`Date sent` = expression `utcNow()`, `Name` = name answer, `Email` = email answer,
`Program` = `ProgramTitle`, `Contact hours` = `ContactHours`,
`Certificate number` = program code + **Response Id**.

### Step 11: *Delete file* (OneDrive for Business): tidy up
- **File:** **Id** from Create file (Step 6)

Click **Save**.

---

## Get told if it fails
1. Below Step 11 (still in the True branch), add another **Send an email (V2)** to yourself.
   Subject: `CNE certificate failed for` + the name answer.
2. On that new action open **… → Configure run after** (in the new designer: **Settings → Run after**).
3. Untick **is successful**, and tick **has failed** and **is skipped**. (When an earlier step fails,
   the steps after it are skipped, so ticking both catches a failure anywhere above.)

---

## Test before you share the form
1. Submit the form yourself with your own email and **Yes**.
2. In Power Automate, open the flow → **28-day run history** and check every step has a green tick.
3. Open the PDF in your email:
   - Is the name, program, date and hours right?
   - Does the layout fit on one page? The converter ignores some styling. If the page breaks
     badly, shorten the text or remove the border in the template.
4. Check that the SharePoint copy and the Excel log row are there.
5. Submit once with **No** and confirm no certificate is sent.

---

## Reusing for the next program
1. Make a copy of the evaluation form (Forms → **…** → **Duplicate**) and update it.
2. In Power Automate open this flow → **… → Save As**, and name the copy for the new program.
3. In the copy, change: the **Form Id** in Steps 1 and 2 (re-select the question answers in
   Steps 4, 5, 8, 9 and 10 afterwards, since the new form's questions have new IDs), the four
   variables in Step 3, the program code, and the SharePoint folder in Step 8.
4. **Turn the copy on** and run the test above.

---

## Things to know
- **Basic plan limits:** standard flows on Microsoft 365 plans have a daily action limit. A program
  of a few hundred attendees is normally well within it, but check **Power Platform admin center →
  Capacity** if you run very large events.
- **Typos in names are printed as-is.** Attendees type their own name, and Forms answers can't be
  edited afterwards. To correct one, submit the form yourself with the corrected name and the
  attendee's email address: the flow sends them a new certificate and logs it.
- **Record keeping:** the SharePoint copies and the Excel log are your record of who received a
  certificate. Keep them for as long as NYONL's accreditation requires.
