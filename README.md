# Student Marks and Grade Management API

A backend-only REST API built with Node.js, Express, and MongoDB. Students,
subjects, exam marks, and users are stored in MongoDB. Administrators and
teachers sign in with email/password and use a JWT bearer token for API access.

## Requirements

- Node.js 20 or later and npm
- MongoDB 6 or later, either local or hosted

## Install and run on Windows PowerShell

From `E:\STUD_API`:

```powershell
npm ci
Copy-Item .env.example .env
```

Edit `.env` and set `MONGODB_URI`, a private random `JWT_SECRET` of at least 32
characters, and the initial administrator's name, email, and password. The
seed password must be at least 12 characters. Generate a secret with:

```powershell
node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"
```

Start MongoDB first. For a local MongoDB installation, its default URI in
`.env.example` creates/uses the `student_marks` database. For MongoDB Atlas,
replace it with the connection string provided by Atlas and allow the
development machine's IP in the Atlas network access settings.

Start the API:

```powershell
npm run dev
```

For a normal (non-watch) run:

```powershell
npm start
```

The API listens on `http://localhost:8000` by default. Change `PORT` in `.env`
to use a different port. `GET http://localhost:8000/health` is a public
availability check.

The first administrator is created from `SEED_ADMIN_NAME`,
`SEED_ADMIN_EMAIL`, and `SEED_ADMIN_PASSWORD` on startup, if that email is not
already registered. Change or remove these seed values after initial setup.
Administrators can create further administrator and teacher accounts through
the user API. There is no public registration endpoint.

## Authentication and conventions

1. Sign in with `POST /api/auth/login` using the seed administrator credentials.
2. Copy the returned token.
3. In Postman, choose **Authorization > Bearer Token** and paste the token.
   Alternatively, send `Authorization: Bearer <token>` on each protected call.
4. Create students, subjects, marks, and user accounts. Both administrators
   and teachers can manage academic records; only administrators can manage
   user accounts.

All API paths are under `http://localhost:8000`. JSON request bodies use
`Content-Type: application/json`. IDs in path parameters are MongoDB document
IDs unless the student endpoint explicitly also accepts a student's
`studentId`. Successful creates return `201`, reads/updates return `200`,
deletes return `204`, invalid input returns `400`, missing records return
`404`, duplicate unique values return `409`, and unauthenticated requests
return `401`.

## Grade and result rules

Grades use the exact percentage bands: `A+` >= 90, `A` >= 80, `B` >= 70,
`C` >= 60, `D` >= 50, otherwise `F`. Each subject defines its own `maxMarks`
and `passMarks`. A student passes an exam only when their overall percentage
is at least 50% **and** their marks meet the pass-mark threshold in every
recorded subject for that exam. Overall
percentage is total marks divided by total possible marks, times 100. Result
reports are calculated from the marks currently stored; they are not stored
separately.

Each marks record is for one student, subject, and exam name. That combination
must be unique. Deleting a student or subject also deletes its related marks.

## API reference and Postman examples

### Import and run the Postman collection

The repository includes an importable collection and a local environment:

- `postman/Student_Marks_API.postman_collection.json`
- `postman/Student_Marks_API.postman_environment.json`

1. Start MongoDB and the API using the setup steps above. Confirm
   `http://localhost:8000/health` returns `{"status":"ok"}`.
2. In Postman, choose **Import**, then select both JSON files from the
   `postman` folder.
3. Select **Student Marks API - Local** in the environment picker.
4. Edit the environment's `adminEmail` and `adminPassword` to match
   `SEED_ADMIN_EMAIL` and `SEED_ADMIN_PASSWORD` in `.env`.
5. Open **Student Marks and Grade Management API** and click **Run**. Run the
   requests in their listed order; the collection saves the login token and
   resource IDs automatically and includes cleanup requests at the end.
6. Check the Collection Runner results for each request's status and test.

The collection creates unique test accounts, student IDs, and subject codes
for each run. Its cleanup folder removes the created records at the end. If a
run stops before cleanup, remove the test data manually or run the relevant
cleanup requests using the saved environment IDs.

