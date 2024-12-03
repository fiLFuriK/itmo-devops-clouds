# Third laboratory work on the subject "Cloud technologies and services"

## Assignment

1. Write a "bad" CI/CD configuration file that works but includes at least five "bad practices."
2. Write a "good" CI/CD configuration file that fixes these bad practices.
3. In this README, describe each of the bad practices in the bad CI/CD file, explain why they are considered bad, and how they were corrected in the good CI/CD file. Also, explain how the corrections improve the overall result.

## What is CI/CD?

CI/CD stands for Continuous Integration and Continuous Delivery. It’s a methodology that allows code changes to be integrated and deployed frequently in small increments. This approach helps identify issues earlier and enables automated processes for building, testing, and deploying applications. CI/CD automates these tasks to ensure that code is always production-ready.

## Bad CI/CD Example

```
name: Bad CI/CD

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Install deps
        run: npm install

      - name: Test
        run: npm run test

      - name: Build
        run: npm run build

      - name: Deploy
        run: |
           rsync -avz --delete ./dist/ user@server_ip:/var/www/html/
```

### Identified Bad Practices:

1. Using ubuntu-latest as a runner:
   The latest tag automatically uses the newest available Ubuntu image, which can change over time. This can lead to compatibility issues and unexpected behavior as the environment evolves.

2. No error handling:
   There is no mechanism in place to stop the pipeline if one of the commands fails. This could result in a failed build being deployed to production, which would introduce broken code.

3. No caching of dependencies:
   Without caching, dependencies are re-installed from scratch every time the pipeline runs. This increases build times and resource usage unnecessarily.

4. Using npm install:
   npm install can update package-lock.json if newer versions of dependencies are available. This can lead to inconsistent builds and hard-to-debug issues, as each build might use different dependency versions.

5. Combining build, test, and deploy steps in a single job:
   Tying together different stages (build, test, deploy) in a single job makes it hard to manage and debug individual steps. This reduces flexibility and makes the pipeline more difficult to maintain.

6. Hardcoding server credentials:
   The server_ip and other credentials are hardcoded in the pipeline script. This is a security risk as sensitive information is exposed in the code, making it accessible to anyone with access to the repository.

## Good CI/CD Example

```
name: Good CI/CD

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Cache dependencies
        uses: actions/cache@v3
        id: cache-id
        with:
          cache: npm-dependencies
          path: ~/.npm
          key: ${{ runner.OS }}-npm-cache-${{ hashFiles('**/package-lock.json') }}

      - name: Install dependencies
        if: steps.cache-id.outputs.cache-hit != 'true'
        run: npm ci

      - name: Build
        run: npm run build

  test:
    runs-on: ubuntu-20.04
    needs: build
    steps:
      - name: Run tests
          run: npm run test

  deploy:
    runs-on: ubuntu-20.04
    needs: test
    steps:
      - name: Deploy
        env:
          SERVER_IP: ${{ secrets.SERVER_IP }}
          USERNAME: ${{ secrets.USERNAME }}
          DEPLOYMENT_PATH: ${{ secrets.DEPLOYMENT_PATH }}
        run: |
          rsync -avz --delete ./dist/ $USERNAME@$SERVER_IP:$DEPLOYMENT_PATH
```

### Improvements in the Good CI/CD:

1. Version-controlled runner environment:
   Instead of using ubuntu-latest, the runner is set to ubuntu-20.04, ensuring a consistent and stable environment. This prevents unexpected changes when new versions of Ubuntu are released.

2. Error handling through job dependencies:
   The pipeline is split into separate jobs for build, test, and deploy. The test job depends on the success of the build job, and the deploy job depends on the success of the test job. This guarantees that a failed build or test will prevent deployment.

3. Caching npm dependencies:
   Caching is introduced for npm dependencies to speed up builds. This reduces the time and resources required by restoring dependencies from cache, avoiding the need to reinstall them each time.

4. Using npm ci for clean installs:
   The npm ci command ensures a clean and reproducible installation of dependencies. Unlike npm install, it installs exactly the versions listed in package-lock.json without updating them, making builds consistent and predictable.

5. Separation of concerns (build, test, deploy):
   By breaking down the pipeline into separate jobs for building, testing, and deploying, each step can be handled independently. This makes the pipeline easier to maintain, debug, and optimize. Failures in one job won’t affect others unnecessarily.

6. Secure handling of credentials:
   Server credentials (SERVER_IP, USERNAME, DEPLOYMENT_PATH) are stored as GitHub secrets and accessed securely during deployment. Secrets are encrypted and not exposed in the code or logs, ensuring better security for sensitive information.

## GitHub Actions


<p style="text-align:center;"><img src="./pipline (test).png" width=600></p>

