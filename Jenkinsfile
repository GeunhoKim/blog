pipeline {
    agent any

    stages {
        stage("Hugo Build") {
            steps {
                echo "##################################"
                echo "# HUGO BUILD STARTS ##############"
                echo "##################################"
                sh "cd site && hugo"
            }
            post {
                always{
                }
                success{
                }
                failure{
                }
            }
        }

        stage("Docker Build") {
            steps {
                echo "##################################"
                echo "# BUILD BLOG DOCKER IMAGE ########"
                echo "##################################"
                sh "docker build -t blog ."
            }
        }

        stage("Docker Run") {
            steps {
                echo "##################################"
                echo "# RUN DOCKER IMAGE ###############"
                echo "##################################"
                sh "docker run --name blog --rm -d -p ${external_port}:80 blog"
            }
        }
    }
    post{
        always{
        }
        success{
        }
        failure{
        }
    }
}