@Library('jenkins-libraries') _

def ENV_MAP = [
    "dev":  [ "AccountId":"012345678900", "AssumeRoleName":"dev-jenkins-iam-role", "SessionName":"dev", "S3BucketName":"dev-cloudfront-origin-s3" ],
    "uat":  [ "AccountId":"012345678999", "AssumeRoleName":"uat-jenkins-iam-role", "SessionName":"uat", "S3BucketName":"uat-cloudfront-origin-s3" ],
    "nft":  [ "AccountId":"012345678922", "AssumeRoleName":"nft-jenkins-iam-role", "SessionName":"nft", "S3BucketName":"nft-cloudfront-origin-s3" ]
]
def CHILD_ELEMENTS = []
def ALL_ELEMENTS = ['ALMOND', 'HAZEL', 'COCONUT']
def PACKAGE_NAME = 'undefined'

pipeline {
    agent any

    options {
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    parameters {
        choice(name: "ENVIRONMENT_NAME",
            choices: ["dev", "uat", "nft"],
            description: "Environment name required. Default value is: dev")
        choice(name: 'SINGLE_ELEMENT',
            choices: ['No', 'Yes'],
            description: "Deploy a single element?)
        choice(name: 'ELEMENT_NAME',
            choices: ['ALMOND', 'HAZEL', 'COCONUT'],
            description: 'Element name required')
        string(defaultValue: "develop",
            name: 'APPLICATION_REPO_BRANCH',
            description: 'Branch name required. Default value is: develop')
    }

    environment {
        GIT_GROUP = 'VW'
        APPLICATION_REPO = 'MasterApp'
        APPLICATION_REPO_URI = "${GIT_GROUP}" + '/' + "${APPLICATION_REPO}"
        APPLICATION_PATH = "${WORKSPACE}/${APPLICATION_REPO}"
        SOURCEOFARTIFACTPATH = "${APPLICATION_PATH}/dist/"
        INCLUDE_ELEMENT_JS = "${APPLICATION_PATH}/src/assets/libs/"
        NEXUS_HOST = 'nexus.wideriver.co.uk'
        NEXUS_URI = "https://${NEXUS_HOST}"
        NEXUS_REPO = "${NEXUS_HOST}/repository"
        NEXUS_REPO_URI = "https://${NEXUS_REPO}"
        NEXUS_NPM_HOSTED_REPO_ID = 'npm-hosted'
        NEXUS_NPM_SNAPSHOT_REPO_ID = 'npm-snapshots'
        NEXUS_NPM_RELEASE_REPO_ID = 'npm-releases'
        APP_ENV = "${params.ENVIRONMENT_NAME}"
    }

    stages {
        stage ('Pipeline variables') {
            stages {
                stage ('Pipeline variables - downstream elements') {
                    steps {
                        script {
                            if ( params.SINGLE_ELEMENT == "Yes" ) {
                                CHILD_ELEMENTS.add(params.ELEMENT_NAME)
                            } else {
                                for (int i = 0; i < ALL_ELEMENTS.size(); i++) {
                                    CHILD_ELEMENTS.add(ALL_ELEMENTS[i])
                                }
                            }
                        }
                    } // end 'steps'
                } // end 'stage ('Pipeline variables - downstream elements')''
                stage ('Pipeline variables - develop') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME == 'dev' }
                            expression { params.APPLICATION_REPO_BRANCH  == 'develop'}
                        }
                    }
                    steps {
                        script {
                            env.APPLICATION_REPO_BRANCH = "develop"
                            echo "Using environment: ${ENVIRONMENT_NAME}"
                            echo "Using branch: ${APPLICATION_REPO_BRANCH}"
                        }
                    } // end 'steps'
                } // end 'stage ('Pipeline variables - develop')'
                stage ('Pipeline variables - release') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME ==~ /(?i)(sit|uat)/ }
                            expression { params.APPLICATION_REPO_BRANCH =~ /^release\/*/ }
                        }
                    }
                    steps {
                        script {
                            env.APPLICATION_REPO_BRANCH = "release"
                            echo "Using environment: ${ENVIRONMENT_NAME}"
                            echo "Using branch: ${APPLICATION_REPO_BRANCH}"
                        }
                    } // end 'steps'
                } // end 'stage ('Pipeline variables - release')'
            } // end 'stages'
        } // end 'stage('Pipeline variables')'

        stage ('Checkout') {
            steps {
                script {
                    ansiColor('xterm') {
                        checkoutGitRepository(repository: APPLICATION_REPO_URI, branch: params.APPLICATION_REPO_BRANCH)
                    }
                }
            }
        } // end 'stage('Checkout')'

        stage ('Invoke pipeline') {
            steps {
                script {
                    for (int i = 0; i < CHILD_ELEMENTS.size(); i++) {
                        echo "Build: ${CHILD_ELEMENTS[i]}"
                        echo "Branch: ${params.APPLICATION_REPO_BRANCH}"
                        build job: "dev-applications/${CHILD_ELEMENTS[i]}/code-build",
                            parameters: [
                                [$class: 'StringParameterValue', name: 'ENVIRONMENT_NAME', value: params.ENVIRONMENT_NAME],
                                [$class: 'StringParameterValue', name: 'APPLICATION_REPO_BRANCH', value: params.APPLICATION_REPO_BRANCH]
                            ],
                            wait: true
                    }
                } // end 'script'
            } // end 'steps'
        } // end 'stage ('Invoke pipeline')'

        stage ('Npm installation config') {
            steps {
                script {
                    ansiColor('xterm') {
                        sh '''
                        #!/bin/bash
                        echo "" > ~/.npmrc
                        npm config set registry ${NEXUS_REPO_URI}/${NEXUS_NPM_HOSTED_REPO_ID}/
                        npm config set optional true
                        npm config set ignore-scripts true
                        npm config set ignore-prepublish true
                        npm config set strict-ssl false
                        npm config set always-auth false
                        npm config set audit false
                        npm config set loglevel info
                        '''
                    }
                }
            }
        } // end 'stage ('Npm installation config')'

        stage ('Npm install') {
            steps {
                script {
                    ansiColor('xterm') {
                        sh '''
                        #!/bin/bash
                        cd ${APPLICATION_REPO}
                        npm ci
                        '''
                    }
                }
            }
        } // end 'stage ('Npm install')'

        stage ('Npm fetch artifact') {
            stages {
                stage ('Npm fetch artifact - develop') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME == 'dev' }
                            expression { params.APPLICATION_REPO_BRANCH == 'develop'}
                        }
                    }
                    steps {
                        script {
                            for (int i = 0; i < ALL_ELEMENTS.size(); i++) {
                                sh """
                                #!/bin/bash
                                cd ${APPLICATION_REPO}
                                npm cache clean --force
                                npm cache verify
                                npm config set registry ${NEXUS_REPO_URI}/${NEXUS_NPM_SNAPSHOT_REPO_ID}/
                                npm pack --cache-min 0 --loglevel info ${ALL_ELEMENTS[i].toLowerCase()}
                                sha1sum ${ALL_ELEMENTS[i].toLowerCase()}-*.tgz
                                tar --extract --verbose --file ${ALL_ELEMENTS[i].toLowerCase()}-*.tgz
                                sha1sum package/elements/${ALL_ELEMENTS[i]}.js
                                cp package/elements/${ALL_ELEMENTS[i]}.js ${INCLUDE_ELEMENT_JS}                          
                                """
                            }
                        } // end 'script'
                    } // end 'steps'
                } // end 'stage ('Npm fetch artifact - develop')'

                stage ('Npm fetch artifact - release') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME ==~ /(?i)(uat|nft)/ }
                            expression { params.APPLICATION_REPO_BRANCH =~ /^release\/*/ }
                        }
                    }
                    steps {
                        script {
                            for (int i = 0; i < ALL_ELEMENTS.size(); i++) {
                                sh """
                                #!/bin/bash
                                cd ${APPLICATION_REPO}
                                npm cache clean --force
                                npm cache verify
                                npm config set registry ${NEXUS_REPO_URI}/${NEXUS_NPM_RELEASE_REPO_ID}/
                                npm pack --cache-min 0 --loglevel info ${ALL_ELEMENTS[i].toLowerCase()}
                                sha1sum ${ALL_ELEMENTS[i].toLowerCase()}-*.tgz
                                tar --extract --verbose --file ${ALL_ELEMENTS[i].toLowerCase()}-*.tgz
                                sha1sum package/elements/${ALL_ELEMENTS[i]}.js
                                cp package/elements/${ALL_ELEMENTS[i]}.js ${INCLUDE_ELEMENT_JS}
                                """
                            }
                        } // end 'script'

                    } // end 'steps'
                } // end 'stage ('Npm fetch artifact - release')'
            } // end 'stages'
        } // end 'stage ('Npm fetch artifact')'

        stage ('Npm build artifact') {
            stages {
                stage('Npm build artifact - develop') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME == 'dev' }
                            expression { params.APPLICATION_REPO_BRANCH == 'develop'}
                        }
                    }
                    steps {
                        script {
                            ansiColor('xterm') {
                                sh '''
                                #!/bin/bash
                                cd ${APPLICATION_REPO}
                                npm config set ignore-scripts false
                                npm run build:${APP_ENV}
                                '''
                            }
                        }

                    }
                } // end 'stage('Npm build artifact - develop')'

                stage('Npm build artifact - release') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME ==~ /(?i)(uat|nft)/ }
                            expression { params.APPLICATION_REPO_BRANCH =~ /^release\/*/ }
                        }
                    }
                    steps {
                        script {
                            ansiColor('xterm') {
                                sh '''
                                #!/bin/bash
                                cd ${APPLICATION_REPO}
                                npm config set ignore-scripts false
                                npm run build:${APP_ENV}
                                '''
                            }
                        }
                    }
                } // end 'stage('Npm build artifact - release')'
            } // end 'stages'
        } // end 'stage ('Build artifact')'


        stage ('Unit Test') {
            steps {
                    script {
                        ansiColor('xterm') {
                            sh '''
                            #!/bin/bash
                            cd ${APPLICATION_REPO}
                            npm config set ignore-scripts false
                            npm run test
                            '''
                        }
                    }
            }
        } // end 'stage ('Unit Test')'

              stage ('Sonar Scan') {
            environment {
                SONAR_TOKEN = credentials('sonar-access-token')
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {
                        ansiColor('xterm') {
                            sh '''
                            #!/bin/bash
                            cd ${APPLICATION_REPO}
                            node_modules/sonar-scanner/bin/sonar-scanner -Dsonar.login=${SONAR_TOKEN}
                            '''
                        }
                    }
                }
            }
        } // end 'stage ('Sonar Scan')'

        stage ('Sonar Quality Gate') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {
                        ansiColor('xterm') {
                            script {
                                sh '''
                                #!/bin/bash
                                set +x
                                cd ${APPLICATION_PATH}
                                SONAR_URI="https://${NEXUS_HOST}"
                                PACKAGE_NAME=$(jq --raw-output '.name' < ./package.json) 
                                if [ ! -z "${PACKAGE_NAME}" ]
                                then
                                    PROJECT_URL="${SONAR_URI}/api/qualitygates/project_status?projectKey=${PACKAGE_NAME}"
                                    HTTP_STATUS_CLI=$(curl -s -w '%{http_code}\n' --silent --output /dev/null ${PROJECT_URL})
                                    if [ "${HTTP_STATUS_CLI}" == "200" ]
                                    then
                                        MAIN_PROJECT_STATUS=$(curl -sL ${SONAR_URI}/api/qualitygates/project_status?projectKey=${PACKAGE_NAME} \
                                            | jq --raw-output '.projectStatus.status')
                                        GATE_PROJECT_STATUS=$(curl -sL ${SONAR_URI}/api/qualitygates/project_status?projectKey=${PACKAGE_NAME} \
                                            | jq --raw-output '.projectStatus.conditions[] | {status: .status, metricKey: .metricKey}')
                                        echo "Analysis details here: ${SONAR_URI}/dashboard?id=${PACKAGE_NAME}"
                                        echo "Quality gate details:"
                                        echo "${GATE_PROJECT_STATUS}"
                                        if [ "${MAIN_PROJECT_STATUS}" == "ERROR" ]
                                        then
                                            exit 1
                                        fi
                                    fi
                                fi
                                set -x
                                '''
                            }
                        }
                    }
                }
            }
        } // end 'stage ('Sonar Quality Gate')'

        stage ('Npm publish config') {
            environment {
                NEXUS_TOKEN = credentials('nexus-access-token')
            }
            stages {
                stage ('Npm publish config - develop') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME == 'dev' }
                            expression { params.APPLICATION_REPO_BRANCH == 'develop'}
                        }
                    }
                    steps {
                        script {
                            ansiColor('xterm') {
                                sh '''
                                #!/bin/bash
                                npm config set ignore-scripts true
                                npm config set strict-ssl false
                                npm config set registry https://${NEXUS_REPO}/${NEXUS_NPM_SNAPSHOT_REPO_ID}/
                                npm set //${NEXUS_REPO}/${NEXUS_NPM_SNAPSHOT_REPO_ID}/:_authToken ${NEXUS_TOKEN}
                                '''
                            }
                        }
                    }
                } // end 'stage ('Npm publish config - develop')'

                stage ('Npm publish config - release') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME ==~ /(?i)(uat|nft)/ }
                            expression { params.APPLICATION_REPO_BRANCH =~ /^release\/*/ }
                        }
                    }
                    steps {
                        script {
                            ansiColor('xterm') {
                                sh '''
                                #!/bin/bash
                                npm config set ignore-scripts true
                                npm config set strict-ssl false
                                npm config set registry https://${NEXUS_REPO}/${NEXUS_NPM_RELEASE_REPO_ID}/
                                npm set //${NEXUS_REPO}/${NEXUS_NPM_RELEASE_REPO_ID}/:_authToken ${NEXUS_TOKEN}
                                '''
                            }
                        }
                    }
                } // end 'stage ('Npm publish config - release')'

            } // end 'stages'
        } // end 'stage ('Npm publish config')'

        stage('Npm publish artifact') {
            stages {
                stage ('Npm publish artifact - develop') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME == 'dev' }
                            expression { params.APPLICATION_REPO_BRANCH == 'develop'}
                        }
                    }
                    steps {
                        script {
                            ansiColor('xterm') {
                                sh '''
                                #!/bin/bash
                                cd ${APPLICATION_REPO}
                                npm publish
                                '''
                            }
                        }
                    }
                }

                stage ('Npm publish artifact - release') {
                    when {
                        allOf {
                            expression { params.ENVIRONMENT_NAME ==~ /(?i)(uat|nft)/ }
                            expression { params.APPLICATION_REPO_BRANCH =~ /^release\/*/ }
                        }
                    }
                    steps {
                        script {
                            ansiColor('xterm') {
                                sh '''
                                #!/bin/bash
                                cd ${APPLICATION_REPO}
                                sed -i -e "s/${NEXUS_NPM_SNAPSHOT_REPO_ID}/${NEXUS_NPM_RELEASE_REPO_ID}/g" ./package.json
                                npm publish
                                '''
                            }
                        }
                    }
                }
            }
        } // 'end stage('Npm publish artifact')'

    } // end 'stages'
} // end 'pipeline'
