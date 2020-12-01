def git_repo = 'https://github.com/penta-data/appsodytest.git'
def public_route_prefix = 'appsody-2'

def git_branch = 'master'
def env = 'demo'
def nexus_base_url = 'http://10.30.30.213:8081'
def nexus_deps_repo = "$nexus_base_url/repository/ist_maven_proxy/"
def nexus_deploy_repo = "$nexus_base_url/repository/maven-releases/"

def ocp_project = 'demo'
def oc_command = 'create'

def cpu_limit = '30m'
def memory_limit = '300Mi'
def max_replica_count = 2


def appName
def appFullVersion
def gitCommitId

node (){
    stage('Checkout') {
        git url: "${git_repo}", branch: "${git_branch}", credentialsId: 'elyusron_github_credential'
    }
    stage('Prepare'){
        withCredentials([[$class: 'UsernamePasswordMultiBinding', 
            credentialsId: 'nexus',
            usernameVariable: 'nexus_username', passwordVariable: 'nexus_password']]) {
                sh """
                    echo 'Downloading ci-cd templates...'
                    curl --fail -u ${nexus_username}:${nexus_password} -o cicd-template.tar.gz ${nexus_base_url}/repository/general-ist/cicd-template-${env}.tar.gz
                    rm -rf cicd-template
                    mkdir cicd-template && tar -xzvf ./cicd-template.tar.gz -C "\$(pwd)/cicd-template"
                    chmod -R 777 "\$(pwd)/cicd-template"
                    """
                prepareSettingsXml(nexus_deps_repo, nexus_username, nexus_password)
                addDistributionToPom(nexus_deploy_repo)
        }
        
        appName = getFromPom('name')
        if(appName == null || appName.trim() == ""){
          appName = getFromPom('artifactId')
        }
        sh "mvn -s ./cicd-template/maven/settings.xml build-helper:parse-version versions:set -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.${BUILD_NUMBER} versions:commit"
        appFullVersion = getFromPom('version')
        gitCommitId = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
        echo "appName: '${appName}', appFullVersion:'${appFullVersion}', gitCommitId:'${gitCommitId}'"
    }
    stage('Build') {
        sh 'mvn clean package -D skipTests -s ./cicd-template/maven/settings.xml'
    }
    //stage ('Test'){
    //    sh 'mvn test -s ./cicd-template/maven/settings.xml'
    //}
    //stage('IntegrationTests') {
    //   sh 'mvn failsafe:integration-test -s ./cicd-template/maven/settings.xml'
    //}
    stage('Update Sonar Code Quality'){
        sh 'mvn sonar:sonar -Dsonar.host.url=http://10.30.30.215:9000/ -s ./cicd-template/maven/settings.xml' 
    }

    stage ('Archive'){
        sh 'mvn deploy  -DskipTests -s ./cicd-template/maven/settings.xml'
    }
    stage ('OpenShift Build'){
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
            credentialsId: 'oc-demo',
            usernameVariable: 'oc_username', passwordVariable: 'oc_password']]) {
                  sh 'oc login -u=${oc_username} -p=${oc_password} https://ocp.mylabzolution.com:8443 --insecure-skip-tls-verify=true'
                    // sh 'oc login --token=KehJFsZ5K7IUhfImlGlAXqukBPgGeIghbBglLP1uISg --server=https://c101-e.jp-tok.containers.cloud.ibm.com:30157'
               }

        appMajorVersion = appFullVersion.substring(0, appFullVersion.indexOf('.'))
        jarFile = sh(returnStdout: true, script: 'find ./target -maxdepth 1 -regextype posix-extended -regex ".+\\.(jar|war)\$" | head -n 1').trim()
        if(jarFile == null || jarFile == ""){
          error 'Can not find the generated jar/war file from "./target" directory'
        }
        jarFile = jarFile.substring('./target/'.length());

        sh """
            set -x
            set -e

            mkdir -p ./target/publish/.s2i
            cp ./target/$jarFile ./target/publish/
           echo 'JAVA_APP_JAR=/deployments/${jarFile}' > ./target/publish/.s2i/environment
        """

        // TODO - make it different stages
        sh """
                set -x
                set -e

                oc project ${ocp_project}
                oc process -f ./cicd-template/openshift/build-config-template.yaml -n ${ocp_project} \
                -p S2I_BUILD_IMAGE='openjdk-11-rhel7' -p S2I_BUILD_IMAGE_PULL_SECRET='default-dockercfg-7kc7p' \
                -p APP_NAME='${appName}' -p APP_FULL_VERSION='${appFullVersion}' -p APP_MAJOR_VERSION='${appMajorVersion}' \
                -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} \
                | oc ${oc_command} -n ${ocp_project} -f -
                
            
                oc start-build ${appName}-v${appMajorVersion} -n ${ocp_project} --from-dir='./target/publish' --follow
           """
    }

   // stage ('OpenShift Applciation ConfigMap'){
   //     sh """
   //             set -x
   //             set -e
   //             if [ -f './src/config/dev/application.properties' ]; then
   //                 export APP_CONFIG_DATA=\$(cat src/config/dev/application.properties)
   //             else 
   //                 export APP_CONFIG_DATA='key=value'
   //             fi
   //
   //             oc project ${ocp_project}
   //             oc process -f ./cicd-template/openshift/configmap-template.yaml -n ${ocp_project} \
   //               -p APP_NAME='${appName}' -p APP_FULL_VERSION='${appFullVersion}' -p APP_MAJOR_VERSION='${appMajorVersion}' \
   //               -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} -p CONFIG_DATA="\$APP_CONFIG_DATA" \
   //               | oc apply -n ${ocp_project} -f -
   //        """
   // }

    stage ('OpenShift Deployment'){
        sh """
            set -x
            set -e

            oc project ${ocp_project}
            oc process -f ./cicd-template/openshift/deployment-config-template.yaml -n ${ocp_project} \
                -p APP_NAME=${appName} -p APP_FULL_VERSION=${appFullVersion} -p APP_MAJOR_VERSION=${appMajorVersion}  \
                -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} -p CPU_LIMIT=${cpu_limit} -p MEM_LIMIT=${memory_limit} \
                | oc ${oc_command} -n ${ocp_project}  -f -
            sleep 5
            """

        if (public_route_prefix != null && public_route_prefix != ''){
            sh """
                set -x
                set -e

                oc project ${ocp_project}
                oc process -f ./cicd-template/openshift/route-template.yaml -n ${ocp_project} \
                    -p APP_NAME=${appName} -p APP_FULL_VERSION=${appFullVersion} -p APP_MAJOR_VERSION=${appMajorVersion}  \
                    -p GIT_COMMIT_ID=${gitCommitId} -p PUBLIC_ROUTE_PREFIX=${public_route_prefix} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} \
                    | oc ${oc_command} -n ${ocp_project}  -f -
                sleep 5

                oc rollout status dc/${appName}-v${appMajorVersion}
                """
        }
    }
}


def getFromPom(key) {
    sh(returnStdout: true, script: "mvn -s ./cicd-template/maven/settings.xml -q -Dexec.executable=echo -Dexec.args='\${project.${key}}' --non-recursive exec:exec").trim()
}

def addDistributionToPom(nexus_deploy_repo) {
    pom = 'pom.xml'
    distMngtSection = readFile('./cicd-template/maven/pom-distribution-management.xml') 
    distMngtSection = distMngtSection.replaceAll('\\$nexus_deploy_repo', nexus_deploy_repo)

    content = readFile(pom)
    newContent = content.substring(0, content.lastIndexOf('</project>')) + distMngtSection + '</project>'
    writeFile file: pom, text: newContent
}

def prepareSettingsXml(nexus_deps_repo, nexus_username, nexus_password) {
    settingsXML = readFile('./cicd-template/maven/settings.xml') 
    settingsXML = settingsXML.replaceAll('\\$nexus_deps_repo', nexus_deps_repo)
    settingsXML = settingsXML.replaceAll('\\$nexus_username', nexus_username)
    settingsXML = settingsXML.replaceAll('\\$nexus_password', nexus_password)

    writeFile file: './cicd-template/maven/settings.xml', text: settingsXML
}