To send a request individually, choose **Authorization > Bearer Token** and
use the token returned by `POST /api/auth/login` for every endpoint except
login and `/health`. The sample requests and responses below can also be used
to create requests manually in Postman.

To run the same collection from PowerShell without the Postman UI (Newman):

```powershell
npm run postman
```

Keep the API running in a separate terminal. Newman uses the local Postman
environment JSON; update its `adminEmail` and `adminPassword` values to match
`.env` before running.

### Authentication and users

| Method and URL | Access | Purpose |
|---|---|---|
| `POST /api/auth/login` | Public | Sign in and receive a JWT |
| `GET /api/auth/me` | Admin or teacher | Get the current account |
| `POST /api/users` | Admin | Create an administrator or teacher |
| `GET /api/users` | Admin | List users |
| `GET /api/users/:id` | Admin | Get a user |
| `PUT /api/users/:id` | Admin | Update user details |
| `DELETE /api/users/:id` | Admin | Delete another user; the last admin cannot be deleted |

**Sign in** - `POST {{baseUrl}}/api/auth/login`

Request:

```json
{
  "email": "admin@example.com",
  "password": "ChangeThisAdminPassword123!"
}
```

Response (`200`):

```json
{
  "token": "<jwt-token>",
  "tokenType": "Bearer",
  "expiresIn": "1d",
  "user": {
    "id": "665000000000000000000001",
    "name": "System Administrator",
    "email": "admin@example.com",
    "role": "admin"
  }
}
```

**Current user** - `GET {{baseUrl}}/api/auth/me`

Response (`200`):

```json
{
  "user": {
    "_id": "665000000000000000000001",
    "name": "System Administrator",
    "email": "admin@example.com",
    "role": "admin"
  }
}
```

**Create user** - `POST {{baseUrl}}/api/users`

Request:

```json
{
  "name": "Taylor Teacher",
  "email": "taylor@example.com",
  "password": "TeacherPassword123!",
  "role": "teacher"
}
```

Response (`201`):

```json
{
  "user": {
    "_id": "665000000000000000000002",
    "name": "Taylor Teacher",
    "email": "taylor@example.com",
    "role": "teacher"
  }
}
```

**List users** - `GET {{baseUrl}}/api/users`

Response (`200`):

```json
[
  {
    "_id": "665000000000000000000001",
    "name": "System Administrator",
    "email": "admin@example.com",
    "role": "admin"
  }
]
```

**Get user** - `GET {{baseUrl}}/api/users/{{userId}}`

Response (`200`):

```json
{
  "_id": "665000000000000000000002",
  "name": "Taylor Teacher",
  "email": "taylor@example.com",
  "role": "teacher",
  "createdAt": "2026-10-30T08:00:00.000Z",
  "updatedAt": "2026-10-30T08:00:00.000Z"
}
```

**Update user** - `PUT {{baseUrl}}/api/users/{{userId}}`

Request (send one or more fields):

```json
{
  "name": "Taylor A. Teacher",
  "role": "teacher"
}
```

Response (`200`):

```json
{
  "user": {
    "_id": "665000000000000000000002",
    "name": "Taylor A. Teacher",
    "email": "taylor@example.com",
    "role": "teacher"
  }
}
```

**Delete user** - `DELETE {{baseUrl}}/api/users/{{userId}}`

Response: `204 No Content` (empty body).

### Students

| Method and URL | Purpose |
|---|---|
| `POST /api/students` | Add a student |
| `GET /api/students` | List students; optional `?search=...` and `?className=...` |
| `GET /api/students/:id` | Get by MongoDB ID or public `studentId` |
| `PUT /api/students/:id` | Update one or more student fields |
| `DELETE /api/students/:id` | Delete a student and their marks |

**Add student** - `POST {{baseUrl}}/api/students`

Request:

