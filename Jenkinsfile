pipeline {
    agent {
        label "local"
    }
    environment {
        VERSION = readFile "${env.WORKSPACE}/version.txt"
        BUILD_BRANCH = readFile "${env.WORKSPACE}/build_branch.txt"
        NEXUS_USER=${env.NEXUS_USER}
        NEXUS_PASS=${env.NEXUS_PASS}
    }
    stages {
        stage("build and publish") {
            agent {
                label "local"
            }
            steps {
              sh """
                set -ex
                docker login -u ${env.DOCKER_USER} -p ${env.DOCKER_PASS}
                git checkout ${BUILD_BRANCH}
                ./build_distribution.sh --spark-profile spark3 --hadoop-profile hadoop3.2 --without-dashboard
                cd deploy/kubernetes/docker ||  exit
                ./build.sh --hadoop-version 3.2.0.13-EE-SNAPSHOT --registry docker.hops.works --nexus-user $NEXUS_USER --nexus-password $NEXUS_PASS
                cd ../../.. #go back to the root of the project
                RSS_VERSION=$(./mvnw help:evaluate -Dexpression=project.version 2>/dev/null | grep -v "INFO" | grep -v "WARNING" | tail -n 1)
                mkdir -p /opt/repository/master/rss/${VERSION}/
                cp  client/target/rss-client-{RSS_VERSION}.jar /opt/repository/master/rss/${VERSION}/
              """
            }
        }
    }

    post {
        success {
            build job:'Remote shuffle services', parameters: [
            string(name: 'image', value: "rss"),
            string(name: 'branch', value: "master")
            ]
        }
    }
}