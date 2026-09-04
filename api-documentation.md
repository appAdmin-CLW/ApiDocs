# EduWorkFlow API Documentation

## Overview

This application consists of multiple services:
1. **Microsoft Graph API** (`https://graph.microsoft.com/v1.0`) - OneDrive file storage
2. **EduWorkFlow Backend** (`config.json` `BackendUrl`) - Server health, user presence, policy helper
3. **AI Backend** (`config.json` `AiBackendUrl`) - RAG, document compare, workbook processing
4. **Analytics Backend** (`config.json` `AnalyticsBackendUrl`) - Event telemetry, ClickHouse queries, domain management
5. **Microsoft Copilot API** - AI-powered content generation

---

## 1. EduWorkFlow Backend (Common)

**Base URL:** `{BackendUrl}` (e.g., `https://marklochrie.co.uk/eduworkflow/backend`)
**Port:** 4201
**Database:** Redis

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Server health check - returns `{ message: "Its-a-me, Mario!" }` |
| `GET` | `/db-check` | Redis health check - returns `{ ok: boolean, dbOk: boolean, database: "redis" }` |
| `GET` | `/policyHelper/defaults` | Get saved policy helper default document IDs |
| `POST` | `/policyHelper/defaults` | Save policy helper default document IDs (global, SENCO only) |
| `POST` | `/userPresence` | Send user presence for a passport |
| `GET` | `/userPresence` | Get users currently viewing a passport |

### 1.1 Server Health Check

#### Root Health Check
- **Method:** `GET`
- **URL:** `{BackendUrl}/`
- **Response:** `{ message: "Its-a-me, Mario!" }`
- **Purpose:** Verify backend server is running

#### Database Check
- **Method:** `GET`
- **URL:** `{BackendUrl}/db-check`
- **Response:** `{ ok: boolean, dbOk: boolean, database: "redis" }`
- **Purpose:** Verify Redis connection is working

### 1.2 Policy Helper Default Documents

#### Get Default Documents
- **Method:** `GET`
- **URL:** `{BackendUrl}/policyHelper/defaults`
- **Response:** `string[]` (array of document IDs)
- **Note:** Global setting - applies to all users
- **Source:** `src/pages/policyHelper/policyHelper.tsx` - `fetchSavedDocs()`

#### Save Default Documents
- **Method:** `POST`
- **URL:** `{BackendUrl}/policyHelper/defaults`
- **Headers:** `Content-Type: application/json`
- **Body:** `{ docIds: string[] }`
- **Response:** `{ ok: boolean }`
- **Note:** Only SENCO users can modify. Saved in Redis under `policy_helper:default_docs`.
- **Source:** `src/pages/policyHelper/policyHelper.tsx` - `saveDocs()`, `clearSavedDocs()`

### 1.3 User Presence

#### Send Presence
- **Method:** `POST`
- **URL:** `{BackendUrl}/userPresence?passportId={passportId}&user={user}`
- **Response:** `{ ok: boolean }`
- **Note:** Sets 60-second TTL per user per passport
- **Source:** `src/services/userPresenceService.ts` - `sendPresence()`

#### Get Presence
- **Method:** `GET`
- **URL:** `{BackendUrl}/userPresence?passportId={passportId}`
- **Response:** `string[]` (array of user names currently viewing the passport)
- **Source:** `src/services/userPresenceService.ts` - `getPresence()`

---

## 2. AI Backend

**Base URL:** `{AiBackendUrl}` (e.g., `https://marklochrie.co.uk/eduworkflow/backend/ai`)
**Port:** 4202
**Database:** PostgreSQL + pgvector + Redis

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | AI backend health check |
| `GET` | `/server-check` | Simple server check |
| `GET` | `/db-check` | Full DB health check (PostgreSQL, pgvector, Redis) |
| `POST` | `/upload` | Upload document for RAG |
| `POST` | `/ask` | Ask question against uploaded documents |
| `POST` | `/compare` | Compare two documents |
| `POST` | `/process-workbook` | Process PDF workbook |
| `GET` | `/documents` | List user's uploaded documents |
| `POST` | `/demo-upload` | Upload document for demo |
| `POST` | `/demo-ask` | Ask question for demo |

