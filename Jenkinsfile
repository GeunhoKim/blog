pipeline {
    agent any

    stages {
        stage("Hugo Build") {
            steps {
                echo "##################################"
                echo "# HUGO BUILD STARTS ##############"
                echo "##################################"
                sh "rm site/content/til/README.md"
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
                sh "rm -rf geunho.github.io"
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
            environment {
                GITHUB_CRED = credentials('GITHUB_CRED')
            }
            steps {
                echo "##################################"
                echo "# PUBLISH TO GITHUB.IO ###########"
                echo "##################################"               
                dir('geunho.github.io') {
                    sh "git add ."
                    sh "COMMIT_URL=\$(sed 's/.\\{4\\}\$//' <<< \"${GIT_URL}/commit/${GIT_COMMIT}\") && git commit -m \"[PUBLISH#${BUILD_NUMBER}] ${BUILD_URL}\n(\$COMMIT_URL)\""
                    sh "git push https://${GITHUB_CRED}@github.com/geunho/geunho.github.io.git"
                }
            }
        }
    }
    post {
        cleanup {
            deleteDir()
            dir("${workspace}@tmp") {
                deleteDir()
            }
            dir("${workspace}@script") {
                deleteDir()
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