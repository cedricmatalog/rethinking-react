# Chapter 17: CI/CD for React Apps

## Introduction

Junior developers manually build and deploy applications, crossing their fingers each time. Senior developers automate the entire pipeline, ensuring consistent, reliable deployments with automated testing, quality checks, and rollback capabilities.

## Learning Objectives

- Set up automated CI/CD pipelines
- Configure GitHub Actions for React apps
- Implement automated testing in CI
- Deploy to multiple environments
- Manage environment variables securely
- Implement deployment strategies (blue-green, canary)
- Set up automated code quality checks
- Handle database migrations and rollbacks

## 17.1 GitHub Actions Fundamentals

### Basic CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run type check
        run: npm run type-check

      - name: Run tests
        run: npm run test:ci

      - name: Build
        run: npm run build

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
```

### Optimizing CI Performance

```yaml
# .github/workflows/ci-optimized.yml
name: CI (Optimized)

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      # Cache node_modules for faster installs
      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Install dependencies
        run: npm ci --prefer-offline

      # Run jobs in parallel
      - name: Lint & Type Check & Test
        run: |
          npm run lint &
          npm run type-check &
          npm run test:ci &
          wait

      - name: Build
        run: npm run build

      # Cache build artifacts
      - name: Cache build
        uses: actions/cache@v3
        with:
          path: dist
          key: ${{ runner.os }}-build-${{ github.sha }}
```

### Running Tests in Parallel

```yaml
# .github/workflows/parallel-tests.yml
name: Parallel Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:integration

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run test:e2e

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
```

## 17.2 Automated Code Quality Checks

### ESLint and Prettier in CI

```yaml
# .github/workflows/code-quality.yml
name: Code Quality

on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - run: npm ci

      - name: Run ESLint
        run: npm run lint -- --format json --output-file eslint-report.json
        continue-on-error: true

      - name: Annotate code
        uses: ataylorme/eslint-annotate-action@v2
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
          report-json: eslint-report.json

      - name: Check formatting
        run: npm run format:check

  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - run: npm ci
      - run: npm run type-check
```

### Bundle Size Checking

```yaml
# .github/workflows/bundle-size.yml
name: Bundle Size Check

on: [pull_request]

jobs:
  bundle-size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - name: Check bundle size
        uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          build_script: build
          skip_step: build
```

### Security Scanning

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0' # Weekly on Sunday

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        run: npm audit --audit-level=moderate

      - name: Snyk Security Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'my-react-app'
          path: '.'
          format: 'HTML'

      - name: Upload security report
        uses: actions/upload-artifact@v3
        with:
          name: security-report
          path: reports/
```

## 17.3 Deployment Workflows

### Deploy to Vercel

```yaml
# .github/workflows/deploy-vercel.yml
name: Deploy to Vercel

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'

      - name: Comment PR
        uses: actions/github-script@v6
        if: github.event_name == 'pull_request'
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ Deployed to Vercel: ${{ steps.deploy.outputs.preview-url }}'
            })
```

### Deploy to Netlify

```yaml
# .github/workflows/deploy-netlify.yml
name: Deploy to Netlify

on:
  push:
    branches: [main, develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - name: Install and build
        run: |
          npm ci
          npm run build

      - name: Deploy to Netlify
        uses: nwtgck/actions-netlify@v2
        with:
          publish-dir: './dist'
          production-branch: main
          github-token: ${{ secrets.GITHUB_TOKEN }}
          deploy-message: 'Deploy from GitHub Actions'
          enable-pull-request-comment: true
          enable-commit-comment: true
        env:
          NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}
        timeout-minutes: 10
```

### Deploy to AWS S3 + CloudFront

```yaml
# .github/workflows/deploy-aws.yml
name: Deploy to AWS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - name: Install and build
        run: |
          npm ci
          npm run build
        env:
          VITE_API_URL: ${{ secrets.PROD_API_URL }}
          VITE_APP_ENV: production

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy to S3
        run: |
          aws s3 sync dist/ s3://${{ secrets.S3_BUCKET }} \
            --delete \
            --cache-control max-age=31536000,public

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_ID }} \
            --paths "/*"

      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Deployed to production'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        if: always()
```

## 17.4 Multi-Environment Deployments

### Environment-Based Workflows

```yaml
# .github/workflows/deploy-multi-env.yml
name: Multi-Environment Deploy

on:
  push:
    branches:
      - main
      - develop
      - staging

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - name: Set environment
        id: set-env
        run: |
          if [[ $GITHUB_REF == 'refs/heads/main' ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
            echo "api_url=${{ secrets.PROD_API_URL }}" >> $GITHUB_OUTPUT
            echo "s3_bucket=${{ secrets.PROD_S3_BUCKET }}" >> $GITHUB_OUTPUT
          elif [[ $GITHUB_REF == 'refs/heads/staging' ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
            echo "api_url=${{ secrets.STAGING_API_URL }}" >> $GITHUB_OUTPUT
            echo "s3_bucket=${{ secrets.STAGING_S3_BUCKET }}" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
            echo "api_url=${{ secrets.DEV_API_URL }}" >> $GITHUB_OUTPUT
            echo "s3_bucket=${{ secrets.DEV_S3_BUCKET }}" >> $GITHUB_OUTPUT
          fi

      - name: Build
        run: npm ci && npm run build
        env:
          VITE_API_URL: ${{ steps.set-env.outputs.api_url }}
          VITE_APP_ENV: ${{ steps.set-env.outputs.environment }}

      - name: Deploy
        run: |
          aws s3 sync dist/ s3://${{ steps.set-env.outputs.s3_bucket }} --delete
```