### 2.1 Server Health Checks

#### AI Backend Health
- **Method:** `GET`
- **URL:** `{AiBackendUrl}/`
- **Response:** `{ message: "EduWorkflow AI BE is running" }`

#### Simple Server Check
- **Method:** `GET`
- **URL:** `{AiBackendUrl}/server-check`
- **Response:** `{ ok: true, service: "ai" }`

#### Full Database Check
- **Method:** `GET`
- **URL:** `{AiBackendUrl}/db-check`
- **Response:** Object with PostgreSQL, pgvector, and Redis status
- **Source:** `app/serverCheck.py`

### 2.2 RAG (Retrieval Augmented Generation)

#### Upload Document
- **Method:** `POST`
- **URL:** `{AiBackendUrl}/upload?user_id={userId}`
- **Headers:** `Content-Type: multipart/form-data`
- **Body:** `file` (PDF file)
- **Response:** `{ document_id: string }`
- **Source:** `app/api.py` - `upload_pdf()`

#### Ask Question
- **Method:** `POST`
- **URL:** `{AiBackendUrl}/ask`
- **Headers:** `Content-Type: application/json`, `Authorization: Bearer {token}`
- **Body:** `{ question: string, user_id: string, document_ids?: string[] }`
- **Response:** `{ question: string, answer: string, sources: Chunk[] }`
- **Source:** `app/api.py` - `ask()`

#### Compare Documents
- **Method:** `POST`
- **URL:** `{AiBackendUrl}/compare`
- **Headers:** `Content-Type: multipart/form-data`
- **Body:** `file1` and `file2` (two PDF files)
- **Response:** Comparison result object
- **Source:** `app/api.py` - `compare_files()`

