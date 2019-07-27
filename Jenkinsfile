pipeline {
    agent any

    stages {
        stage("Hugo Build") {
            steps {
                echo "##################################"
                echo "# HUGO BUILD STARTS ##############"
                echo "##################################"
                sh "cd site && /usr/local/bin/hugo"
            }
            /*post {
                always{
                }
                success{
                }
                failure{
                }
            }*/
        }

        stage("Docker Build") {
            steps {
                echo "##################################"
                echo "# BUILD BLOG DOCKER IMAGE ########"
                echo "##################################"
                sh "cd site && /usr/local/bin/docker build -t blog ."
            }
        }

        stage("Docker Run") {
            steps {
                echo "##################################"
                echo "# RUN DOCKER IMAGE ###############"
                echo "##################################"
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "/usr/local/bin/docker rm -f blog"
                }
                sh "/usr/local/bin/docker run --name blog -d -p ${external_port}:8080 blog"
            }
        }
    }
    /*post{
        always{
        }
        success{
        }
        failure{
        }
    }*/
}