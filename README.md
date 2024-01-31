# Sample Integration of Mobb with Snyk SAST scan in CircleCI 
This is a sample integration that demonstrates the use of Mobb CLI (Bugsy) in a CI/CD environment. The scenario demonstrates a developer performing a pull request on a GitHub repository, which triggers a CircleCI job. During the CircleCI Job, a Snyk SAST scan will run followed by Mobb analysis on the scan result. 

# Usage

## Register

To perform this integration, you will need the following:

* Sign up for a free account at https://mobb.ai
* Sign up for a free Snyk Account https://snyk.io 
* Sign up for a free CircleCI Account https://circleci.com/signup/
* Access to a GitHub repository to perform this integration

## GitHub Personal Access Token (PAT)

The first step is to generate a GitHub Personal Access Token (PAT). You can generate one by clicking on your profile -> Settings -> Developer Settings -> Personal Access Tokens -> Fine-grained tokens. 

For this integration, we need to provide the following permissions: 

* Commit Statuses - Read and Write
* Contents - Read and Write
* Pull Requests - Read and Write
* Access to the repository where this integration will be performed

Note down the generated GitHub PAT and store it in a safe place. 

## Generate Mobb API Key and Snyk API Key

After logging into the Mobb portal, click on the "Settings" icon on the bottom left, then select "Access tokens". From here, you can generate an API key by selecting the "Add API Key" button.

To integrate with Snyk, you  will also need to generate a Snyk API Key. This can be achieved by following this [guide](https://docs.snyk.io/snyk-api/authentication-for-api) from Snyk documentation. 

## Loading Credentials into the CircleCI Project

At this point, we should have 3 API keys ready: 
* GitHub Personal Access Token (PAT)
* Mobb API Key
* Snyk API Key

The next step is to load them into CircleCI. To do so, go to your CircleCI project, under "Project Settings", go to "Environment Variables". The variable names we are using in this example are as follows:
```
GITHUB_PAT_SECRET
MOBB_API_KEY
SNYK_API_KEY
```

By the time you have loaded the 3 entries, you should have something similar to this:
![image](https://github.com/antonychiu2/mobb-circleci-integration/assets/5158535/d4925800-71d7-47a0-89ae-676568686984)

## Optional - Build only pull requests

By default, CircleCI will build all the commits in the project. However, you may want to only build branches that have associated pull requests open. If you only want the build to run when there is a pull request, make sure to go to "Project Settings" -> "Advanced" and enable "Only build pull requests". 

![image](https://github.com/antonychiu2/mobb-circleci-integration/assets/5158535/0e6bb3a8-ff22-4896-bb4a-7a5a260a8328)


## CircleCI 

In this demo, we are checking in the build script into the source code repository under the path `.circleci/config.yml`. Here is a sample `yaml` script for your reference:

``` yaml
version: 2.1
orbs:
  node: circleci/node@5.2.0
jobs:
  sast-autofixer:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      # Installing Node.js 18
      - node/install:
          node-version: '18'
      - run:
          name: "Prepare Environment for SAST and Mobb Steps"
          command: |
            # Extract the GitHub repo URL
            GITHUBURL=$(echo $CIRCLE_REPOSITORY_URL | sed 's/^git@github.com:/https:\/\/github.com\//;s/\.git$//')
            echo "Extracted GitHub URL: $GITHUBURL"
            echo "export GITHUBURL="$GITHUBURL"" >> $BASH_ENV
      - run:
          name: "SAST scan"
          command: |
            npx snyk auth $SNYK_API_KEY
            npx snyk code test --sarif-file-output=report.json
      - run:
          name: "Mobb Autofixer"
          command: |
            # Run Mobb Autofixer against the report.json file generated in the previous step with Snyk SAST scan
            MOBBURL=$(npx mobbdev@latest analyze -f report.json -r $GITHUBURL --ref $CIRCLE_BRANCH --api-key $MOBB_API_KEY --ci)
            echo "Mobb URL: $MOBBURL"
            # Store the Mobb URL in a html file and save it as a CircleCI artifact
            echo "<html><body><a href="$MOBBURL">Click here for the Mobb URL</a></body></html>" > mobburl.html
            ./.circleci/update_github_status.sh \
              "$GITHUB_PAT_SECRET" \
              "$CIRCLE_USERNAME" \
              "$CIRCLE_PROJECT_REPONAME" \
              "$CIRCLE_SHA1" \
              "success" \
              "$MOBBURL" \
              "Click on \\\"Details\\\" to access the Mobb Fix Link" \
              "Mobb Fix Link"

          when: on_fail
      - store_artifacts:
          path: mobburl.html
          destination: /MobbURL
      - store_artifacts:
          path: report.json
          destination: /Snyk Report
# Orchestrate jobs using workflows
workflows:
  test-workflow:
    jobs:
      - sast-autofixer

```

## Triggering the pipeline

The CircleCI job will run when a pull request is detected in the GitHub Source Code repository that it is connected with. To test this, go to your GitHub and trigger a Pull Request by making some updates to your source code in a new branch. 
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/171bad00-c5c0-4bb1-89c5-fc291e63d3b8)

Once the Pull Request is initiated, the job in CircleCI will initiate. 

If vulnerabilities are found by the SAST scanner, Mobb will also run to consume the results of the SAST scan. Once the analysis is ready, a URL to the Mobb dashboard will be provided via the "Details" button. 
![image](https://github.com/antonychiu2/mobb-circleci-integration/assets/5158535/c4478a69-6c22-4c49-8175-0ad373b1d2a7)

Once we arrive at the analysis page for the project, we can see a list of available fixes. Let's click on the "Link to Fix" button next to the XSS finding.
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/11248919-19ec-456d-bfc0-0caba74a21db)

Mobb provides a powerful self-guided remediation engine. As a developer, all you have to do is answer a few questions and validate the fix that Mobb is proposing. From there, Mobb will take over the remediation process and commit the code on your behalf.

Once you're ready, select the "Commit Changes" button.
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/2454da9e-b1cb-4b01-bf55-537389d994e6)

As the last step, enter the name of the target branch where this merge request will be merged. And select "Commit Changes".
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/03544f61-681c-4b21-8566-fcd4739afa06)

Mobb has successfully committed the remediated code back to your repository under a new branch along with a new Pull Request. Since this pipeline is configured to run on every Pull Request events, a new SAST scan will be conducted to validate the proposed changes to ensure the vulnerabilities have been remediated.
![image](https://github.com/antonychiu2/mobb-circleci-integration/assets/5158535/0b1da58e-f9ba-4c17-897d-ed665588ded0)