#### List User Documents
- **Method:** `GET`
- **URL:** `{AiBackendUrl}/documents?user_id={userId}`
- **Response:** `Document[]` (array of user's uploaded documents)
- **Source:** `app/api.py` - `get_documents()`

### 2.3 Workbook Processing

#### Process Workbook
- **Method:** `POST`
- **URL:** `{AiBackendUrl}/process-workbook`
- **Headers:** `Content-Type: multipart/form-data`
- **Body:** `file` (PDF workbook)
- **Response:** `{ pages: PageData[] }`
- **Source:** `app/api.py` - `process_workbook()`

### 2.4 Demo Routes

#### Demo Upload
- **Method:** `POST`
- **URL:** `{AiBackendUrl}/demo-upload`
- **Headers:** `Content-Type: multipart/form-data`
- **Body:** `file` (document)
- **Response:** `{ document_id: string, filename: string }`
- **Source:** `app/api.py` - `demo_upload()`

#### Demo Ask
- **Method:** `POST`
- **URL:** `{AiBackendUrl}/demo-ask`
- **Headers:** `Content-Type: application/json`
- **Body:** `{ question: string, document_ids?: string[] }`
- **Response:** `{ question: string, answer: string, sources: Chunk[] }`
- **Source:** `app/api.py` - `demo_ask()`

---

## 3. Analytics Backend

**Base URL:** `{AnalyticsBackendUrl}` (e.g., `https://marklochrie.co.uk/eduworkflow/analytics`)
**Port:** 4200
**Database:** ClickHouse + SQLite

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/telemetry` | Track events (forwarded to OpenPanel → ClickHouse) |
| `GET` | `/query` | Execute raw SQL against ClickHouse |
| `GET` | `/domains` | List all registered domains |
| `POST` | `/domains` | Register a new domain |
| `POST` | `/domains/:domain/allow` | Allow a domain |
| `POST` | `/domains/:domain/disable` | Disable a domain |
| `DELETE` | `/domains/:domain` | Delete a domain |
| `GET` | `/consent-form` | Download consent form PDF |
| `POST` | `/consent-form` | Upload/update consent form PDF |

### 3.1 Telemetry

#### Track Event
- **Method:** `POST`
- **URL:** `{AnalyticsBackendUrl}/telemetry`
- **Headers:** `Content-Type: application/json`
- **Body:** `{ name: string, profileId: string, properties?: Record<string, any>, domain?: string }`
- **Response:** `{ success: true }`
- **Note:** Events are fire-and-forget to OpenPanel. Only tracks events from allowed domains.
- **Source:** `src/openpanel.js` - `router.post("/telemetry")`

### 3.2 ClickHouse Query

#### Execute SQL Query
- **Method:** `GET`
- **URL:** `{AnalyticsBackendUrl}/query?sql={encodedSQL}`
- **Response:** `any[]` (query results as JSON array)
- **Note:** Direct access to ClickHouse `openpanel` database. Used by analytics dashboard.
- **Source:** `src/query.js` - `router.get("/query")`

### 3.3 Domain Management

#### List Domains
- **Method:** `GET`
- **URL:** `{AnalyticsBackendUrl}/domains`
- **Response:** `Domain[]` (array of domain objects with `isAllowed`, `schoolName`, `address`, `email`, `enabledFeatures`)
- **Source:** `src/domains.js` - `router.get("/domains")`

#### Add Domain
- **Method:** `POST`
- **URL:** `{AnalyticsBackendUrl}/domains`
- **Headers:** `Content-Type: application/json`
- **Body:** `{ domain: string, schoolName?: string, address?: string, email?: string, enabledFeatures?: string[] }`
- **Response:** `{ domain: string, isAllowed: boolean, ... }`
- **Source:** `src/domains.js` - `router.post("/domains")`

#### Allow Domain
- **Method:** `POST`
- **URL:** `{AnalyticsBackendUrl}/domains/{domain}/allow`
- **Response:** Updated domain object
- **Source:** `src/domains.js` - `router.post("/domains/:domain/allow")`

#### Disable Domain
- **Method:** `POST`
- **URL:** `{AnalyticsBackendUrl}/domains/{domain}/disable`
- **Response:** Updated domain object
- **Source:** `src/domains.js` - `router.post("/domains/:domain/disable")`

#### Delete Domain
- **Method:** `DELETE`
- **URL:** `{AnalyticsBackendUrl}/domains/{domain}`
- **Response:** `{ domain: string }`
- **Source:** `src/domains.js` - `router.delete("/domains/:domain")`

### 3.4 Consent Form

#### Download Consent Form
- **Method:** `GET`
- **URL:** `{AnalyticsBackendUrl}/consent-form`
- **Response:** PDF file
- **Source:** `src/documents.js` - `router.get("/consent-form")`

#### Upload Consent Form
- **Method:** `POST`
- **URL:** `{AnalyticsBackendUrl}/consent-form`
- **Headers:** `Content-Type: application/pdf`
- **Body:** PDF file (up to 20MB)
- **Response:** `{ success: boolean, filename: string, size: number }`
- **Source:** `src/documents.js` - `router.post("/consent-form")`

---

## 4. Microsoft Graph API

All Graph API calls use the configured `OneDriveId` from `config.json` and authenticate with a Bearer token obtained via MSAL.

### 4.1 Reports

#### Create Reports Pupil Folder
- **Method:** `POST`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/Reports:/children`
- **Body:** `{ name: {pupilName}, folder: {} }`
- **Response:** OneDrive folder object with `id`
- **Source:** `src/services/reportsService.ts` - `createPupilFolder()`

#### Get All Pupils in Reports
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/Reports:/children`
- **Response:** OneDrive children array
- **Source:** `src/services/reportsService.ts` - `returnPupilsInReports()`

#### Delete Reports Pupil Folder
- **Method:** `DELETE`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{folderId}`
- **Source:** `src/services/reportsService.ts` - `deletePupilFolder()`

#### Rename Reports Pupil Folder
- **Method:** `PATCH`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{folderId}`
- **Body:** `{ name: {newName} }`
- **Source:** `src/services/reportsService.ts` - `editPupilName()`

#### Create Report File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{pupilFolderId}:/{year}.json:/content`
- **Body:** `{ report: { report } }`
- **Response:** OneDrive item with `id`
- **Source:** `src/services/reportsService.ts` - `createReport()`

#### Get Reports for Pupil (List Years)
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{pupilFolderId}/children`
- **Response:** Array of year names (e.g., `["2024", "2025"]`)
- **Source:** `src/services/reportsService.ts` - `getReportsForPupil()`

#### Save Report Data
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{pupilFolderId}:/{year}.json:/content`
- **Body:** `report` (full report JSON)
- **Source:** `src/services/reportsService.ts` - `saveReportData()`

#### Get Report Data
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{pupilFolderId}:/{year}.json:/content`
- **Response:** `report` (JSON)
- **Source:** `src/services/reportsService.ts` - `getReportData()`

#### Get Report Documents Folder Info
- **Step 1:** `GET` `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}/children?$select=id,name,parentReference`
- **Step 2:** Find "Report Documents" folder from children
- **Step 3:** `POST` `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{folderId}/createLink` with body `{ type: "view" }`
- **Response:** `{ folderId, driveId, folderUrl }`
- **Source:** `src/services/reportDocGenerationService.ts` - `getReportFolderInfo()`

### 4.2 Reports - User Saved Students

#### Get Saved Reports Pupils
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/me/drive/root:/eduWorkFlow/reportsPupils.json:/content`
- **Response:** `{ students: string[] }`
- **Source:** `src/services/userDataService.ts` - `getSavedReportsPupilNamesForUser()`

