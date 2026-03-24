# Reusable Copilot Prompt: Enhance Local Dev Experience

Copy the following prompt and use it in each Single Front Door (SFD) service repo. Replace `{SERVICE_NAME}` with the actual service name (e.g., `fcp-sfd-object-processor`, `fcp-sfd-frontend`).

---

**Prompt:**

Apply the following local development experience enhancements to this service (`{SERVICE_NAME}`). All changes should follow ES module conventions (`"type": "module"`) and use neostandard formatting. Make each change exactly as specified:

### 1. Migrate linter from standard to neostandard
- Remove `standard` from `devDependencies` in `package.json` by running `npm uninstall standard`
- Ensure the `package.json` is up to date by running `npm install` followed by `npm audit`
- Based on the result of the `npm audit`, apply any fixes necessary
- Add `"eslint": "9.39.4"` and `"neostandard": "0.13.0"` to `devDependencies` by running `npm install -D neostandard eslint`
- Create `eslint.config.js` in the project root with this exact content by running `npx neostandard --esm > eslint.config.js`
- In `package.json` scripts, remove any `pretest` script that references `test:lint` and remove the `test:lint` script itself
- Add these scripts:
  - `"lint": "npx eslint . --ext .js"`
  - `"lint:fix": "npx eslint . --ext .js --fix"`

### 2. Add convenience Docker npm scripts
Add these scripts to `package.json` (preserve all existing scripts):
- `"docker:build": "docker compose build"`
- `"docker:dev": "docker compose up"`
- `"docker:dev:d": "docker compose up -d"`
- `"docker:stop": "docker compose down"`
- `"docker:stop:v": "docker compose down -v"`
- `"docker:debug": "docker compose -f compose.yaml -f compose.debug.yaml -p '{SERVICE_NAME}' up"`

Also add a `start:debug` script:
- `"start:debug": "nodemon --watch src --exec 'node --experimental-vm-modules --inspect-brk=0.0.0.0 src/index.js'"`

And a sonar scan script:
- `"sonar": "node scripts/sonar-scan.js"`

### 3. Create Docker debug compose override
Create `compose.debug.yaml` in the project root:
```yaml
services:
  {SERVICE_NAME}:
    command: npm run start:debug
```

### 4. Create VS Code tasks
Create `.vscode/tasks.json` with this content (adapt the script names to match the npm scripts above):
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "npm",
      "label": "🛠️ Build Docker container",
      "script": "docker:build"
    },
    {
      "type": "npm",
      "label": "🏁 Start Docker container",
      "script": "docker:dev",
      "problemMatcher": []
    },
    {
      "type": "npm",
      "label": "✂️ Start Docker container (detached mode)",
      "script": "docker:dev:d",
      "problemMatcher": []
    },
    {
      "type": "npm",
      "label": "🛑 Stop Docker container",
      "script": "docker:stop",
      "problemMatcher": []
    },
    {
      "type": "npm",
      "label": "😵 Stop Docker container + delete volumes",
      "script": "docker:stop:v",
      "problemMatcher": []
    },
    {
      "type": "npm",
      "label": "🧪 Run tests",
      "script": "docker:test",
      "problemMatcher": []
    },
    {
      "type": "npm",
      "label": "👀 Run tests (watch mode)",
      "script": "docker:test:watch"
    },
    {
      "type": "npm",
      "label": "🪲 Run debugger",
      "script": "docker:debug",
      "problemMatcher": []
    },
    {
      "type": "npm",
      "label": "📋 Run linter",
      "script": "lint",
      "problemMatcher": []
    },
    {
      "type": "npm",
      "label": "🧹 Run linter + auto fix known issues",
      "script": "lint:fix",
      "problemMatcher": []
    },
    {
      "type": "npm",
      "label": "☁️ SonarQube Cloud scan",
      "script": "sonar",
      "problemMatcher": []
    }
  ]
}
```

### 5. Enable pre-commit hooks
Check if a `.pre-commit-config.yaml` file exists in the project root.

**If it does NOT exist**, create `.pre-commit-config.yaml` with the following exact contents:
```yaml
repos:
- repo: https://github.com/Yelp/detect-secrets
  rev: v1.5.0
  hooks:
    - id: detect-secrets
      args: ['--baseline', '.secrets.baseline']

- repo: local
  hooks:
    - id: eslint-fix
      name: ESLint with neostandard
      entry: npm run lint:fix
      language: node
