# DevSecOps Reference CI/CD Pipeline (Jenkins + SonarQube + Snyk + Trivy + Nexus + OWASP ZAP)

This repository demonstrates an end-to-end DevSecOps pipeline that enforces:
- Code quality gates (SonarQube)
- SAST/SCA (Snyk)
- Filesystem & container vulnerability scanning (Trivy)
- Secure container build and push to Nexus Repository
- DAST baseline testing (OWASP ZAP)
- Evidence reports archived as pipeline artifacts

## Pipeline Overview
1. Unit tests
2. SonarQube scan + quality gate
3. Snyk scans (dependencies + code)
4. Trivy filesystem scan
5. Build Docker image
6. Trivy image scan
7. Push image to Nexus
8. Run ephemeral container
9. OWASP ZAP baseline scan
10. Publish reports

## Security Gates (Fail Criteria)
- SonarQube quality gate must PASS
- Trivy HIGH/CRITICAL => fail pipeline
- Snyk HIGH severity dependency issues => fail pipeline
- ZAP => report published (can be switched to fail mode)

## Reports Generated
- snyk-report.json
- trivy-fs-report.json
- trivy-image-report.json
- zap-report.html / zap-report.json

## Jenkins Credentials Required
- sonar-token (Secret Text)
- snyk-token (Secret Text)
- nexus-docker-creds (Username/Password)

## How to Run Locally
Use docker-compose.local.yml to start SonarQube and Nexus.