#### Add Reports Pupil to User File
- **Method:** `GET` (existing) + `PUT` (update)
- **URL:** `https://graph.microsoft.com/v1.0/me/drive/root:/eduWorkFlow/reportsPupils.json:/content`
- **Body:** `{ students: string[] }`
- **Source:** `src/services/userDataService.ts` - `addReportsPupilToUserFile()`

#### Remove Reports Pupil from User File
- **Method:** `GET` (existing) + `PUT` (update)
- **URL:** `https://graph.microsoft.com/v1.0/me/drive/root:/eduWorkFlow/reportsPupils.json:/content`
- **Body:** `{ students: string[] }`
- **Source:** `src/services/userDataService.ts` - `removeReportsPupilFromUserFile()`

### 4.3 Passport Data

#### Create/Update Passport File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PupilPassportStudents/{studentName}/{year}/{term}/{passportTitle}.json:/content`
- **Headers:** `Authorization: Bearer {authCode}`, `Content-Type: application/json`
- **Request Body:** `studentPassportData` (JSON)
- **Response:** OneDrive item object with `id` field
- **Source:** `src/services/passportService.ts` - `createPassportFolder()`

#### Get Passport Data by File ID
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{fileId}/content`
- **Headers:** `Authorization: Bearer {authCode}`
- **Response:** `studentPassportData` (JSON)
- **Source:** `src/services/passportService.ts` - `getPassportDataByFileId()`

#### Save/Update Passport Data
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{passportFileId}/content`
- **Headers:** `Authorization: Bearer {authCode}`, `Content-Type: application/json`
- **Request Body:** `studentPassportData` (JSON)
- **Source:** `src/services/passportService.ts` - `savePassportData()`

#### Get Passport File Metadata
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{passportFileId}`
- **Headers:** `Authorization: Bearer {authCode}`
- **Response:** OneDrive item metadata (createdDateTime, lastModifiedDateTime, createdBy, lastModifiedBy)
- **Source:** `src/services/passportService.ts` - `loadPassportFileMetadataByFileId()`

#### Delete Passport Folder
- **Method:** `DELETE`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}:/{year}/{term}:`
- **Headers:** `Authorization: Bearer {authCode}`
- **Source:** `src/services/passportService.ts` - `deletePassportFolder()`