### Using GitHub Environments

```yaml
# .github/workflows/deploy-environments.yml
name: Deploy with Environments

on:
  push:
    branches: [main, staging]

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/staging'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: npm run deploy:staging
        env:
          API_URL: ${{ secrets.STAGING_API_URL }}

  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: npm run deploy:prod
        env:
          API_URL: ${{ secrets.PROD_API_URL }}
```

## 17.5 Advanced Deployment Strategies

### Blue-Green Deployment

```yaml
# .github/workflows/blue-green-deploy.yml
name: Blue-Green Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: |
          npm ci
          npm run build

      - name: Get current environment
        id: current-env
        run: |
          CURRENT=$(aws elbv2 describe-target-groups \
            --names prod-tg \
            --query 'TargetGroups[0].Tags[?Key==`Environment`].Value' \
            --output text)

          if [ "$CURRENT" == "blue" ]; then
            echo "active=blue" >> $GITHUB_OUTPUT
            echo "inactive=green" >> $GITHUB_OUTPUT
          else
            echo "active=green" >> $GITHUB_OUTPUT
            echo "inactive=blue" >> $GITHUB_OUTPUT
          fi

      - name: Deploy to inactive environment
        run: |
          aws s3 sync dist/ s3://myapp-${{ steps.current-env.outputs.inactive }}

      - name: Health check
        run: |
          for i in {1..30}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
              https://${{ steps.current-env.outputs.inactive }}.myapp.com/health)
            if [ $STATUS -eq 200 ]; then
              echo "Health check passed"
              exit 0
            fi
            sleep 10
          done
          echo "Health check failed"
          exit 1

      - name: Switch traffic
        run: |
          aws elbv2 modify-target-group \
            --target-group-arn ${{ secrets.TARGET_GROUP_ARN }} \
            --tags Key=Environment,Value=${{ steps.current-env.outputs.inactive }}

      - name: Rollback on failure
        if: failure()
        run: |
          echo "Deployment failed, traffic remains on ${{ steps.current-env.outputs.active }}"
```

### Canary Deployment

```yaml
# .github/workflows/canary-deploy.yml
name: Canary Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and deploy canary
        run: |
          npm ci
          npm run build
          aws s3 sync dist/ s3://myapp-canary

      - name: Route 10% traffic to canary
        run: |
          aws appmesh update-route \
            --mesh-name my-mesh \
            --route-name my-route \
            --spec '{
              "httpRoute": {
                "action": {
                  "weightedTargets": [
                    {"virtualNode": "prod", "weight": 90},
                    {"virtualNode": "canary", "weight": 10}
                  ]
                }
              }
            }'

      - name: Monitor canary (15 minutes)
        run: |
          sleep 900
          ERROR_RATE=$(aws cloudwatch get-metric-statistics \
            --namespace AWS/ApplicationELB \
            --metric-name HTTPCode_Target_5XX_Count \
            --dimensions Name=TargetGroup,Value=canary \
            --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 900 \
            --statistics Sum \
            --query 'Datapoints[0].Sum' \
            --output text)

          if [ "$ERROR_RATE" -gt "10" ]; then
            echo "Canary failed - error rate too high"
            exit 1
          fi

      - name: Promote canary to production
        run: |
          aws appmesh update-route \
            --mesh-name my-mesh \
            --route-name my-route \
            --spec '{
              "httpRoute": {
                "action": {
                  "weightedTargets": [
                    {"virtualNode": "canary", "weight": 100}
                  ]
                }
              }
            }'

      - name: Rollback canary on failure
        if: failure()
        run: |
          aws appmesh update-route \
            --mesh-name my-mesh \
            --route-name my-route \
            --spec '{
              "httpRoute": {
                "action": {
                  "weightedTargets": [
                    {"virtualNode": "prod", "weight": 100}
                  ]
                }
              }
            }'
```

## 17.6 GitLab CI/CD

### Basic GitLab Pipeline

```yaml
# .gitlab-ci.yml
image: node:20

stages:
  - install
  - test
  - build
  - deploy

cache:
  paths:
    - node_modules/

install:
  stage: install
  script:
    - npm ci
  artifacts:
    paths:
      - node_modules/
    expire_in: 1 hour

test:unit:
  stage: test
  script:
    - npm run test:unit
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

test:e2e:
  stage: test
  image: mcr.microsoft.com/playwright:v1.40.0
  script:
    - npx playwright install
    - npm run test:e2e
  artifacts:
    when: always
    paths:
      - playwright-report/
    expire_in: 7 days

lint:
  stage: test
  script:
    - npm run lint
    - npm run type-check

build:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

deploy:staging:
  stage: deploy
  script:
    - npm run deploy:staging
  environment:
    name: staging
    url: https://staging.myapp.com
  only:
    - develop

deploy:production:
  stage: deploy
  script:
    - npm run deploy:prod
  environment:
    name: production
    url: https://myapp.com
  when: manual
  only:
    - main
```

