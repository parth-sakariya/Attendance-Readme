# Temple Attendance — Flowcharts & Module Diagrams

Visual reference for **every role, every login, and every functionality**.
All diagrams are [Mermaid](https://mermaid.js.org) — they render on GitHub and in most
Markdown viewers (VS Code: install "Markdown Preview Mermaid Support").

## Contents
1. [System & hierarchy](#1-system--hierarchy)
2. [Roles & scope](#2-roles--scope)
3. [Login flows (all 6 logins)](#3-login-flows)
4. [Routing & guards](#4-routing--guards)
5. [Role → module access](#5-role--module-access)
6. [Account provisioning](#6-account-provisioning)
7. [Attendance — mark, lock, correction](#7-attendance--mark-lock-correction)
8. [QR scan attendance](#8-qr-scan-attendance)
9. [Student Check Register (SCR)](#9-student-check-register-scr)
10. [Notifications + FCM push](#10-notifications--fcm-push)
11. [P.A. / transfers](#11-pa--transfers)
12. [Events](#12-events)
13. [Shibir](#13-shibir)
14. [Import people (CSV)](#14-import-people-csv)
15. [Reports & export](#15-reports--export)
16. [Parent & student portals](#16-parent--student-portals)
17. [Firestore data model](#17-firestore-data-model)

---

## 1. System & hierarchy

```mermaid
flowchart TD
  SA(["Super Admin<br/>(global)"]) --> MT["Main Temple<br/>tenant root"]
  MT --> ST["Sub Temple<br/>branch"]
  ST --> SB["Sabha<br/>grade 1/2/3"]
  SB --> SG["Sabha Group<br/>section 1A/1B"]
  SG --> PE["Person<br/>member / student"]
  PE -.-> PR["Parent"]

  MTA(["Main Temple Admin"]) -.manages.-> MT
  STA(["Sub Temple Admin"]) -.manages.-> ST
  MEN(["Mentor"]) -.assigned to.-> SG
```

Every record stores denormalized ancestor IDs (`templeId / subTempleId / sabhaId /
sabhaGroupId`) so the tree stays shallow and queries/rules stay cheap.

---

## 2. Roles & scope

```mermaid
flowchart LR
  subgraph Roles
    SA["super_admin"]
    MTA["sub_admin<br/>(Main Temple Admin)"]
    STA["sub_temple_admin"]
    MEN["mentor"]
    STU["student"]
    PAR["parent"]
  end

  SA --> S1["ALL Main Temples"]
  MTA --> S2["One whole Main Temple<br/>(all its sub temples)"]
  STA --> S3["One Sub Temple"]
  MEN --> S4["Assigned Sabha Groups only"]
  STU --> S5["Own attendance + profile"]
  PAR --> S6["Own children only"]
```

| Role | value | Signs in with | Scope |
|---|---|---|---|
| Super Admin | `super_admin` | email + password (seeded) | all temples |
| Main Temple Admin | `sub_admin` | email + password | one Main Temple |
| Sub Temple Admin | `sub_temple_admin` | email + password | one Sub Temple |
| Mentor | `mentor` | email + password | assigned Sabha Groups |
| Student | `student` | Unique ID + password | own data |
| Parent | `parent` | mobile + password | own children |

---

## 3. Login flows

### 3.1 Which login screen
```mermaid
flowchart TD
  start(["App opens"]) --> splash["Splash: restoreSession()"]
  splash --> has{Valid session?}
  has -- yes --> route["homeRouteFor(role)"]
  has -- no --> needSA{Any super_admin exists?}
  needSA -- no --> setup["Setup: create first Super Admin"]
  needSA -- yes --> login["Staff/Student Login screen"]
  login --> plink["'Are you a parent?' link"]
  plink --> plogin["Parent Login screen"]
  route --> home["role shell"]
```

### 3.2 Staff / Admin / Mentor (email + password)
```mermaid
sequenceDiagram
  actor U as Staff / Admin / Mentor
  participant L as Login (staff mode)
  participant A as AuthRepository
  participant FA as Firebase Auth
  participant US as users/{uid}
  U->>L: email + password
  L->>A: login(email, pw)
  A->>FA: signInWithEmailAndPassword
  FA-->>A: uid
  A->>US: get profile
  US-->>A: role, templeId, subTempleId, isActive, assignedSabhaGroupIds
  alt inactive / no profile
    A-->>U: error, signed out
  else ok
    A->>A: register FCM token
    A->>A: homeRouteFor(role)
    A-->>U: temple shell (Manage / Attendance)
  end
```

### 3.3 Student (Unique ID + password)
```mermaid
sequenceDiagram
  actor S as Student
  participant L as Login (student mode)
  participant A as AuthRepository
  participant DIR as student_directory/{uniqueId}
  participant FA as Firebase Auth
  participant US as users/{uid}
  S->>L: Unique ID + password
  L->>A: loginWithUniqueId(id, pw)
  A->>DIR: lookup uniqueId
  DIR-->>A: authEmail (hidden synthesized)
  A->>FA: signInWithEmailAndPassword(authEmail, pw)
  FA-->>A: uid
  A->>US: resolve role == student
  US-->>A: AppUser (personId)
  A-->>S: Student portal
```

### 3.4 Parent (mobile + password)
```mermaid
sequenceDiagram
  actor P as Parent
  participant L as Parent Login
  participant PA as ParentAuthRepository
  participant DIR as parent_directory/{phone}
  participant FA as Firebase Auth
  participant US as users/{uid}
  P->>L: mobile number + password
  L->>PA: loginPhonePassword(phone, pw)
  PA->>DIR: lookup normalized phone
  DIR-->>PA: authEmail
  PA->>FA: signInWithEmailAndPassword(authEmail, pw)
  FA-->>PA: uid
  PA->>US: _resolve role == parent
  US-->>PA: AppUser (parentId, templeId)
  PA->>PA: register FCM token
  PA-->>P: Parent Portal
```

### 3.5 First-run Super Admin
```mermaid
flowchart LR
  seed["Seed out-of-band:<br/>Firebase Auth user +<br/>users/{uid} role=super_admin"] --> signin["Sign in (email+pw)"]
  signin --> work["Create Main Temples → admins →<br/>sub temples → sabhas → groups →<br/>mentors → people / parents"]
```

---

## 4. Routing & guards

```mermaid
flowchart TD
  nav(["Navigate to route"]) --> mw{Middleware}
  mw --> ag["AuthGuard:<br/>logged in?"]
  mw --> pg["PermissionGuard(permission):<br/>session.can(permission)?"]
  mw --> gg["GuestGuard:<br/>redirect if already logged in"]
  mw --> parg["ParentGuard:<br/>role == parent?"]
  ag -- no --> login["/login"]
  pg -- no --> home["/home"]
  parg -- no --> login
  ag -- yes --> page["Page builds<br/>(Binding injects controller)"]
  pg -- yes --> page
```

`homeRouteFor(role)`: parent → `/parent/home`, student → `/student/home`, everyone else → `/home`.

---

## 5. Role → module access

```mermaid
flowchart LR
  SUPER["Super Admin"] --> M1["Main Temples → enter workspace"]
  MTA["Main Temple Admin"] --> HUB["Manage hub"]
  STA["Sub Temple Admin"] --> HUB2["Manage hub (scoped)"]
  MEN["Mentor"] --> TABS["Attendance · Scan · Check · Report"]
  STU["Student"] --> SP["My attendance + My QR"]
  PAR["Parent"] --> PP["Dashboard · Attendance · Alerts · Profile"]

  HUB --> mods["Sub Temples*, Admins*, Activity Log*,<br/>Sabhas, Sabha Groups, Mentors, People, Parents,<br/>Mark/Scan Attendance, Check Register, Reports,<br/>Correction Requests, P.A./Transfers,<br/>Custom Fields, Events, Shibir, Notifications, Import"]
  HUB2 --> mods
  M1 --> mods
```
`*` = Main-Temple-admin / Super-admin only (Sub Temple Admin does not see Sub Temples, Admins, Activity Log).

### Permission matrix
| Capability (Permission) | super | sub_admin | sub_temple_admin | mentor | student/parent |
|---|:--:|:--:|:--:|:--:|:--:|
| manageTemples | ✅ | — | — | — | — |
| manageAdmins / manageSubTemples | ✅ | ✅ | — | — | — |
| manageSabhas / manageMentors / managePeople / manageParents | ✅ | ✅ | ✅ | people only | — |
| markAttendance / editAttendance | ✅ | ✅ | ✅ | ✅ | — |
| manageSchedule (lock) / reviewAttendanceRequests | ✅ | ✅ | ✅ | — | — |
| runChecker | ✅ | ✅ | ✅ | ✅ | — |
| managePA / transferPeople | ✅ | ✅ | ✅ | — | — |
| manageEvents / manageShibir / manageFieldConfig | ✅ | ✅ | ✅ | — | — |
| sendParentNotifications | ✅ | ✅ | ✅ | ✅ | — |
| generateReports / importData | ✅ | ✅ | ✅ | reports | — |
| viewActivityLog / viewAnalytics | ✅ | ✅ | — | — | — |
| viewOwnAttendance | — | — | — | — | ✅ |

---

## 6. Account provisioning

How an admin creates logins without being signed out (secondary Firebase app).

```mermaid
sequenceDiagram
  actor AD as Admin (signed in)
  participant FORM as Create form
  participant REPO as Repository
  participant AC as AccountCreator (secondary app)
  participant FA as Firebase Auth
  participant FS as Firestore
  AD->>FORM: name + email/mobile/uniqueId + password
  FORM->>REPO: create(...)
  REPO->>AC: createAccount(authEmail, pw)
  AC->>FA: createUserWithEmailAndPassword (secondary app)
  FA-->>AC: new uid (admin stays logged in)
  REPO->>FS: users/{uid} = {role, templeId, ...}
  REPO->>FS: directory entry (student_directory / parent_directory)
  REPO-->>AD: success
```
Applies to: **Admins**, **Mentors**, **People** (optional login), **Parents**.

---

## 7. Attendance — mark, lock, correction

```mermaid
flowchart TD
  open(["Mark Attendance"]) --> pick["Pick Sabha Group + date"]
  pick --> roster["Load roster + existing marks + lock state"]
  roster --> locked{Day locked?}

  locked -- "no" --> edit["Toggle Present / Absent"]
  edit --> save["Save (idempotent:<br/>groupId_date_personId)"]
  save --> notify{"Notify parents?"}
  notify -- yes --> push["Notification doc → FCM"]
  notify -- no --> done(["Saved"])

  locked -- "yes (admin)" --> unlock["Edit or Unlock"]
  locked -- "yes (mentor)" --> req["Request correction<br/>(status + reason)"]
  req --> pending["attendanceRequests: pending"]
  pending --> review["Admin: Correction Requests"]
  review --> approve{Approve?}
  approve -- yes --> rewrite["Rewrite attendance record<br/>+ mark approved (atomic)"]
  approve -- no --> reject["Mark rejected"]
```

Admins lock/unlock via the lock icon; the lock is also **enforced in Firestore rules**
(`dayLocked()` blocks mentor writes; admins bypass).

---

## 8. QR scan attendance

```mermaid
sequenceDiagram
  actor M as Mentor
  participant SC as Scan screen (camera)
  participant C as ScanAttendanceController
  participant AT as attendance
  participant N as notifications
  participant CF as Cloud Function
  participant FCM as FCM
  M->>SC: pick group, point camera
  SC->>C: onScan(rawValue)
  C->>C: parse KKATTEND:templeId:personId
  alt invalid / wrong temple / not in group / already scanned
    C-->>M: warning, ignore
  else valid
    C->>AT: saveBatch present (idempotent)
    AT-->>C: ok → add to "scanned" list
    opt notifyOnScan && has parent
      C->>N: create notification (PRESENT)
      N->>CF: onCreate trigger
      CF->>FCM: push to parent tokens
      FCM-->>M: parent's device notified
    end
  end
```

Each student carries a QR (`KKATTEND:<templeId>:<personId>`) shown from **People → Show QR**
or the **student's own portal**.

---

## 9. Student Check Register (SCR)

```mermaid
flowchart TD
  open(["Check Register"]) --> pick["Pick group + date"]
  pick --> tap["Tap 'Check in' for a person"]
  tap --> time["Record check-in time (HH:mm)"]
  time --> cmp{"Time ≤ Sub Temple cutoff?"}
  cmp -- yes --> full["Slot = FULL"]
  cmp -- "no / no cutoff set → FULL" --> half["Slot = HALF"]
  full --> save["Save SCR record"]
  half --> save
  save --> auto["Auto-mark attendance PRESENT (atomic)"]
  auto --> ovr["Mentor may override Full/Half"]
```

---

## 10. Notifications + FCM push

### 10.1 Compose (in-app)
```mermaid
flowchart LR
  comp["Send Notification screen"] --> aud{Audience}
  aud -- "All parents" --> a1["all parents in temple"]
  aud -- "Group parents" --> a2["parents of one Sabha Group"]
  a1 --> fan["createForParents:<br/>1 notification doc per parent"]
  a2 --> fan
  fan --> bell["Parent Portal → Alerts (in-app)"]
  fan --> trig["triggers Cloud Function"]
```

### 10.2 Push delivery (server)
```mermaid
sequenceDiagram
  participant N as notifications/{id} (create)
  participant CF as pushOnNotification (Cloud Function)
  participant US as users where parentId == doc.parentId
  participant FCM as Firebase Cloud Messaging
  actor D as Parent device
  N->>CF: onDocumentCreated trigger
  CF->>US: query recipient's fcmTokens
  US-->>CF: token list
  CF->>FCM: sendEachForMulticast(title, body)
  FCM-->>D: push (tray if closed / banner if open)
  CF->>US: prune invalid tokens
```
Tokens are saved by `PushService.registerFor(uid)` on every login / session restore.
Requires the function deployed (Blaze). In-app Alerts work without it.

---

## 11. P.A. / transfers

```mermaid
flowchart TD
  open(["P.A. / Transfers"]) --> pick["Pick a Sabha Group"]
  pick --> scan["Scan attendance per person"]
  scan --> calc["Compute absent streak & present streak"]
  calc --> sug{Suggestion}
  sug -- "regular & absent ≥ threshold" --> toPA["Suggest → P.A. group"]
  sug -- "in P.A. & present ≥ threshold" --> toReg["Suggest → regular group"]
  sug -- "otherwise" --> none["No suggestion"]
  toPA --> move["Admin picks target + reason → Move"]
  toReg --> move
  move --> apply["Update person group + P.A. flag<br/>+ write transfer log (atomic)"]
```
Thresholds are configured per Sub Temple (`paAbsentThreshold` / `paPromoteThreshold`).

---

## 12. Events

```mermaid
flowchart LR
  ev["Events list"] --> create["Create event<br/>(date, location, access PIN)"]
  ev --> att["Open event → Mark attendance"]
  att --> grp["Pick Sabha Group"]
  grp --> chk["Check 'attended' per person"]
  chk --> saveE["Save eventAttendance<br/>(eventId_personId)"]
  saveE --> rep["Counts / attended summary"]
```

---

## 13. Shibir

```mermaid
flowchart TD
  sh["Shibir list"] --> build["Create shibir + form builder<br/>(custom fields, fee)"]
  build --> reg["Registrations → 'Register'"]
  reg --> fill["Fill name, mobile, custom fields"]
  fill --> fee["Fee paid? + amount"]
  fill --> file["Attach file → Firebase Storage"]
  fee --> submit["Save submission"]
  file --> submit
  submit --> repS["Summary: count, paid, ₹ collected<br/>+ Excel export"]
```

---

## 14. Import people (CSV)

```mermaid
flowchart LR
  imp["Import People"] --> grp["Pick target Sabha Group"]
  grp --> file["Pick CSV (Name, Surname, Father, Mobile)"]
  file --> parse["Parse + preview rows"]
  parse --> bulk["bulkCreate:<br/>sequential Unique IDs + batch write"]
  bulk --> done(["Imported N people"])
```

---

## 15. Reports & export

```mermaid
flowchart TD
  rep(["Attendance Report"]) --> pick["Pick Sabha Group + month + layout"]
  pick --> gen["Query attendance for month range"]
  gen --> lay{Layout}
  lay -- Summary --> s1["Per person: Present / Absent / %"]
  lay -- Datewise --> s2["Per person × day matrix (P / A / –)"]
  s1 --> out["Summary chips + table"]
  s2 --> out
  out --> exp{Export}
  exp -- PDF --> pdf["printing (share/print/download)"]
  exp -- Excel --> xls["excel + share_plus"]
```

---

## 16. Parent & student portals

### Parent portal
```mermaid
flowchart LR
  PH["Parent Home"] --> D["Dashboard:<br/>children + avg attendance"]
  PH --> A["Attendance:<br/>calendar + trend + history (per child)"]
  PH --> AL["Alerts:<br/>notifications (bell)"]
  PH --> PR["Profile:<br/>edit, change password, sign out"]
  D --> CD["Child details (read-only)"]
```
Strictly isolated: queries only `people where parentId == own` and that child's attendance.

### Student portal
```mermaid
flowchart LR
  SH["Student Home"] --> own["Own attendance (calendar/history)"]
  SH --> qr["My QR (present to mentor)"]
  SH --> out["Sign out"]
```

---

## 17. Firestore data model

```mermaid
erDiagram
  USERS ||--o| PEOPLE : "personId (student)"
  USERS ||--o| PARENTS : "parentId (parent)"
  MAIN_TEMPLE ||--o{ SUB_TEMPLE : has
  SUB_TEMPLE ||--o{ SABHA : has
  SABHA ||--o{ SABHA_GROUP : has
  SABHA_GROUP ||--o{ PERSON : has
  PARENT ||--o{ PERSON : "children (parentId)"
  SABHA_GROUP ||--o{ ATTENDANCE : "per day"
  PERSON ||--o{ ATTENDANCE : marked
  PERSON ||--o{ SCR_RECORD : "check-in"
  PERSON ||--o{ TRANSFER : "moved"
  SHIBIR ||--o{ SHIBIR_SUBMISSION : has

  USERS {
    string uid PK
    string role
    string templeId
    string subTempleId
    string personId
    string parentId
    list assignedSabhaGroupIds
    list fcmTokens
    bool isActive
  }
  PERSON {
    string personId PK
    string templeId
    string sabhaGroupId
    string uniqueId
    string parentId
    map fields
  }
  ATTENDANCE {
    string id PK "groupId_date_personId"
    string personId
    string dateKey
    string status
  }
```

Flat lookup collections (read before sign-in): `parent_directory/{phone}`,
`student_directory/{uniqueId}`. Everything else lives under `mainTemples/{templeId}/…`.
