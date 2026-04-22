# Copilot Workbench

An archive of skills, prompts, and agents created by and for GitHub Copilot.

## Available Resources

| Resource | Copilot Type | Purpose | When to Use | Additional resources |  
| --- | --- | --- | --- | --- |
| [sonarqube-cloud](.github/agents/sonarqube-cloud.agent.md) | Agent | Runs a local SonarQube Cloud code scan or analyses existing SonarQube Cloud scan results from a pull request, parses the output, and implements fixes for all identified code quality and security issues. | A SonarQube Cloud scan fails, a quality gate is failing, or there are open bugs, vulnerabilities, code smells, or security hotspots to resolve. | Intended to be paired with the [`sonar-scan`](./resources/scripts/sonar-scan.js) script which enables running the [SonarScanner CLI](https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/scanners/sonarscanner) locally. |
| [audit-ecs-logging-compliance](./prompts/audit-ecs-logging-compliance.prompt.md) | Prompt | Scans entire codebase of any CDP-hosted (a.k.a. Defra's Core Delivery Platform) service. Compares all calls on the CDP logger to CDP's approved ECS (Elastic Common Schema). Violations are identified by file name and line number (down to the code snippet) and listed in order of severity to streamline prioritisation for developers. | To audit use of the CDP logger on any CDP-hosted service to ensure full compliance. | TBC |
