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
          sshagent(['GITHUB_ADM_KEY']) {
            sh '''
              git clone git@github.com:team_org/jenkins-pipeline.git
            '''
          }
          withCredentials([string(credentialsId: 'CHEF_DECRYPTION_KEY', variable: 'decrypt_key')]) {
            sh '''
              openssl aes-256-cbc -d -a -in jenkins-pipeline/knife.rb.enc -out knife.rb -k $decrypt_key
              openssl aes-256-cbc -d -a -in jenkins-pipeline/chefprod.pem.enc -out chefprod.pem -k $decrypt_key
            '''
          }
        }
      }
    }

    stage('Get Statistics') {
      steps {
        script {
          LAST_MONTH = sh(script: "date '+%B' --date='-1 month'", returnStdout: true)
          CBOOK_NUM = sh(script: 'knife cookbook list -c knife.rb|wc -l', returnStdout: true)
          NODE_NUM = sh(script: 'knife node list -c knife.rb|wc -l', returnStdout: true)
          SPRMRT_NUM = sh(script: 'knife cookbook site list -m https://supermarket.chef.io -w -c knife.rb|wc -l', returnStdout: true)
        }
      }
    }

    stage('Email Results') {
      steps {
        script {
          String body = """
          <h1 style="text-align: center;">Chef Statistics</h1>
          <hr />
          <table style="width:100%">
            <tr>
              <td>Number of cookbooks on Chef server</td>
              <td>${CBOOK_NUM}</td>
            </tr>
            <tr>
              <td>Number of nodes registered to Chef server</td>
              <td>${NODE_NUM}</td>
            </tr>
            <tr>
              <td>Number of cookbooks on Chef Supermarket</td>
              <td>${SPRMRT_NUM}</td>
            </tr>
          </table>
          """
          echo "----- Mailing Results -----"
          mail (to: "example@email.com",
          subject: "Chef Statistics for ${LAST_MONTH}",
          body: "${body}",
          mimeType: 'text-html')
        }
      }
    }
  }

  post {
    always {
      echo "----- Clean up Workspace -----"
      cleanWs()
    }

    failure {
      echo "----- FAILURE! -----"
      mail (to: "admin@email.com",
      subject: "Chef Number Pipeline Failure: '${env.JOB_NAME}' (#${env.BUILD_ID})",
      body: "View the project build log <a href=\"${env.BUILD_URL}console\">here</a>",
      mimeType: 'text/html')
    }
  }
}