#### Get Passport Documents Folder Info
- **Step 1:** `GET` `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}/children?$select=id,name,parentReference`
- **Step 2:** Find "Passport Documents" folder from children
- **Step 3:** `POST` `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{folderId}/createLink` with body `{ type: "view" }`
- **Response:** `{ folderId, driveId, folderUrl }`
- **Source:** `src/services/passportService.ts` - `getPassportFolderInfo()`

### 4.4 Application Data Files

#### Get Years List
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/ApplicationData/Years.json:/content`
- **Response:** `string[]` (e.g., `["Pre-School", "Reception", "Year 1", ...]`)
- **Source:** `src/services/fileService.ts` - `getYearList()`

#### Get Terms List
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/ApplicationData/Terms.json:/content`
- **Response:** `string[]` (e.g., `["Autumn", "Spring", "Summer"]`)
- **Source:** `src/services/fileService.ts` - `getTermsList()`

#### Get Needs List
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/ApplicationData/Needs.json:/content`
- **Response:** `string[]` (e.g., `["Communication and Interaction", "Cognition and Learning", ...]`)
- **Source:** `src/services/fileService.ts` - `getNeedsList()`

#### Get Passport Stages
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/ApplicationData/PassportStages.json:/content`
- **Response:** `string[]` (e.g., `["Draft", "Incomplete", "Awaiting Approval", ...]`)
- **Source:** `src/services/fileService.ts` - `getPassportStages()`

#### Create Years File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/ApplicationData/Years.json:/content`
- **Body:** `["Pre-School", "Reception", "Year 1", "Year 2", "Year 3", "Year 4", "Year 5", "Year 6"]`
- **Source:** `src/services/fileService.ts` - `createYearsFile()`

#### Create Terms File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/ApplicationData/Terms.json:/content`
- **Body:** `["Autumn", "Spring", "Summer"]`
- **Source:** `src/services/fileService.ts` - `createTermsFile()`

#### Create Passport Stages File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/ApplicationData/PassportStages.json:/content`
- **Body:** `["Draft", "Incomplete", "Awaiting Approval", "Completed Pending Evaluation", "Completed"]`
- **Source:** `src/services/fileService.ts` - `createPassportStagesFile()`

#### Create Needs File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/ApplicationData/Needs.json:/content`
- **Body:** `["Communication and Interaction", "Cognition and Learning", "Social, Emotional and Mental Health", "Sensory and Physical Needs"]`
- **Source:** `src/services/fileService.ts` - `createNeedsFile()`

### 4.5 Student Folders

#### Create Student Folder
- **Method:** `POST`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PupilPassportStudents:/children`
- **Body:** `{ name: {studentName}, folder: {} }`
- **Response:** OneDrive folder object with `id`
- **Source:** `src/services/studentFolderService.ts` - `createStudentFolder()`

#### Get All Students in Pupil Passport
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PupilPassportStudents:/children`
- **Response:** OneDrive children array
- **Source:** `src/services/studentFolderService.ts` - `returnStudentsInPP()`

#### Get Year Folders for Student
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}/children`
- **Source:** `src/services/studentFolderService.ts` - `getYearFoldersForStudent()`

#### Get Term Folders for Student
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{yearFolderId}/children`
- **Source:** `src/services/studentFolderService.ts` - `getTermFoldersForStudent()`

#### Get Term Folder Content
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{termFolderId}/children`
- **Source:** `src/services/studentFolderService.ts` - `getTermFolderContent()`

#### Delete Student Folder
- **Method:** `DELETE`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{folderId}`
- **Source:** `src/services/studentFolderService.ts` - `deleteStudentFolder()`

