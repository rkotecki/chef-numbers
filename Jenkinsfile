pipeline {
  agent any

  stages {
    stage('Get SSL Certs') {
      steps {
        script {
          sh '''
            knife ssl fetch https://chef.url.com
            knife ssl fetch https://supermarket.chef.io
          '''
        }
      }
    }
    stage('Get Knife Files') {
      steps {
        script {
          sshagent(['GITHUB_SCMADM_SSH_KEY']) {
            sh '''
              git clone git@github.com:team_org/jenkins-pipeline.git
            '''
          }
          withCredentials([string(credentialsId: 'CHEF_DECRYPTION_KEY', variable: 'decrypt_key')]) {
            sh '''
              openssl aes-256-cbc -d -a -in jenkins-pipeline/dfs_knife_qa.rb.enc -out dfs_knife_qa.rb -k $decrypt_key
              openssl aes-256-cbc -d -a -in jenkins-pipeline/vrachefprod.pem.enc -out vrachefprod.pem -k $decrypt_key
              openssl aes-256-cbc -d -a -in jenkins-pipeline/dfs_knife.rb.enc -out dfs_knife.rb -k $decrypt_key
              openssl aes-256-cbc -d -a -in jenkins-pipeline/vrachefqa.pem.enc -out grommel.pem -k $decrypt_key
            '''
          }
        }
      }
    }
    stage('Get Statistics') {
      steps {
        script {
          sh """
            inspec compliance login https://chef-automate.com --insecure --user=admin  --ent=default --token $JENKINS_CHEF_COMPLIANCE_TOKEN
            inspec compliance profiles > profiles.txt
            cat profiles.txt | cut -d ' ' -f 3- | grep -o '^.* ' | sed 's/v[0-9].[0-9].[0-9].*//' | sort -u > profile_ct.txt
          """

          LAST_MONTH = sh(script: "date '+%B' --date='-1 month'", returnStdout: true)
          YEAR = sh(script: "date '+%Y'", returnStdout: true)
          PROD_CBOOK_NUM = sh(script: 'knife cookbook list -c dfs_knife.rb|wc -l', returnStdout: true)
          PROD_NODE_NUM = sh(script: 'knife node list -c dfs_knife.rb|wc -l', returnStdout: true)
          QA_CBOOK_NUM = sh(script: 'knife cookbook list -c dfs_knife_qa.rb|wc -l', returnStdout: true)
          QA_NODE_NUM = sh(script: 'knife node list -c dfs_knife_qa.rb|wc -l', returnStdout: true)
          SPRMRT_NUM = sh(script: 'knife cookbook site list -m https://supermarket.chef.io -w -c dfs_knife.rb| wc -l', returnStdout: true)
          COMP_NUM = sh(script: 'cat profile_ct.txt | wc -l',returnStdout: true)
        }
      }
    }
    stage('Email Results') {
      steps {
        script {
          String body = """
          <html>
            <head>
              <title>Chef Statistics - ${LAST_MONTH}</title>
            </head>
            <body>
              <table style="width:100%">
                <tr>
                  <td>
                    <h1 style="text-align: center;">Chef Statistics - ${LAST_MONTH}</h1>
                    <br>
                  </td>
                </tr>
                <tr>
                  <td>
                    <h2 style="text-align: center;">Nodes</h2>
                    <table style="width:100%">
                      <tr>
                        <td>Number of nodes registered to QA Chef Server</td>
                        <td>${QA_NODE_NUM}</td>
                      </tr>
                      <tr>
                        <td>Number of nodes registered to Prod Chef Server</td>
                        <td>${PROD_NODE_NUM}</td>
                      </tr>
                    </table>
                    <h2 style="text-align: center;">Cookbooks</h2>
                    <table style="width:100%">
                      <tr>
                        <td>Number of cookbooks on QA Chef server</td>
                        <td>${QA_CBOOK_NUM}</td>
                      </tr>
                      <tr>
                        <td>Number of cookbooks on Prod Chef server</td>
                        <td>${PROD_CBOOK_NUM}</td>
                      </tr>
                      <tr>
                        <td>Number of cookbooks on Chef Supermarket</td>
                        <td>${SPRMRT_NUM}</td>
                      </tr>
                    </table>
                    <h2 style="text-align: center;">Compliance Profiles</h2>
                    <table style="width:100%">
                      <tr>
                        <td>Number of Chef Compliance Profiles</td>
                        <td>${COMP_NUM}</td>
                      </tr>
                    </table>
                  </td>
                </tr>
                <tr>
                  <td align="center"><br>${YEAR} - Cloud Integration and Automation</td>
                </tr>
              </table>
            </body>
          </html>
          """
          echo "----- Mailing Results -----"
          mail (to: "teamthatwantsstats@email.com",
          subject: "Chef Statistics for ${LAST_MONTH}",
          body: "${body}",
          mimeType: 'text/html')
        }
      }
    }
  }
  
  post {
    always {
      echo "----- Clean Up After Yoself! -----"
      sh 'inspec compliance logout'
      cleanWs()
    }
    failure {
      echo "----- FAILURE! -----"
      mail (to: "ryankotecki@email.com",
      subject: "Chef Numbers Pipeline Failure: '${env.JOB_NAME}' (#${env.BUILD_ID})",
      body: "View the project build log <a href=\"${env.BUILD_URL}console\">here</a>",
      mimeType: 'text/html')
    }
  }
}