```json
{
  "studentId": "STU-2026-001",
  "firstName": "Asha",
  "lastName": "Rao",
  "email": "asha.rao@example.com",
  "dateOfBirth": "2008-04-15",
  "className": "Grade 12",
  "guardianPhone": "+1-555-0100"
}
```

Response (`201`):

```json
{
  "_id": "665000000000000000000010",
  "studentId": "STU-2026-001",
  "firstName": "Asha",
  "lastName": "Rao",
  "email": "asha.rao@example.com",
  "dateOfBirth": "2008-04-15T00:00:00.000Z",
  "className": "Grade 12",
  "guardianPhone": "+1-555-0100",
  "createdAt": "2026-10-30T08:00:00.000Z",
  "updatedAt": "2026-10-30T08:00:00.000Z"
}
```

**List or search students** - `GET {{baseUrl}}/api/students?search=Asha&className=Grade%2012`

Response (`200`):

```json
[
  {
    "_id": "665000000000000000000010",
    "studentId": "STU-2026-001",
    "firstName": "Asha",
    "lastName": "Rao",
    "email": "asha.rao@example.com",
    "dateOfBirth": "2008-04-15T00:00:00.000Z",
    "className": "Grade 12",
    "guardianPhone": "+1-555-0100"
  }
]
```

The array is empty if no student matches. `search` checks the public ID, first
name, last name, and email.

**Get student** - `GET {{baseUrl}}/api/students/{{studentId}}`

Response (`200`):

```json
{
  "_id": "665000000000000000000010",
  "studentId": "STU-2026-001",
  "firstName": "Asha",
  "lastName": "Rao",
  "email": "asha.rao@example.com",
  "dateOfBirth": "2008-04-15T00:00:00.000Z",
  "className": "Grade 12",
  "guardianPhone": "+1-555-0100"
}
```

The path can also use `{{studentObjectId}}`.

**Update student** - `PUT {{baseUrl}}/api/students/{{studentObjectId}}`

Request (send one or more fields):

```json
{
  "className": "Grade 12 - A"
}
```

Response (`200`):

```json
{
  "_id": "665000000000000000000010",
  "studentId": "STU-2026-001",
  "firstName": "Asha",
  "lastName": "Rao",
  "email": "asha.rao@example.com",
  "dateOfBirth": "2008-04-15T00:00:00.000Z",
  "className": "Grade 12 - A",
  "guardianPhone": "+1-555-0100"
}
```

**Delete student** - `DELETE {{baseUrl}}/api/students/{{studentObjectId}}`

Response: `204 No Content` (empty body); its marks are removed as well.

### Subjects

| Method and URL | Purpose |
|---|---|
| `POST /api/subjects` | Add a subject |
| `GET /api/subjects` | List subjects |
| `PUT /api/subjects/:id` | Update a subject |
| `DELETE /api/subjects/:id` | Delete a subject and its marks |

**Add subject** - `POST {{baseUrl}}/api/subjects`

Request:

```json
{
  "code": "MATH",
  "name": "Mathematics",
  "maxMarks": 100,
  "passMarks": 35,
  "description": "End-of-term mathematics exam"
}
```

Response (`201`):

```json
{
  "_id": "665000000000000000000030",
  "code": "MATH",
  "name": "Mathematics",
  "maxMarks": 100,
  "passMarks": 35,
  "description": "End-of-term mathematics exam",
  "createdAt": "2026-10-30T08:00:00.000Z",
  "updatedAt": "2026-10-30T08:00:00.000Z"
}
```

**List subjects** - `GET {{baseUrl}}/api/subjects`

Response (`200`):

```json
[
  {
    "_id": "665000000000000000000030",
    "code": "MATH",
    "name": "Mathematics",
    "maxMarks": 100,
    "passMarks": 35,
    "description": "End-of-term mathematics exam"
  }
]
```

**Update subject** - `PUT {{baseUrl}}/api/subjects/{{subjectId}}`

Request (send one or more fields):

```json
{
  "passMarks": 40
}
```

Response (`200`):

