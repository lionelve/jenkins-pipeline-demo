jettyUrl = 'http://localhost:8081/'

def servers

input message: 'Where do you want to build this?', parameters: [[$class: 'LabelParameterDefinition', allNodesMatchingLabel: false, defaultValue: '', description: 'Options are: ec2, docker', name: 'label']]

//echo "The label is $nodeLabel"

stage 'Dev'
node {
   checkout scm
   servers = load 'servers.groovy'
   def mvnHome = tool 'M3'
   sh "${mvnHome}/bin/mvn clean package"
   dir('target') {stash name: 'war', includes: 'x.war'}
}

stage 'QA'
parallel(longerTests: {
    runTests(servers, 30)
}, quickerTests: {
    runTests(servers, 20)
})

stage name: 'Staging', concurrency: 1
node {
    servers.deploy 'staging'
}

input message: "Does ${jettyUrl}staging/ look good?"
try {
    checkpoint('Before production')
} catch (NoSuchMethodError _) {
    echo 'Checkpoint feature available in CloudBees Jenkins Enterprise.'
}

stage name: 'Production', concurrency: 1
node {
    sh "wget -O - -S ${jettyUrl}staging/"
    echo 'Production server looks to be alive'
    servers.deploy 'production'
    echo "Deployed to ${jettyUrl}production/"
}

def mvn(args) {
    sh "${tool 'M3'}/bin/mvn ${args}"
}

def runTests(servers, duration) {
    node {
        checkout scm
        servers.runWithServer {id ->
            mvn "-o -f sometests test -Durl=${jettyUrl}${id}/ -Dduration=${duration}"
        }
    }
}
