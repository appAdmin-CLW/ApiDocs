# EduWorkFlow API Documentation

## Overview

This application makes API calls to two main services:
1. **Microsoft Graph API** (`https://graph.microsoft.com/v1.0`) - For OneDrive file storage operations
2. **Custom Backend API** (configured via `config.json` `BackendUrl`) - For copilot/LLM, user presence, RAG, and workbook processing
3. **Microsoft Copilot API** (`https://graph.microsoft.com/beta/copilot/conversations`) - For AI-powered content generation

---

## 1. Microsoft Graph API - OneDrive Storage

All Graph API calls use the configured `OneDriveId` from `config.json` and authenticate with a Bearer token obtained via MSAL.

### 1.1 Passport Data

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

### 1.2 Application Data Files

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

### 1.3 Student Folders

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
- **Source:** `src/services/studentFolderService.ts` - `ensureDatabaseStructure()`

### 1.4 Logs

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

### 1.5 Provisions

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

### 1.6 Evaluation Summary

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

### 1.7 User-Specific Saved Students

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

### 1.8 Archive

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

### 1.9 Passport Documents (Word File Upload)

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

### 1.10 Policy Documents

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

## 2. Custom Backend API

Base URL is configured via `config.json` `BackendUrl` field.

### 2.1 Server Health Check

#### Server Check
- **Method:** `GET`
- **URL:** `{BackendUrl}/server-check`
- **Response:** `{ ok: boolean, dbOk: boolean }` (or `true/false` / `{ database: boolean }`)
- **Source:** `src/services/backendService.ts` - `serverCheck()`

### 2.2 User Presence

#### Send Presence
- **Method:** `POST`
- **URL:** `{BackendUrl}/userPresence?passportId={passportId}&user={user}`
- **Source:** `src/services/userPresenceService.ts` - `sendPresence()`

#### Get Presence
- **Method:** `GET`
- **URL:** `{BackendUrl}/userPresence?passportId={passportId}`
- **Response:** Array of user names
- **Source:** `src/services/userPresenceService.ts` - `getPresence()`

### 2.3 RAG (Retrieval Augmented Generation)

#### Load Documents
- **Method:** `GET`
- **URL:** `{BackendUrl}/documents?user_id={userId}`
- **Headers:** `authToken: {token}`
- **Response:** `Document[]`
- **Source:** `src/services/ragService.ts` - `loadDocuments()`

#### Upload Document
- **Method:** `POST`
- **URL:** `{BackendUrl}/upload?user_id={userId}`
- **Headers:** `authToken: {token}`
- **Body:** FormData with `file` field
- **Source:** `src/services/ragService.ts` - `uploadDocument()`

#### Ask Question
- **Method:** `POST`
- **URL:** `{BackendUrl}/ask`
- **Headers:** `authorization: {token}`, `Content-Type: application/json`
- **Body:** `{ question: string, document_ids: string[], user_id: string }`
- **Source:** `src/services/ragService.ts` - `askQuestion()`

#### Compare Documents
- **Method:** `POST`
- **URL:** `{BackendUrl}/compare`
- **Body:** FormData with `file1` and `file2` fields
- **Source:** `src/services/ragService.ts` - `compareDocuments()`

### 2.4 Workbook Processing

#### Process Workbook
- **Method:** `POST`
- **URL:** `{BackendUrl}/process-workbook`
- **Headers:** `authToken: {token}`
- **Body:** FormData (raw data)
- **Source:** `src/services/workbookService.ts` - `processWorkbook()`

### 2.5 Edit Counts

#### Send Edit Counts
- **Method:** `POST`
- **URL:** `/api/editCounts` (relative to app origin)
- **Headers:** `Content-Type: application/json`
- **Body:** `{ editCounts: Record<string, { targetEdited: boolean, strategiesEdited: boolean[] }> }`
- **Source:** `src/pages/pupilPassport/passport/passport.tsx` - `savePassport()`

---

## 3. Microsoft Copilot API

### 3.1 Create New Conversation
- **Method:** `POST`
- **URL:** `https://graph.microsoft.com/beta/copilot/conversations`
- **Headers:** `Authorization: Bearer {authCode}`, `Content-Type: application/json`
- **Body:** `{}`
- **Response:** `{ chatId: string }` (from `data.id`)
- **Source:** `src/services/copilotService.ts` - `createNewConversation()`

### 3.2 Send Message to Copilot
- **Method:** `POST`
- **URL:** `https://graph.microsoft.com/beta/copilot/conversations/{conversationID}/chat`
- **Headers:** `Authorization: Bearer {authCode}`, `Content-Type: application/json`
- **Body:** `{ message: { text: string }, locationHint: { timeZone: "Europe/London" } }`
- **Response:** `{ messages: [...] }` - extracts last message's `text` field
- **Source:** `src/services/copilotService.ts` - `sendMessageToCopilot()`

### 3.3 Copilot Use Cases

| Use Case | Controller Function | Prompt File | Message Content |
|---|---|---|---|
| Generate Targets | `getTargetsFromCopilot()` | `getTargetsPrompt.txt` | Prompt + target details + student name |
| Generate Strategies | `getStrategiesFromCopilot()` | `getStratsPrompt.txt` | Prompt + targets + student name + count + saved strategies + policy docs |
| Generate Evaluation | `getEvaluationFromCopilot()` | `getEvalPrompt.txt` | Prompt + rowDTO (needTitle, logs, smartTargets, etc.) |
| Generate Provisions | `getProvisionsFromCopilot()` | `getProvisionsPrompt.txt` | Prompt + strategies JSON + existing provisions |
| Generate Evaluation Summary | `getEvaluationSummaryFromCopilot()` | `getEvaluationSummaryPrompt.txt` | Prompt + evaluations + term + year + previous summary |

---

## 4. Data Models

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
  startegyEvaluation: logsStartegyEvaluation[]
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
}