```json
{
  "_id": "665000000000000000000030",
  "code": "MATH",
  "name": "Mathematics",
  "maxMarks": 100,
  "passMarks": 40,
  "description": "End-of-term mathematics exam"
}
```

`maxMarks` cannot be lowered below marks already recorded for that subject.

**Delete subject** - `DELETE {{baseUrl}}/api/subjects/{{subjectId}}`

Response: `204 No Content` (empty body); its marks are removed as well.

### Marks

| Method and URL | Purpose |
|---|---|
| `POST /api/marks` | Record marks for a student and subject |
| `GET /api/marks/:studentId` | List a student's marks; optional `?examName=...` and `?subjectId=...` |
| `PUT /api/marks/:id` | Update marks, exam name, or remarks |
| `DELETE /api/marks/:id` | Delete a marks record |

For create, `studentId` accepts the student's public ID or MongoDB ID.
`subjectId` accepts a subject code or MongoDB ID.

**Add marks** - `POST {{baseUrl}}/api/marks`

Request:

```json
{
  "studentId": "STU-2026-001",
  "subjectId": "MATH",
  "examName": "Final 2026",
  "marks": 88,
  "remarks": "Excellent work"
}
```

Response (`201`):

```json
{
  "_id": "665000000000000000000020",
  "student": {
    "_id": "665000000000000000000010",
    "studentId": "STU-2026-001",
    "firstName": "Asha",
    "lastName": "Rao",
    "className": "Grade 12"
  },
  "subject": {
    "_id": "665000000000000000000030",
    "code": "MATH",
    "name": "Mathematics",
    "maxMarks": 100,
    "passMarks": 35
  },
  "examName": "Final 2026",
  "marks": 88,
  "remarks": "Excellent work",
  "createdAt": "2026-10-30T08:00:00.000Z",
  "updatedAt": "2026-10-30T08:00:00.000Z"
}
```

**Get student marks** - `GET {{baseUrl}}/api/marks/STU-2026-001?examName=Final%202026`

Response (`200`):

```json
[
  {
    "_id": "665000000000000000000020",
    "student": {
      "_id": "665000000000000000000010",
      "studentId": "STU-2026-001",
      "firstName": "Asha",
      "lastName": "Rao",
      "className": "Grade 12"
    },
    "subject": {
      "_id": "665000000000000000000030",
      "code": "MATH",
      "name": "Mathematics",
      "maxMarks": 100,
      "passMarks": 35
    },
    "examName": "Final 2026",
    "marks": 88,
    "remarks": "Excellent work"
  }
]
```

The path also accepts the student's MongoDB ID.

**Update marks** - `PUT {{baseUrl}}/api/marks/{{markId}}`

Request:

```json
{
  "marks": 92,
  "remarks": "Rechecked"
}
```

Response (`200`):

```json
{
  "_id": "665000000000000000000020",
  "student": {
    "_id": "665000000000000000000010",
    "studentId": "STU-2026-001",
    "firstName": "Asha",
    "lastName": "Rao",
    "className": "Grade 12"
  },
  "subject": {
    "_id": "665000000000000000000030",
    "code": "MATH",
    "name": "Mathematics",
    "maxMarks": 100,
    "passMarks": 35
  },
  "examName": "Final 2026",
  "marks": 92,
  "remarks": "Rechecked"
}
```

**Delete marks** - `DELETE {{baseUrl}}/api/marks/{{markId}}`

Response: `204 No Content` (empty body).

### Results and performance reports

| Method and URL | Purpose |
|---|---|
| `GET /api/results/:studentId` | Student report(s); optional `?examName=...` |
| `GET /api/results` | Reports for all students; optional `?className=...&examName=...` |
| `GET /api/results/topper` | Top reports, ordered by percentage; optional class, exam, and `limit` filters |

**Student result** - `GET {{baseUrl}}/api/results/STU-2026-001?examName=Final%202026`

Response (`200`):

