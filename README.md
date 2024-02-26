# How to: Integrate your SF org with Github using Jenkins (From the org to the GitHub pipeline).
## Script to copy organization's SF code changes to GitHub using Jenkins

Version 1. 26/02/2024.

0.  Prerequisites: You have configured the SF project using Package.xml with all the filters on the data you want to replicate.
1.	Install Jenkins: https://www.jenkins.io/ Default installations with default plugins.
2.	Install OpenSSL. It can be already installed if you use git bash on this PC. Just check by the command “which openssl”.
3.	Create a certificate: https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_key_and_cert.htm.
4.	Create a connected app in your Org. https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_connected_app.htm With JWT only options.
5.	Store the private key file (from step 3) as a Jenkins Secret File: https://www.jenkins.io/doc/book/using/using-credentials/.
6.	Go to http://<your _jenkins_site_and_port>/manage/configure and set global variables (Jenkins Environment Variable Set https://www.youtube.com/watch?v=gKB6ZiE02mw)
- GIT_TOKEN — your Git access token;
- SERVER_KEY_CREDENTALS_ID – secret file ID from step 5;
- SF_CONSUMER_KEY — from step 4;
- SF_INSTANCE_URL – your org full url like https://<your_org>.my.salesforce.com;
- SF_USERNAME – your Org login for the integration user.
7.	Create a new Jenkins Item with a free configuration.
8.	Go to http://<your _jenkins_site_and_port>/manage/configureTools/ and set a new custom tool called toolbelt with your sfdx location.

![Custom tool panel](/images/1.png)

9.	In the Pipeline section on this new Item set a script.

![Script panel](/images/2.png)

The script (you can read more about Jenkins scripts here: https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_ci_jenkins_sample_walkthrough.htm):


```
#!groovy

import groovy.json.JsonSlurperClassic

node {
    def SF_CONSUMER_KEY=env.SF_CONSUMER_KEY
    def SF_USERNAME=env.SF_USERNAME
    def SERVER_KEY_CREDENTALS_ID=env.SERVER_KEY_CREDENTALS_ID
    def SF_INSTANCE_URL = env.SF_INSTANCE_URL ?: "https://login.salesforce.com"

    def toolbelt = tool 'toolbelt'

    withEnv(["HOME=${env.WORKSPACE}"]) {
        
        withCredentials([file(credentialsId: SERVER_KEY_CREDENTALS_ID, variable: 'server_key_file')]) {

            // -------------------------------------------------------------------------
            // Authorize the Dev Hub org with JWT key and give it an alias.
            // -------------------------------------------------------------------------

            stage('Authorize DevHub') {
                //rc = command "${toolbelt}/sfdx auth:jwt:grant --instanceurl ${SF_INSTANCE_URL} --clientid ${SF_CONSUMER_KEY} --username ${SF_USERNAME} --jwtkeyfile ${server_key_file} --setdefaultdevhubusername --setalias HubOrg"
                rc = bat returnStatus: true, script: "${toolbelt}/sfdx auth:jwt:grant --instanceurl ${SF_INSTANCE_URL} --clientid ${SF_CONSUMER_KEY} --username ${SF_USERNAME} --jwtkeyfile <your_server_key_path>/server.key --setdefaultdevhubusername --setalias HubOrg"
                if (rc != 0) {
                    println rc
                   // error 'Salesforce dev hub org authorization failed.'
                }
            }
            
            stage('Update via sfdx') {
               dir('<your_local_git_repo_with_sf_project>') {
                    rc = bat returnStatus: true, script: "${toolbelt}/sfdx force:source:retrieve -x <your_local_git_repo_with_sf_project>/package.xml -u ${SF_USERNAME}"
                    if (rc != 0) {
                        println rc
                    }
               }
            }
            
            stage('Commit') {
                dir('<your_local_git_repo_with_sf_project>') {
                    bat returnStatus: false, script: "git config user.email \"jenkins@<your company email>\""
                    bat returnStatus: false, script: "git config user.name \"jenkins\""
                    rc = bat returnStatus: true, script: "git add -A && git commit -m \"Jenkins autocommit\""
                    if (rc != 0) {
                        println rc
                    }
                }
            }
            
               stage('Push') {
                dir('<your_local_git_repo_with_sf_project>') {
                    rc = bat returnStatus: true, script: "git push https://${GIT_TOKEN}@<your_cloud_git_repo>.git --all"
                    if (rc != 0) {
                        println rc
                    }
                }
            }
        }
    }
}
```

## Useful links

- Continuous Integration Using Jenkins with SalesforceDX | JWT-Based Flow - https://amitsalesforce.blogspot.com/2019/01/continuous-integration-using-jenkins-with-salesforceDx.html
- Continuous Integration using SalesforceDX and Jenkins – Apex Hours https://www.youtube.com/watch?v=IvTlx6lBkQY
