# Copilot Workbench

An archive of skills, prompts, and agents created by and for GitHub Copilot.

## Available Resources

| Resource | Copilot Type | Purpose | When to Use | Additional resources |
| --- | --- | --- | --- |
| [sonar-qube-cloud-fix](.github/agents/sonar-qube-cloud-fix.agent.md) | Agent | Runs the SonarQube Cloud local code scan or analyses existing SonarQube Cloud scan results from a pull request, parses the output, and implements fixes for all identified code quality and security issues. | A SonarQube Cloud scan fails, a quality gate is failing, or there are open bugs, vulnerabilities, code smells, or security hotspots to resolve. | Pairs well with the [`sonar-scan`](./resources/scripts/sonar-scan.js) script which enables running the [SonarScanner CLI](https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/scanners/sonarscanner) locally. |