#### Rename Pupil Folder
- **Method:** `PATCH`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{folderId}`
- **Body:** `{ name: {newName} }`
- **Source:** `src/services/studentFolderService.ts` - `editpupilName()`

#### Ensure Database Structure
- Creates `eduWorkFlow` root folder if not exists
- Creates subfolders: `Archive`, `PolicyDocuments`, `PupilPassportStudents`, `ApplicationData`
- Creates JSON files: Years.json, Terms.json, PassportStages.json, Needs.json
- **Source:** `src/services/databaseService.ts` - `ensureDatabaseStructure()`

### 4.6 Logs

#### Create Logs Folder
- **Method:** `POST`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PupilPassportStudents/{studentName}/{year}/{term}:/children`
- **Body:** `{ name: "Logs", folder: {}, "@microsoft.graph.conflictBehavior": "replace" }`
- **Response:** Folder object with `id`
- **Source:** `src/services/logsService.ts` - `createLogsFolder()`

#### Create Folders for Each Need (under Logs)
- **Method:** `POST`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{logFolderId}/children`
- **Body:** `{ name: {needId}, folder: {}, "@microsoft.graph.conflictBehavior": "fail" }`
- **Source:** `src/services/logsService.ts` - `createFoldersForNeeds()`

#### Create/Update Log File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{logFolderId}:/{needFolderName}/{logDate}.json:/content`
- **Body:** `{ strategyEvaluation: [...], comments: string }`
- **Source:** `src/services/logsService.ts` - `createLogFile()`

#### Get Needs Folders (under Logs)
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{logFolderId}/children`
- **Response:** Array of folder names
- **Source:** `src/services/logsService.ts` - `getNeedsFolders()`

#### Get Log Files for a Need
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{logFolderId}:/{needFolderName}:/children`
- **Response:** Array of file names
- **Source:** `src/services/logsService.ts` - `getLogFilesForNeed()`

#### Get Content of a Log File
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{logFolderId}:/{needId}/{logFileName}:/content`
- **Response:** `{ comments: string, strategyEvaluation: [...] }`
- **Source:** `src/services/logsService.ts` - `getContentOfLogFile()`

### 4.7 Provisions

#### Create Provisions File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}:/provisions.json:/content`
- **Body:** `{}`
- **Source:** `src/services/provisionsService.ts` - `createProvisionsFile()`

#### Get Provisions File Content
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}/children/provisions.json/content`
- **Response:** `ProvisionsTable` (JSON)
- **Source:** `src/services/provisionsService.ts` - `getProvisionsFileContentByStudentFolderId()`

#### Update Provisions File Content
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}/children/provisions.json/content`
- **Body:** `ProvisionsTable` (JSON)
- **Source:** `src/services/provisionsService.ts` - `updateProvisionsFileContentByStudentFolderId()`

### 4.8 Evaluation Summary

#### Get Evaluation Summary File
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}:/evaluationSummary.json:/content`
- **Response:** `evaluationSummary[]` (JSON)
- **Source:** `src/services/fileService.ts` - `getEvaluationSummaryFileContentByStudentFolderId()`

#### Update Evaluation Summary File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}:/evaluationSummary.json:/content`
- **Body:** `evaluationSummary[]` (JSON)
- **Source:** `src/services/fileService.ts` - `updateEvaluationSummaryFileContentByStudentFolderId()`

#### Create Pupil Summary File
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}:/evaluationSummary.json:/content`
- **Body:** `{}`
- **Source:** `src/services/pupilSummaryService.ts` - `createPupilSummaryFile()`

### 4.9 User-Specific Saved Students

#### Get Saved Students
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/me/drive/root:/eduWorkFlow/savedStudents.json:/content`
- **Response:** `{ students: string[] }`
- **Source:** `src/services/fileService.ts` - `getSavedStudentNamesForUser()`

