import groovy.json.JsonSlurperClassic 

def version = ''
node {
   stage('checkout') { // for display purposes
      // Get some code from a GitHub repository
      git 'https://github.com/swapnilbarwat/webportal.git'
      version = readFile('version').trim()
      currentBuild.displayName = "${version}-${env.BRANCH_NAME}"
      // Get the Maven tool.
      // ** NOTE: This 'M3' Maven tool must be configured
      // **       in the global configuration.           
   }
   stage('Build') {
       docker.withRegistry('https://docker.io', 'docker-hub-credentials') {
          def app = docker.build("harshals/webportal:${version}")
          sh "docker push docker.io/harshals/webportal:${version}"
       }
    }
}

node {
   sh "curl -X GET http://104.154.31.116:8080/api/v1/deployments > output.json"
   def objectList = jsonParse(readFile('output.json'))
   if(objectList.size() == 0)
   {
    stage('Deploying to cluster') { // for display purposes
       sh "curl -H \"Content-Type: application/x-yaml\" -X PUT http://104.154.31.116:8080/api/v1/deployments/webportal:${version} --data-binary @deployment/blueprint.yml"
    }
   }
   else {
       objectList.each {
          print "Name: $it.name"
          def values = "${it.name}".tokenize(':')
          print values[0]
          print values[1]
          if(values[0] == "webportal")
          {
            print("project name matched..")
            if(values[1] != "${version}")
            {
              stage('50-50% deployment') { // for display purposes
                 input message: 'Deploy to cluster? This will rollout new build to 50% cluster.'
                  sh "curl -H \"Content-Type: application/x-yaml\" -X PUT http://104.154.31.116:8080/api/v1/deployments/webportal:${version} --data-binary @deployment/blueprint.yml"
                  sh "curl -H \"Content-Type: application/x-yaml\" -X POST http://104.154.31.116:8080/api/v1/gateways --data-binary @deployment/split_gateway.yml"
              }
            }
            else
            {
              stage('Deploying to cluster') { // for display purposes
                 sh "curl -H \"Content-Type: application/x-yaml\" -X PUT http://104.154.31.116:8080/api/v1/deployments/webportal:${version} --data-binary @deployment/blueprint.yml"
              }
            }
          }
        }
     }
}

node {
   stage('move full') { // for display purposes
      input message: 'Deploy to full cluster?'
   }
   stage('undeploy previous version') {
      echo "undeploying cluster"
   }
}

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}