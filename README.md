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

## Jenkins - Credentials

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

## Jenkins - Plugins

This integration makes use of the following Jenkins plugins, please ensure you have these installed in your Jenkins environment:
* [GitHub Plugin](https://plugins.jenkins.io/github)
* [GitHub Pull Request Builder](https://plugins.jenkins.io/ghprb)
* [NodeJS](https://plugins.jenkins.io/nodejs/)


## Jenkins - Configure Plugins Credentials
The next step is to configure the plugins to ensure they are using the crednentials we've provided. Go to "Dashboard" -> "Manage Jenkins" -> "System". 

First, locate the section called **GitHub**. Provide a Name to the connection. Under the "Credentials" section, click on the drop-down and select your GitHub Credential. 

Make sure to click on the "Test Connection" to ensure Jenkins is able to access your GitHub account using the provided credentials. 
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/0b6a871d-df33-4c54-8597-57c86cbbebdd)

Second, locate the section called **GitHub Pull Request Builder**. Similar to the previous step, select your GitHub credential. Under the "Shared Secret" section, enter your GitHub Personal Access Token. Make sure to test the connection again to verify that Jenkins is able to connect to your GitHub account using the supplied credentials. 

![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/027d7efb-24cf-4a15-9ec8-04efafe0cf27)

## GitHub - Creating a Webhook

The next step is to configure Webhook for GitHub to publish events to Jenkins. Specifically, we want to trigger the Jenkins Job when there is a pull request to initiate the SAST Scan and trigger Mobb analyze to parse the result of the SAST scan. 

> [!NOTE]
> GitHub must have network connectivity to your Jenkins instance in order to perform this step. Speak to your network administrator on setting up this connection. For ease of demo, this particular sample integration was built by exposing Jenkins to GitHub Cloud via a secure tunnel using [Cloudflare Zerotrust Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/). 
 

To setup the Webhook, first go to your GitHub repository. Select "Settings" -> "Webhooks". 
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/c11e183a-d84e-4586-aab2-050d36afce9b)

For the Payload URL, you want to specify your Jenkins URL in the format:
`https:\\<JENKINS_DOMAIN>:<JENKINS_PORT>/ghprbhook/`

For Content Type, select `application/json`

For events to trigger the webhook, select "Let me select individual events". Under the event list, select "Pull requests". 
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/5597944b-a119-4d6b-8cd5-73debfa7af1b)

To verify that the webhook is able to connect to Jenkins, go to "Recent Deliveries" tab. Verify that there is a green checkmark next to your most recent request. 
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/62496a31-5a96-4ce8-becc-5ef0efa05d6b)

We are finally done with the infrastructure-related configurations. Next, we will move onto configuring the actual pipeline. 

## Jenkins - Creating the pipeline

We will create the pipeline using Jenkins "Pipeline" project type.  In your Jenkins Dashboard, click on "New Item" and select "Pipeline". 
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/143a7280-3df1-47a1-9020-10d75f346795)

In the Pipeline configuration page, go to "GitHub Project". Provide your GitHub project URL here. 
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/e0b6a48a-050e-45fd-b8c8-09d461d598b2)

Next, go to "Build Triggers", and locate "GitHub Pull Request Builder". 

For the GitHub API Credentials, click on the drop down to select your GitHub credential. 

![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/edd0f2c8-ab2b-4653-a164-9c2ff35309af)

Under the "GitHub Pull Request Builder" -> **Advanced** section, the following values were used:

|Name  |Value |Explanation |
| ----------------- | --------------- | -- |
|Crontab line  | * * * * * | This value will cause Jenkins to check for Webhook every minute. Modify this as needed. |
| White list  | your GitHub username goes here | |

Under the "GitHub Pull Request Builder" -> **Trigger Setup** section, the following values were used:
|Name  |Value |Explanation |
| ----------------- | --------------- | -- |
|Commit Status Context  | Jenkins Pipeline | This is what appears in GitHub commit status message |

Under the "GitHub Pull Request Builder" -> Trigger Setup -> **Commit Status Build Result** section, add 2 sections with the following values

For the first section:
|Name  |Value |Explanation |
| ----------------- | --------------- | -- |
|Build Result  | Success | during a successful build event  |
|Message  | SAST Scan Complete - No issues found | This is what appears in GitHub commit status message |

For the second section:
|Name  |Value |Explanation |
| ----------------- | --------------- | -- |
|Build Result  | Failure |  |
|Message  | Build failed. Click on "Details" for more info.  | This is what appears in GitHub commit status message during a fail build event |