#### Add Student to User File
- **Method:** `GET` (existing) + `PUT` (update)
- **URL:** `https://graph.microsoft.com/v1.0/me/drive/root:/eduWorkFlow/savedStudents.json:/content`
- **Body:** `{ students: string[] }`
- **Source:** `src/services/fileService.ts` - `addStudentToUserFile()`

#### Remove Student from User File
- **Method:** `GET` (existing) + `PUT` (update)
- **URL:** `https://graph.microsoft.com/v1.0/me/drive/root:/eduWorkFlow/savedStudents.json:/content`
- **Body:** `{ students: string[] }`
- **Source:** `src/services/fileService.ts` - `removeStudentFromUserFile()`

#### Check/Create User File
- **Method:** `GET` (check) + `PUT` (create if missing)
- **URL:** `https://graph.microsoft.com/v1.0/me/drive/root:/eduWorkFlow/savedStudents.json:`
- **Body:** `{ students: [] }`
- **Source:** `src/services/userDataService.ts` - `checkUserFile()`

### 4.10 Archive

#### Move Pupil to Archive
- **Method:** `PATCH`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}`
- **Body:** `{ parentReference: { id: {archiveFolderId} } }`
- **Source:** `src/services/fileService.ts` - `movePupilToArchive()`

#### Get Archive Folder ID
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/Archive`
- **Response:** Folder object with `id`
- **Source:** `src/services/fileService.ts` - `getArchiveFolderId()`

#### Get Pupils in Archive
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/Archive:/children`
- **Response:** Array of item names
- **Source:** `src/services/fileService.ts` - `getpupilsInArchive()`

#### Move Archived Pupil Back
- **Method:** `PATCH`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/Archive/{studentName}`
- **Body:** `{ parentReference: { id: {pupilPassportsFolderId} } }`
- **Source:** `src/services/fileService.ts` - `moveArchivedPupilBack()`

### 4.11 Passport Documents (Word File Upload)

#### Upload Passport as .docx
- **Method:** `PUT`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{folderId}:/Passport Documents/{fileName}:/content`
- **Headers:** `Content-Type: application/vnd.openxmlformats-officedocument.wordprocessingml.document`
- **Body:** Blob (docx file)
- **Source:** `src/services/fileService.ts` - `uploadPassportToFolder()`

#### Create Passport Documents Folder
- **Method:** `POST`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/items/{studentFolderId}/children`
- **Body:** `{ name: "Passport Documents", folder: {} }`
- **Source:** `src/services/fileService.ts` - `createPassportDocsFolder()`

#### Get Pupil Passports Folder ID
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PupilPassportStudents`
- **Response:** Folder object with `id`
- **Source:** `src/services/fileService.ts` - `getpupilPassportsFolderId()`

### 4.12 Policy Documents

#### Get Inclusive Quality First Teaching Document
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PolicyDocuments/InclusiveQualityFirstTeaching.docx:/content`
- **Response:** ArrayBuffer (converted to text via mammoth)
- **Source:** `src/services/docsForAiService.ts` - `getIncStartsDoc()`

#### Get Strategies Document
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PolicyDocuments/Strategies.docx:/content`
- **Response:** ArrayBuffer (converted to text via mammoth)
- **Source:** `src/services/docsForAiService.ts` - `getStrategiesDoc()`

#### Get Outcomes Document
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PolicyDocuments/Outcomes.pdf:/content`
- **Response:** ArrayBuffer (converted to text via mammoth)
- **Source:** `src/services/docsForAiService.ts` - `getOutcomesDoc()`

#### Get Strategies Text File
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PolicyDocuments/Strategies.txt:/content`
- **Response:** Plain text
- **Source:** `src/services/docsForAiService.ts` - `getStratsTxt()`

#### Get Inclusive First Teaching Text File
- **Method:** `GET`
- **URL:** `https://graph.microsoft.com/v1.0/drives/{DriveId}/root:/eduWorkFlow/PolicyDocuments/InclusiveQualityFirstTeaching.txt:/content`
- **Response:** Plain text
- **Source:** `src/services/docsForAiService.ts` - `getIncFirstTeachingTxt()`

