node('maven') {
    // define commands
    def mvnCmd = "mvn"
    def app='shiro-demo'

    configFileProvider(
    [configFile(fileId: 'maven-settings-xml', variable: 'MAVEN_SETTINGS')]) {
        mvnCmd = "mvn -s $MAVEN_SETTINGS"
    }

    stage ('Build') {
        git branch: 'master', url: 'http://gogs:3000/gogs/${app}.git'
        sh "${mvnCmd} clean install -DskipTests=true"
    }

    stage ('Test and Analysis') {
        parallel (
            'Test': {
                sh "${mvnCmd} test"
                //step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
            },
            'Static Analysis': {
                sh "${mvnCmd} jacoco:report sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
            }
        )
    }

    stage ('Push to Nexus') {
    sh "${mvnCmd} deploy -DskipTests=true"
    }

    stage ('Deploy DEV') {
        sh "rm -rf oc-build && mkdir -p oc-build/deployments"
        sh "cp target/${app}.war oc-build/deployments/ROOT.war"
        sh "oc project dev"
        // clean up. keep the image stream
        sh "oc delete bc,dc,svc,route -l app=${app} -n dev"
        // create build. override the exit code since it complains about exising imagestream
        sh "oc new-build --name=${app} --image-stream=jboss-eap70-openshift --binary=true --labels=app=${app} -n dev || true"
        // build image
        sh "oc start-build ${app} --from-dir=oc-build --wait=true -n dev"
        // deploy image
        sh "oc new-app ${app}:latest -n dev"
        sh "oc expose svc/${app} -n dev"
    }

    stage ('Deploy STAGE') {
        timeout(time:60, unit:'MINUTES') {
        input message: "Promote to STAGE?", ok: "Promote"
        }

        def v = version()
        // tag for stage
        sh "oc tag dev/${app}:latest stage/${app}:${v}"
        sh "oc project stage"
        // clean up. keep the imagestream
        sh "oc delete bc,dc,svc,route -l app=${app} -n stage"
        // deploy stage image
        sh "oc new-app ${app}:${v} -n stage"
        sh "oc expose svc/${app} -n stage"
    }
}

def version() {
  def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}