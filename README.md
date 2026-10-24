# React Midterm — CI/CD with Jenkins and Docker

React + Vite calculator used for the midterm assessment. Jenkins builds and
tests the application in a multi-stage Docker image, archives a versioned
release artifact, and a second, manually triggered job deploys that artifact to
dev, test, acceptance, or production without rebuilding React.

Student: Alexander Jade Francisco

## Application

`src/calculator.js` exports `add` and `subtract`. The UI in `src/App.jsx`
renders one button per operation and shows the current application version.

## Local development

```bash
npm install
npm test          # vitest run, 6 tests
npm run build     # production output in dist/
npm run dev       # local Vite server
```

## Docker

- `Dockerfile` — multi-stage build: `node:lts-alpine` installs dependencies,
  runs the tests, and builds the app; `nginx:stable-alpine` serves the result.
- `Dockerfile.deploy` — artifact-only image. Copies a prebuilt `site/` folder
  into Nginx. It never runs npm, so deployment cannot rebuild React.
- `.dockerignore` — keeps `node_modules`, `dist`, `.git`, and pipeline
  scratch directories out of the build context.

```bash
docker build -t react-midterm .
docker run -d -p 8080:80 react-midterm
```

## Jenkins jobs

### React-CI (freestyle, Poll SCM `*/2 * * * *`)

1. `docker build` the multi-stage image. A failing test fails the build, so no
   artifact is produced.
2. Extract `/usr/share/nginx/html` from the image into `extracted/site`.
3. Write `release.json` with `ci_build` and `git_commit`.
4. Copy `Dockerfile.deploy` and package everything as
   `artifacts/react-app-${BUILD_NUMBER}.tar.gz`.
5. Archive `artifacts/react-app-*.tar.gz` (only on success, fingerprinted).
6. Grant React-Deploy permission to copy artifacts.

### React-Deploy (freestyle, parameterized, manual)

Parameters: `TARGET_ENVIRONMENT` (dev, test, acceptance, production) and
`CI_BUILD_NUMBER` (string, blank by default).

1. Validate both parameters and clear `incoming/` and `deploy-context/`.
2. Copy `artifacts/react-app-${CI_BUILD_NUMBER}.tar.gz` from React-CI into
   `incoming/` (flatten, optional unchecked). A missing artifact fails here,
   before any environment is touched.
3. Extract, build `react-midterm-release:${CI_BUILD_NUMBER}` from
   `Dockerfile.deploy`, replace only `react-${TARGET_ENVIRONMENT}`, and poll the
   container until Nginx answers.

| Environment | Container | URL |
| --- | --- | --- |
| dev | react-dev | http://localhost:8081 |
| test | react-test | http://localhost:8082 |
| acceptance | react-acceptance | http://localhost:8083 |
| production | react-production | http://localhost:8084 |

Release metadata for any environment: `http://localhost:<port>/release.json`.