---

## 5. Microsoft Copilot API

### 5.1 Create New Conversation
- **Method:** `POST`
- **URL:** `https://graph.microsoft.com/beta/copilot/conversations`
- **Headers:** `Authorization: Bearer {authCode}`, `Content-Type: application/json`
- **Body:** `{}`
- **Response:** `{ chatId: string }` (from `data.id`)
- **Source:** `src/services/copilotService.ts` - `createNewConversation()`

### 5.2 Send Message to Copilot
- **Method:** `POST`
- **URL:** `https://graph.microsoft.com/beta/copilot/conversations/{conversationID}/chat`
- **Headers:** `Authorization: Bearer {authCode}`, `Content-Type: application/json`
- **Body:** `{ message: { text: string }, locationHint: { timeZone: "Europe/London" } }`
- **Response:** `{ messages: [...] }` - extracts last message's `text` field
- **Source:** `src/services/copilotService.ts` - `sendMessageToCopilot()`

### 5.3 Copilot Use Cases

| Use Case | Controller Function | Prompt File | Message Content |
|---|---|---|---|
| Generate Targets | `getTargetsFromCopilot()` | `getTargetsPrompt.txt` | Prompt + target details + student name |
| Generate Strategies | `getStrategiesFromCopilot()` | `getStratsPrompt.txt` | Prompt + targets + student name + count + saved strategies + policy docs |
| Generate Evaluation | `getEvaluationFromCopilot()` | `getEvalPrompt.txt` | Prompt + rowDTO (needTitle, logs, smartTargets, etc.) |
| Generate Provisions | `getProvisionsFromCopilot()` | `getProvisionsPrompt.txt` | Prompt + strategies JSON + existing provisions |
| Generate Evaluation Summary | `getEvaluationSummaryFromCopilot()` | `getEvaluationSummaryPrompt.txt` | Prompt + evaluations + term + year + previous summary |

---

## 6. Data Models

### studentPassportData
```typescript
{
  pupilName?: string
  preferredName?: string
  dob?: string
  pupilClass?: string
  term?: string
  classTeacher?: string
  mainAreaOfNeed?: string
  aboutMeNotes?: string
  supportPlans?: supportPlanRow[]
  consentToAnalytics?: boolean
  evaluationSummary?: string
}
```

### supportPlanRow
```typescript
{
  id: string
  needTitle: string
  secondaryNeedTitle: string
  tertiaryNeeds: string[]
  impact: string
  status: string
  logs?: LogEntry[]
  smartTargets: string[]
  strategies: string[]
  prevEvaluation?: string
}
```

### LogEntry
```typescript
{
  id: string
  date: string
  comments: string
  strategyEvaluation: logsStrategyEvaluation[]
}
```

### report
```typescript
{
  name: string
  year: string
  teacher: string
  teacherComments: string
  headTeacherComments: string
  notes: reportNotes[]
  stage: number
}
```

### reportNotes
```typescript
{
  id: string
  notes: string
  date: string
  name: string
  category: string
  description: string
}
```

### ProvisionsTable
```typescript
{
  provisions: { year: string; provisions: string }[]
}
```

### evaluationSummary
```typescript
{
  evaluation: string
  term: string
  year: string
}
```

### config
```typescript
{
  schoolName: string
  EntraClientId: string
  EntraTenantId: string
  EntraRedirectUri: string
  Email: string
  ConsentToAnalytics: boolean
  Address: string
  OneDriveId: string
  pupilPassport: boolean
  ResourceSupport: boolean
  PolicyHelper: boolean
  DocumentCompare: boolean
  IsMainLogo: boolean
  IsSecondaryLogo: boolean
  PassportSchoolPhrase: string
  BackendUrl: string
  AiBackendUrl: string
  AnalyticsBackendUrl: string
  AnalyticsUrl: string
}
```