## 17.7 Automated Rollbacks

### Rollback on Error Rate Spike

```yaml
# .github/workflows/deploy-with-rollback.yml
name: Deploy with Automatic Rollback

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Get previous deployment
        id: prev-deploy
        run: |
          PREV_VERSION=$(aws s3api list-object-versions \
            --bucket myapp-prod \
            --prefix index.html \
            --query 'Versions[1].VersionId' \
            --output text)
          echo "version=$PREV_VERSION" >> $GITHUB_OUTPUT

      - name: Deploy new version
        id: deploy
        run: |
          npm ci
          npm run build
          aws s3 sync dist/ s3://myapp-prod

      - name: Monitor deployment (5 minutes)
        id: monitor
        run: |
          sleep 300

          ERROR_COUNT=$(aws cloudwatch get-metric-statistics \
            --namespace AWS/ApplicationELB \
            --metric-name HTTPCode_Target_5XX_Count \
            --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 300 \
            --statistics Sum \
            --query 'Datapoints[0].Sum' \
            --output text)

          if [ "$ERROR_COUNT" -gt "50" ]; then
            echo "Error rate too high: $ERROR_COUNT errors"
            exit 1
          fi

      - name: Rollback on failure
        if: failure()
        run: |
          echo "Rolling back to version ${{ steps.prev-deploy.outputs.version }}"
          aws s3api copy-object \
            --bucket myapp-prod \
            --copy-source myapp-prod/index.html?versionId=${{ steps.prev-deploy.outputs.version }} \
            --key index.html

      - name: Notify team
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          text: '🚨 Deployment failed and rolled back'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## 17.8 Database Migrations in CI/CD

### Running Migrations

```yaml
# .github/workflows/deploy-with-migrations.yml
name: Deploy with Database Migrations

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run database migrations
        run: |
          npm run db:migrate
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: Create migration backup
        run: |
          pg_dump ${{ secrets.DATABASE_URL }} > backup-$(date +%s).sql
          aws s3 cp backup-*.sql s3://myapp-backups/

      - name: Deploy application
        run: |
          npm ci
          npm run build
          npm run deploy

      - name: Rollback migrations on failure
        if: failure()
        run: |
          npm run db:migrate:rollback
```

## Real-World Scenario: Setting Up CI/CD from Scratch

### The Challenge

New project needs:
- Automated testing
- Multi-environment deployment
- Security scanning
- Performance monitoring
- Automatic rollbacks

### Senior Approach

```yaml
# Complete production-ready pipeline
# .github/workflows/production.yml
name: Production Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Quality checks
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run format:check

  # Security scan
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # Tests
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - run: npm ci
      - run: npm run test:ci
      - uses: codecov/codecov-action@v3

  # E2E tests
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run test:e2e

  # Build and deploy
  deploy:
    needs: [quality, security, test, e2e]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'

      - name: Build
        run: |
          npm ci
          npm run build
        env:
          VITE_API_URL: ${{ secrets.PROD_API_URL }}

      - name: Check bundle size
        uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}

      - name: Deploy to production
        run: npm run deploy:prod

      - name: Monitor deployment
        run: npm run monitor:deployment

      - name: Notify team
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## Chapter Exercise: Build Complete CI/CD Pipeline

Create a production-ready CI/CD pipeline:

**Requirements:**
1. Automated testing (unit, integration, E2E)
2. Code quality checks (lint, type-check, format)
3. Security scanning
4. Multi-environment deployment
5. Bundle size monitoring
6. Automated rollback capability
7. Deployment notifications
8. Performance monitoring

**Bonus:**
- Blue-green or canary deployment
- Database migration handling
- Preview deployments for PRs

## Review Checklist

- [ ] CI runs on every PR
- [ ] All tests automated
- [ ] Code quality checks pass
- [ ] Security scanning enabled
- [ ] Multi-environment setup
- [ ] Secrets properly managed
- [ ] Deployment notifications configured
- [ ] Rollback strategy in place
- [ ] Performance monitoring
- [ ] Documentation updated

## Key Takeaways

1. **Automate everything** - Manual deployments are error-prone
2. **Test before deploy** - Catch issues early
3. **Monitor deployments** - Detect problems quickly
4. **Enable rollbacks** - Have a safety net
5. **Use environments** - Staging catches production issues
6. **Secure secrets** - Never commit credentials
7. **Notify on failures** - Quick response is critical

## Further Reading

- GitHub Actions documentation
- GitLab CI/CD guide
- AWS deployment strategies
- Continuous Delivery by Jez Humble
- The DevOps Handbook

## Next Chapter

[Chapter 18: Monitoring & Observability](./18-monitoring-observability.md)
