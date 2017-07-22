import groovy.json.JsonSlurperClassic

def version = ''
def vampIP="104.198.21.175"
def deploymentPresentFlag = false
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
   sh "curl -X GET http://${vampIP}:8080/api/v1/deployments > output.json"
   def objectList = jsonParse(readFile('output.json'))
   if(objectList.size() == 0)
   {
    stage('Deploying to cluster') { // for display purposes
       sh "curl -H \"Content-Type: application/x-yaml\" -X PUT http://${vampIP}:8080/api/v1/deployments/webportal:${version} --data-binary @deployment/blueprint.yml"
       deploymentPresentFlag=false
    }
   }
   else {
       objectList.each {
          print "Name: $it.name"
          def values = "${it.name}".tokenize(':')
          print values[0]
          print values[1]
          
          if(values[1] != "${version}")
          {
            stage('50-50% deployment') { // for display purposes
              input message: 'Deploy to cluster? This will rollout new build to 50% cluster.'
              sh "curl -H \"Content-Type: application/x-yaml\" -X PUT http://${vampIP}:8080/api/v1/deployments/webportal:${version} --data-binary @deployment/blueprint.yml"
              sh "curl -H \"Content-Type: application/x-yaml\" -X POST http://${vampIP}:8080/api/v1/gateways --data-binary @deployment/split_gateway.yml"
              deploymentPresentFlag=true
            }
          }
          else
          {
            stage('Deploying to cluster') { // for display purposes
              sh "curl -H \"Content-Type: application/x-yaml\" -X PUT http://${vampIP}:8080/api/v1/deployments/webportal:${version} --data-binary @deployment/blueprint.yml"
              sh "curl -H \"Content-Type: application/x-yaml\" -X POST http://${vampIP}:8080/api/v1/gateways --data-binary @deployment/external.yml"
              deploymentPresentFlag=false
            }
          }
        }
     }
}

node {
   if(deploymentPresentFlag == true) {
    stage('move full & undeploy previous version') { 
      input message: 'Deploy to full cluster?'
         deploymentContent=readFile("deployment/blueprint.yml")
         deploymentContent.replace("0.0.2","0.0.1")
         def fp = new File("/tmp/oldDeployment.yml")
         fp.write(deploymentContent)
         echo "${deploymentContent}"
         sh "curl -H \"Content-Type: application/x-yaml\" -X DELETE http://${vampIP}:8080/api/v1/deployments/webportal:0.0.1 --data-binary @/tmp/oldDeployment.yml"
         sh "curl -H \"Content-Type: application/x-yaml\" -X DELETE http://${vampIP}:8080/api/v1/gateways/webportal"
       }
   }
}

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}

def combinePath(version1, version2) 
{
    String[] levels1 = version1.split("\\.");
    String[] levels2 = version2.split("\\.");

    int length = Math.max(levels1.length, levels2.length);
    for (int i = 0; i < length; i++){
        Integer v1 = i < levels1.length ? Integer.parseInt(levels1[i]) : 0;
        Integer v2 = i < levels2.length ? Integer.parseInt(levels2[i]) : 0;
        int compare = v1.compareTo(v2);
        if (compare != 0){
            return compare;
        }
    }

    return 0;
}