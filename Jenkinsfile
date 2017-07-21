this.getClass().classLoader.rootLoader.addURL(new File("/var/lib/jenkins/jar/snakeyaml-1.4.jar").toURL())

import groovy.json.JsonSlurperClassic

import org.yaml.snakeyaml.DumperOptions
import org.yaml.snakeyaml.Yaml

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
     sh "echo ${version} > /tmp/preversion"
}

node {
   stage('move full') { // for display purposes
      input message: 'Deploy to full cluster?'
   }
   stage('undeploy previous version') {
        def prevVersion=readFile("/tmp/preversion")
        Yaml yaml = new Yaml()
        def data=readFile("deployment/blueprint.yml")
        def Map map = (Map) yaml.load(data)
        map.name="webportal:${prevVersion}"
        map.clusters.webportal.services.breed.deployable="harshals/webportal:${prevVersion}"
        DumperOptions options = new DumperOptions()
        options.setPrettyFlow(true)
        options.setDefaultFlowStyle(DumperOptions.FlowStyle.BLOCK)
        yaml = new Yaml(options)
        yaml.dump(map, new FileWriter(prevblueprint.yml))
        sh "curl -H \"Content-Type: application/x-yaml\" -X DELETE http://104.154.31.116:8080//api/v1/deployments/webportal:${prevVersion} --data-binary @prevblueprint.yml"
   }
}

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}