![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/aebd6f10-ec70-471d-b787-264c6360a823)

## Jenkins - Creating the pipeline - Pipeline Script

We can now supply our Pipeline script:

``` groovy
def MOBBURL

pipeline {
    agent any
    // Setting up environment variables
    environment {
        MOBB_API_KEY = credentials('MOBB_API_KEY')
        SNYK_API_KEY = credentials('SNYK_API_KEY')
        GITHUBREPOURL = 'https://github.com/antonychiu2/testrepo' //change this to your GitHub Repository URL
    }
    tools {
        nodejs 'NodeJS'
    }
    stages {
        // Checkout the source code from the branch being committed
        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '$ghprbActualCommit']], 
                    extensions: [], 
                    userRemoteConfigs: [[
                        credentialsId: '2760a171-4592-4fe0-84da-2c2f561c8c88', 
                        refspec: '+refs/pull/*:refs/remotes/origin/pr/*', 
                        url: "${GITHUBREPOURL}"]]
                        )

            }
        }
        // Run SAST scan
        stage('SAST') {
            steps {
                sh 'npx snyk auth $SNYK_API_KEY'
                sh 'npx snyk code test --sarif-file-output=report.json'
            }
        }
    }
    post {
        // If SAST scan complete with no issues found, pipeline is successful
        success {
            echo 'Pipeline succeeded!'
        }
        // If SAST scan complete WITH issues found, pipeline enters fail state, triggering Mobb autofix analysis
        failure {
            echo 'Pipeline failed!'

                script {
                    MOBBURL = sh(returnStdout: true,
                                script:'npx mobbdev@latest analyze -f report.json -r $GITHUBREPOURL --ref $ghprbSourceBranch --api-key $MOBB_API_KEY  --ci')
                                .trim()
                }     
            echo 'Mobb Fix Link: $MOBBURL'
            // Provide a "Mobb Fix Link" in the GitHub pull request page as a commit status
            step([$class: 'GitHubCommitStatusSetter', 
                    commitShaSource: [$class: 'ManuallyEnteredShaSource', sha: '$ghprbActualCommit'], 
                    contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: 'Mobb Fix Link'], 
                    reposSource: [$class: 'ManuallyEnteredRepositorySource', url: '$GITHUBREPOURL'], 
                    statusBackrefSource: [$class: 'ManuallyEnteredBackrefSource', backref: "${MOBBURL}"], 
                    statusResultSource: [$class: 'ConditionalStatusResultSource', 
                        results: [[$class: 'AnyBuildResult', message: 'Click on "Details" to access the Mobb Fix Link', state: 'SUCCESS']]]
            ])
        }
    }
}

```
Click on "Save" to save the pipeline project. 

## Triggering the pipeline

The jenkins job will run when a pull request is detected in the GitHub Source Code repository that it is connected with. To test this, go to your GitHub and trigger a Pull Request by making some updates to your source code. 

![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/171bad00-c5c0-4bb1-89c5-fc291e63d3b8)

Once the Pull Request is initiated, the job in Jenkins should trigger (how quickly the Jenkins job triggers is dependent on your Crontab setting defined in the "GitHub Pull Request Builder" trigger step. However, when triggered, you should be able to see updates in GitHub's Pull Request page to see status of the checks. 

If vulnerabilities are found by the SAST scanner, Mobb will also run to consume the results of the SAST scan. Once the analysis is ready, an URL will be provided to Mobb dashboard via the "Details" button. 

![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/0959c0bf-14d5-46f4-a7ab-ea8ef8d64361)

Once we arrive at the analysis page for the project, we can see a list of available fixes. Let's click on the "Link to Fix" button next to the XSS finding.

![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/11248919-19ec-456d-bfc0-0caba74a21db)

Mobb provides a powerful self-guided remediation engine. As a developer, all you have to do is answer a few questions and validate the fix that Mobb is proposing. From there, Mobb will take over the remediation process and commit the code on your behalf.

Once you're ready, select the "Commit Changes" button.

![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/2454da9e-b1cb-4b01-bf55-537389d994e6)

As the last step, enter the name of the target branch where this merge request will be merged. And select "Commit Changes".
![image](https://github.com/antonychiu2/jenkins-mobb-integration/assets/5158535/03544f61-681c-4b21-8566-fcd4739afa06)

Mobb has successfully committed the remediated code back to your repository under a new branch along with a new Pull Request. Since this pipeline is configured to run on every Pull Request events, a new SAST scan will be conducted to validate the proposed changes to ensure the vulnerabilities have been remediated.