```

**If it already exists**, apply these modifications:
- Bump `detect-secrets` rev to `v1.5.0` if it is an older version
- Ensure hooks are properly indented (hooks list items should be 4-space indented under their repo)
- Add the local `eslint-fix` hook block (shown above) if not already present

After creating or updating the file, run `pre-commit install` in the terminal to activate the hooks for the local repository.

### 6. Create SonarQube Cloud local scan script
Create the `scripts/` directory if it doesn't exist, then create `scripts/sonar-scan.js` with the following exact contents:
```js
import { readFile } from 'node:fs/promises'
import { execFileSync, spawn } from 'node:child_process'
import { existsSync } from 'node:fs'
import { resolve } from 'node:path'

const SONARCLOUD_BASE_URL = 'https://sonarcloud.io'
const BORDER = '═'.repeat(51)
const THIN_BORDER = '─'.repeat(51)
const MAX_ISSUES_DISPLAYED = 30

const SEVERITY_ORDER = ['BLOCKER', 'CRITICAL', 'MAJOR', 'MINOR', 'INFO']
const SEVERITY_ICONS = {
  BLOCKER: '🔴',
  CRITICAL: '🟠',
  MAJOR: '🟡',
  MINOR: '🔵',
  INFO: '⚪'
}

const HOTSPOT_ICONS = {
  HIGH: '🔴',
  MEDIUM: '🟠',
  LOW: '🟡'
}

const METRIC_LABELS = {
  new_reliability_rating: 'Reliability Rating',
  new_security_rating: 'Security Rating',
  new_maintainability_rating: 'Maintainability Rating',
  new_coverage: 'Coverage on New Code',
  new_duplicated_lines_density: 'Duplication on New Code',
  new_violations: 'New Issues',
  new_security_hotspots_reviewed: 'Security Hotspots Reviewed',
  new_blocker_violations: 'Blocker Issues',
  new_critical_violations: 'Critical Issues'
}

const COMPARATOR_SYMBOLS = {
  GT: '>',
  LT: '<',
  EQ: '=',
  NE: '≠'
}

