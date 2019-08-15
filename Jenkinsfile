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
            when {
                not {
                    changelog '.*^\\[SKIP DEPLOY\\] .+$'
                }
            }
            steps {
                echo "##################################"
                echo "# BUILD BLOG DOCKER IMAGE ########"
                echo "##################################"
                sh "cd site && /usr/local/bin/docker build -t blog ."
            }
        }

        stage("Docker Run") {
            when {
                not {
                    changelog '.*^\\[SKIP DEPLOY\\] .+$'
                }
            }
            steps {
                echo "##################################"
                echo "# RUN DOCKER IMAGE ###############"
                echo "##################################"
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "/usr/local/bin/docker rm -f blog"
                }
                sh "/usr/local/bin/docker run --name blog --restart always -d -p ${external_port}:80 -p ${external_ssl_port}:443 -v ${certificate_path}:/letsencrypt blog"
            }
        }

        stage("Docker Clean") {
            steps {
                echo "##################################"
                echo "# WIPEOUT DANGLING DOCKER IMAGES #"
                echo "##################################"
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "/usr/local/bin/docker rmi \$(/usr/local/bin/docker images --quiet --filter \"dangling=true\")"
                }
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