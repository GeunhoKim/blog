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

        stage("Clone github.io") {
            when {
                not {
                    changelog '.*^\\[SKIP DEPLOY\\] .+$'
                }
            }
            steps {
                echo "##################################"
                echo "# CLONE GITHUB.IO ################"
                echo "##################################"
                sh "git clone https://github.com/geunho/geunho.github.io.git"
            }
        }

        stage("Copy site published") {
            when {
                not {
                    changelog '.*^\\[SKIP DEPLOY\\] .+$'
                }
            }
            steps {
                echo "##################################"
                echo "# COPY SITE PUBLISHED ############"
                echo "##################################"
                sh "cp -r site/public/* geunho.github.io/"
            }
        }

        stage("Publish to github.io") {
            when {
                not {
                    changelog '.*^\\[SKIP DEPLOY\\] .+$'
                }
            }
            steps {
                echo "##################################"
                echo "# PUBLISH TO GITHUB.IO ###########"
                echo "##################################"
                sh "cd geunho.github.io && git add ."
                sh "COMMIT_URL=\$(sed 's/.\\{4\\}\$//' <<< \"${GIT_URL}/commit/${GIT_COMMIT}\") && git commit -m \"[${BUILD_NUMBER}] ${BUILD_URL}\n(${COMMIT_URL})\""
                sh "git push"
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