const parseDotEnv = async (filePath) => {
  const vars = {}

  try {
    const content = await readFile(filePath, 'utf8')

    for (const line of content.split('\n')) {
      const trimmed = line.trim()

      if (!trimmed || trimmed.startsWith('#')) continue

      const eqIndex = trimmed.indexOf('=')

      if (eqIndex === -1) continue

      const key = trimmed.slice(0, eqIndex).trim()
      const value = trimmed.slice(eqIndex + 1).trim().replace(/^["']|["']$/g, '')
      vars[key] = value
    }
  } catch {
    // .env file not found or unreadable — continue with process.env only
  }

  return vars
}

const parseSonarProperties = async (filePath) => {
  const props = {}
  const content = await readFile(filePath, 'utf8')

  for (const line of content.split('\n')) {
    const trimmed = line.trim()

    if (!trimmed || trimmed.startsWith('#')) continue

    const eqIndex = trimmed.indexOf('=')

    if (eqIndex === -1) continue

    props[trimmed.slice(0, eqIndex).trim()] = trimmed.slice(eqIndex + 1).trim()
  }

  return props
}

const getCurrentBranch = () =>
  execFileSync('git', ['rev-parse', '--abbrev-ref', 'HEAD'], { encoding: 'utf8' }).trim()

const runScanner = (sonarToken, cwd, branch) =>
  new Promise((resolve, reject) => {
    const args = [
      'run',
      '--rm',
      '--name',
      'sonar-scan',
      '-v',
      `${cwd}:/usr/src`,
      '-e',
      `SONAR_TOKEN=${sonarToken}`,
      'sonarsource/sonar-scanner-cli',
      '-Dsonar.issuesReport.console.enable=true',
      '-Dsonar.qualitygate.wait=true',
      `-Dsonar.branch.name=${branch}`,
      '-Dsonar.verbose=true'
    ]

    const child = spawn('docker', args, { stdio: 'inherit' })

    child.on('error', reject)
    // Always resolve with the exit code — a non-zero exit may simply mean the
    // quality gate failed (analysis was still uploaded). We check the gate
    // status via the API after the scan and exit accordingly.
    child.on('close', resolve)
  })

const sonarcloudFetch = async (path, sonarToken) => {
  const url = `${SONARCLOUD_BASE_URL}${path}`

  const response = await fetch(url, {
    headers: {
      Authorization: `Bearer ${sonarToken}`
    }
  })

  if (!response.ok) {
    throw new Error(`SonarCloud API error ${response.status}: ${response.statusText} (${url})`)
  }

  return response.json()
}

const fetchQualityGate = (projectKey, sonarToken, branch) =>
  sonarcloudFetch(
    `/api/qualitygates/project_status?projectKey=${encodeURIComponent(projectKey)}&branch=${encodeURIComponent(branch)}`,
    sonarToken
  )

const fetchMeasures = (projectKey, sonarToken, branch) =>
  sonarcloudFetch(
    `/api/measures/component?component=${encodeURIComponent(projectKey)}&branch=${encodeURIComponent(branch)}&metricKeys=new_violations,accepted_issues,security_hotspots,new_coverage,new_duplicated_lines_density`,
    sonarToken
  )

const fetchIssues = (projectKey, sonarToken, branch) =>
  sonarcloudFetch(
    `/api/issues/search?componentKeys=${encodeURIComponent(projectKey)}&branch=${encodeURIComponent(branch)}&resolved=false&inNewCodePeriod=true&ps=500&statuses=OPEN,CONFIRMED,REOPENED`,
    sonarToken
  )

const fetchSecurityHotspots = (projectKey, sonarToken, branch) =>
  sonarcloudFetch(
    `/api/hotspots/search?projectKey=${encodeURIComponent(projectKey)}&branch=${encodeURIComponent(branch)}&inNewCodePeriod=true&ps=500&status=TO_REVIEW`,
    sonarToken
  )

const getMeasureValue = (measures, key) => {
  const measure = measures.find((m) => m.metric === key)
  if (!measure) return 'N/A'

  return measure.value ?? measure.periods?.[0]?.value ?? 'N/A'
}

const formatPercent = (value) => (value === 'N/A' ? 'N/A' : `${parseFloat(value).toFixed(1)}%`)

const row = (label, value) => ` ${`  ${label}`.padEnd(28)}${value}`

const extractFilePath = (component, projectKey) => {
  const prefix = `${projectKey}:`
  return component.startsWith(prefix) ? component.slice(prefix.length) : component
}

const printFailedConditions = (qualityGate) => {
  const conditions = qualityGate.projectStatus?.conditions ?? []
  const failed = conditions.filter((c) => c.status === 'ERROR')

  if (failed.length === 0) return

  console.log(THIN_BORDER)
  console.log(' ⛔ Failed Conditions')

  for (const condition of failed) {
    const label = METRIC_LABELS[condition.metricKey] ?? condition.metricKey
    const comparator = COMPARATOR_SYMBOLS[condition.comparator] ?? condition.comparator

    const actual = condition.metricKey.includes('coverage') || condition.metricKey.includes('duplicat')
      ? formatPercent(condition.actualValue)
      : condition.actualValue

    const threshold = condition.metricKey.includes('coverage') || condition.metricKey.includes('duplicat')
      ? formatPercent(condition.errorThreshold)
      : condition.errorThreshold

    console.log(`    ${label}: ${actual} (threshold ${comparator} ${threshold})`)
  }
}

const printIssues = (issuesResponse, projectKey) => {
  const issues = issuesResponse?.issues ?? []
  const total = issuesResponse?.total ?? issues.length

  if (issues.length === 0) return

  // Sort by severity
  issues.sort((a, b) =>
    SEVERITY_ORDER.indexOf(a.severity) - SEVERITY_ORDER.indexOf(b.severity)
  )

  // Group by file
  const byFile = new Map()

  for (const issue of issues) {
    const filePath = extractFilePath(issue.component, projectKey)
    if (!byFile.has(filePath)) byFile.set(filePath, [])
    byFile.get(filePath).push(issue)
  }

  const issuesUrl = `${SONARCLOUD_BASE_URL}/project/issues?id=${encodeURIComponent(projectKey)}&resolved=false&inNewCodePeriod=true`

  console.log(`\n${BORDER}`)
  console.log(` 🐛 Issues (${total} total)`)
  console.log(BORDER)

  let displayed = 0

  for (const [filePath, fileIssues] of [...byFile.entries()].sort(([a], [b]) => a.localeCompare(b))) {
    if (displayed >= MAX_ISSUES_DISPLAYED) break

    console.log(`\n  📄 ${filePath}`)

    for (const issue of fileIssues) {
      if (displayed >= MAX_ISSUES_DISPLAYED) break

      const icon = SEVERITY_ICONS[issue.severity] ?? '⚪'
      const rule = issue.rule ? ` (${issue.rule})` : ''

      console.log(`    ${icon} L${issue.line ?? '?'} ${issue.message}${rule}`)

      const issueUrl = `${SONARCLOUD_BASE_URL}/project/issues?id=${encodeURIComponent(projectKey)}&open=${encodeURIComponent(issue.key)}`

      console.log(`       ${issueUrl}`)

      displayed++
    }
  }

  if (total > MAX_ISSUES_DISPLAYED) {
    console.log(`\n  ... and ${total - MAX_ISSUES_DISPLAYED} more`)
  }

  console.log(THIN_BORDER)
  console.log(` 🔗 ${issuesUrl}`)
  console.log(`${BORDER}\n`)
}

const printHotspots = (hotspotsResponse, projectKey) => {
  const hotspots = hotspotsResponse?.hotspots ?? []

  if (hotspots.length === 0) return

  const total = hotspotsResponse?.paging?.total ?? hotspots.length

  console.log(`\n${BORDER}`)
  console.log(` 🔥 Security Hotspots (${total} to review)`)
  console.log(BORDER)

  let displayed = 0

  for (const hotspot of hotspots) {
    if (displayed >= MAX_ISSUES_DISPLAYED) break

    const filePath = extractFilePath(hotspot.component, projectKey)
    const icon = HOTSPOT_ICONS[hotspot.vulnerabilityProbability] ?? '🟡'
    const probability = hotspot.vulnerabilityProbability ?? 'UNKNOWN'

    console.log(`\n  📄 ${filePath}`)
    console.log(`    ${icon} [${probability}] L${hotspot.line ?? '?'} ${hotspot.message}`)

    const hotspotUrl = `${SONARCLOUD_BASE_URL}/security_hotspots?id=${encodeURIComponent(projectKey)}&hotspots=${encodeURIComponent(hotspot.key)}`

    console.log(`       ${hotspotUrl}`)

    displayed++
  }

  if (total > MAX_ISSUES_DISPLAYED) {
    console.log(`\n  ... and ${total - MAX_ISSUES_DISPLAYED} more`)
  }

  const hotspotsUrl = `${SONARCLOUD_BASE_URL}/security_hotspots?id=${encodeURIComponent(projectKey)}&inNewCodePeriod=true`

  console.log(THIN_BORDER)
  console.log(` 🔗 ${hotspotsUrl}`)
  console.log(`${BORDER}\n`)
}

const printSummary = (qualityGate, measuresResponse, projectKey, branch) => {
  const measures = measuresResponse.component?.measures ?? []
  const status = qualityGate.projectStatus?.status

  const passed = status === 'OK'
  const statusLabel = passed ? '✅ PASSED' : status === 'WARN' ? '⚠️  WARN' : '❌ FAILED'

  // Issues
  const newIssues = getMeasureValue(measures, 'new_violations')
  const acceptedIssues = getMeasureValue(measures, 'accepted_issues')

  // Measures
  const securityHotspots = getMeasureValue(measures, 'security_hotspots')
  const coverageOnNew = formatPercent(getMeasureValue(measures, 'new_coverage'))
  const duplicationOnNew = formatPercent(getMeasureValue(measures, 'new_duplicated_lines_density'))

  const dashboardUrl = `${SONARCLOUD_BASE_URL}/summary/overall?id=${encodeURIComponent(projectKey)}`

  console.log(`\n${BORDER}`)
  console.log(` SonarCloud Quality Gate: ${statusLabel}`)
  console.log(BORDER)
  console.log(' Issues')
  console.log(row('New Issues:', newIssues))
  console.log(row('Accepted Issues:', acceptedIssues))
  console.log(' Measures')
  console.log(row('Security Hotspots:', securityHotspots))
  console.log(row('Coverage on New Code:', coverageOnNew))
  console.log(row('Duplication on New Code:', duplicationOnNew))

  if (!passed) {
    printFailedConditions(qualityGate)
  }

  console.log(BORDER)
  console.log(` 🔀 Branch: ${branch}`)
  console.log(` 🔗 ${dashboardUrl}`)
  console.log(`${BORDER}\n`)

  return passed || status === 'WARN'
}

const sonarScan = async () => {
  const cwd = resolve('.')

  // Load .env if present (mirrors `source .env` from the old npm script)
  const envPath = resolve(cwd, '.env')
  const envVars = existsSync(envPath) ? await parseDotEnv(envPath) : {}
  const sonarToken = envVars.SONAR_TOKEN ?? process.env.SONAR_TOKEN

  if (!sonarToken) {
    console.error(
      'Error: SONAR_TOKEN is not set. Add it to your .env file.'
    )

    process.exit(1)
  }

  // Read project config from sonar-project.properties
  const propsPath = resolve(cwd, 'sonar-project.properties')
  const props = await parseSonarProperties(propsPath)
  const projectKey = props['sonar.projectKey']

  if (!projectKey) {
    console.error('Error: sonar.projectKey not found in sonar-project.properties')
    process.exit(1)
  }

  // Detect current branch so the scan targets it on SonarCloud
  const branch = getCurrentBranch()
  console.log(`\n🔀 Branch: ${branch}\n`)

  // Run the scanner — resolves with exit code (0 = success, non-zero = quality
  // gate failed or scan error). We always attempt to fetch the summary.
  const scanCode = await runScanner(sonarToken, cwd, branch)

  // Fetch quality gate + metrics and print summary
  let qualityGate, measuresResponse

  try {
    ;[qualityGate, measuresResponse] = await Promise.all([
      fetchQualityGate(projectKey, sonarToken, branch),
      fetchMeasures(projectKey, sonarToken, branch)
    ])
  } catch (apiErr) {
    // API fetch failed — the scan likely didn't upload (e.g. auth error, network)
    if (scanCode !== 0) {
      console.error(`\nSonar scanner exited with code ${scanCode}. No results to display.`)
      process.exit(scanCode)
    }

    throw apiErr
  }

  const passed = printSummary(qualityGate, measuresResponse, projectKey, branch)

  if (!passed) {
    // Fetch detailed issues and hotspots to help developers fix problems locally
    try {
      const measures = measuresResponse.component?.measures ?? []
      const hotspotCount = getMeasureValue(measures, 'security_hotspots')
      const shouldFetchHotspots = hotspotCount !== 'N/A' && parseInt(hotspotCount, 10) > 0

      const fetches = [fetchIssues(projectKey, sonarToken, branch)]

      if (shouldFetchHotspots) {
        fetches.push(fetchSecurityHotspots(projectKey, sonarToken, branch))
      }

      const [issuesResponse, hotspotsResponse] = await Promise.all(fetches)

      printIssues(issuesResponse, projectKey)

      if (hotspotsResponse) {
        printHotspots(hotspotsResponse, projectKey)
      }
    } catch (detailErr) {
      console.error(`\nCould not fetch issue details: ${detailErr.message}`)
    }

    process.exit(1)
  }
}

sonarScan().catch((err) => {
  console.error(`\nSonar scan failed: ${err.message}`)
  process.exit(1)
})
```

### 7. Update CI workflow
In `.github/workflows/check-pull-request.yml`, find the SonarQube scan step and add these environment variables if not already present:
```yaml
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  SONAR_SCANNER_OPTS: "-Dsonar.verbose=true"
```

### 8. Update README.md
Add the following sections to the README (adapt to the service's existing structure):

**SonarQube Cloud token section** (under Prerequisites):
- Explain that `npm run sonar` enables local SonarQube Cloud scanning
- Link to SonarQube Cloud login, instruct user to generate a personal token under My Account → Security → Generate Tokens
- Tell user to add `SONAR_TOKEN` to their `.env`

**Pre-commit Hooks section**:
- Explain this repo includes pre-commit hooks: `detect-secrets` and `eslint-fix` (ESLint + neostandard with `--fix`)
- Note that committing via command line shows full hook output
- Inform that Python and its package manager pip need to be install and provide install instructions: `pip3 install pre-commit`

**VS Code tasks note**:
- Mention that VS Code users can access tasks via Command Palette → Tasks: Run Task
- Key combo: `Ctrl+Shift+P` (Windows) / `Cmd+Shift+P` (Mac)

**Update existing build/run/test sections** to reference the new npm scripts as alternatives to raw Docker Compose commands. Fix any references from `standard` to `neostandard` and from `test:lint` to `lint`.

### Important notes
- Do NOT modify any application source code, only tooling/config files
- Keep all existing `docker:test` and `docker:test:watch` scripts unchanged (they use service-specific project names and compose files)
- If a `compose.debug.yaml` already exists, merge rather than overwrite
- If `.vscode/tasks.json` already exists, merge the new tasks in
- All file contents should use ES module syntax and match the project's existing formatting style
- The service name used in Docker compose project names (`-p` flag) and `compose.debug.yaml` must match what's in the existing `compose.yaml`

