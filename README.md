# Newman API Automation

This repository contains an Express/Mongoose task API in `mern-backend` and a
Postman collection that Newman runs in CI.

## Run locally

1. Install Node.js 22 LTS and MongoDB, or start MongoDB with Docker:

   ```powershell
   docker run --name task-api-mongo -p 27017:27017 -d mongo:7
   ```

2. From `mern-backend`, create a `.env` file:

   ```text
   MONGO_URI=mongodb://127.0.0.1:27017/taskdb
   JWT_SECRET=replace-with-a-local-secret
   PORT=5000
   ```

3. Install dependencies and start the API:

   ```powershell
   npm ci
   npm start
   ```

   The API is available at `http://localhost:5000`. The health check is
   `http://localhost:5000/health`.

## Run the Newman tests

Keep the API running, open a second terminal in `mern-backend`, and run:

```powershell
npm run test:api
```

The collection registers and logs in a test user, stores the returned JWT in
the `authToken` environment variable, then creates, reads, updates, and
deletes a task. Newman writes a JUnit report to `mern-backend/newman/report.xml`
when run from the repository root through CI, or to `newman/report.xml` when
run from `mern-backend`.

## GitHub Actions CI

The workflow at `.github/workflows/api-tests.yml` runs for every push to
`main` and every pull request targeting `main`. It:

1. Checks out the repository and installs Node.js 22.
2. Starts a fresh MongoDB 7 service container.
3. Runs `npm ci` in `mern-backend`.
4. Starts the API with CI-only database and JWT values.
5. Waits for `/health` to return HTTP 200.
6. Runs the Newman collection.
7. Uploads the JUnit report as the `newman-report` artifact, even when tests
   fail.

No MongoDB Atlas or repository secrets are required for this CI workflow.
After pushing backend changes to `main`, open the repository's **Actions** tab
and select **API Tests with Newman**. A green run means the API started and all
collection assertions passed; a red run means the Newman output and uploaded
report should be inspected. You can also start a run manually with
**Actions > API Tests with Newman > Run workflow**.

## Block merging when API tests fail

The workflow result becomes a required status check only after branch
protection is configured:

1. Push this workflow to GitHub and open **Settings > Rules > Rulesets** (or
   **Branches > Add branch protection rule**).
2. Target the `main` branch.
3. Require a pull request before merging.
4. Require status checks to pass before merging.
5. Add the check named **API Tests with Newman / api-tests**.
6. Enable **Require branches to be up to date before merging** if you want the
   pull request to be retested against the latest `main`.
7. Save the rule.

Now a failing Newman assertion makes the `api-tests` check red and GitHub
disables the merge button. A pull request cannot merge until the code is
updated and the check succeeds. Repository administrators can bypass rules
unless the rule is configured to apply to administrators too.

## Postman and Newman

Postman is used to design and manually send the requests and assertions.
The exported files are `mern-backend/TestAPI/task-api-collection.json` and
`mern-backend/TestAPI/task-env.json`. Newman is Postman's command-line runner:
it executes the same collection non-interactively in local terminals and
GitHub Actions. The login test stores the JWT, and the task requests send it
as `Authorization: Bearer {{authToken}}`.