```json
{
  "student": {
    "_id": "665000000000000000000010",
    "studentId": "STU-2026-001",
    "firstName": "Asha",
    "lastName": "Rao",
    "className": "Grade 12"
  },
  "results": [
    {
      "student": {
        "_id": "665000000000000000000010",
        "studentId": "STU-2026-001",
        "firstName": "Asha",
        "lastName": "Rao",
        "email": "asha.rao@example.com",
        "className": "Grade 12"
      },
      "examName": "Final 2026",
      "marks": [
        {
          "id": "665000000000000000000020",
          "examName": "Final 2026",
          "marks": 88,
          "remarks": "Excellent work",
          "subject": {
            "_id": "665000000000000000000030",
            "code": "MATH",
            "name": "Mathematics",
            "maxMarks": 100,
            "passMarks": 35
          }
        }
      ],
      "totalMarks": 88,
      "maxMarks": 100,
      "percentage": 88,
      "grade": "A",
      "status": "Pass"
    }
  ]
}
```

Omit `examName` to get an entry in `results` for each exam with marks.

**All/class results** - `GET {{baseUrl}}/api/results?className=Grade%2012&examName=Final%202026`

Response (`200`):

```json
[
  {
    "student": {
      "_id": "665000000000000000000010",
      "studentId": "STU-2026-001",
      "firstName": "Asha",
      "lastName": "Rao",
      "email": "asha.rao@example.com",
      "className": "Grade 12"
    },
    "examName": "Final 2026",
    "marks": [
      {
        "id": "665000000000000000000020",
        "examName": "Final 2026",
        "marks": 88,
        "remarks": "Excellent work",
        "subject": {
          "_id": "665000000000000000000030",
          "code": "MATH",
          "name": "Mathematics",
          "maxMarks": 100,
          "passMarks": 35
        }
      }
    ],
    "totalMarks": 88,
    "maxMarks": 100,
    "percentage": 88,
    "grade": "A",
    "status": "Pass"
  }
]
```

Omit either query filter to include all classes or exams.

**Top students** - `GET {{baseUrl}}/api/results/topper?className=Grade%2012&examName=Final%202026&limit=5`

Response (`200`):

```json
[
  {
    "student": {
      "_id": "665000000000000000000010",
      "studentId": "STU-2026-001",
      "firstName": "Asha",
      "lastName": "Rao",
      "email": "asha.rao@example.com",
      "className": "Grade 12"
    },
    "examName": "Final 2026",
    "marks": [
      {
        "id": "665000000000000000000020",
        "examName": "Final 2026",
        "marks": 88,
        "remarks": "Excellent work",
        "subject": {
          "_id": "665000000000000000000030",
          "code": "MATH",
          "name": "Mathematics",
          "maxMarks": 100,
          "passMarks": 35
        }
      }
    ],
    "totalMarks": 88,
    "maxMarks": 100,
    "percentage": 88,
    "grade": "A",
    "status": "Pass"
  }
]
```

Reports are ordered by percentage, with `limit` defaulting to 10 and accepting
values from 1 to 100.

## Validation and error responses

Required fields, lengths, email, date, and numeric ranges are validated before
database writes. A mark cannot exceed its subject's `maxMarks`. Subject
`passMarks` must be at least 1 and no greater than `maxMarks`. Duplicate
student IDs/emails, subject codes, user emails, or student/subject/exam mark
combinations return `409`. Passwords are hashed with bcrypt; the stored hash
is never included in API responses. Passwords must be at least 12 characters
and no more than 72 UTF-8 bytes. Login is rate limited.

Example validation error (`400`):

```json
{
  "error": "Validation failed",
  "details": [
    {
      "field": "email",
      "message": "Invalid email"
    }
  ]
}
```

## Tests

Run the grade-boundary and result-calculation tests:

```powershell
npm test
```

## Dependencies

Runtime packages are declared in `package.json`: Express, Mongoose, Zod,
`jsonwebtoken`, `bcryptjs`, `dotenv`, Helmet, CORS, and `express-rate-limit`.
Supertest is included as a development dependency for API tests.
