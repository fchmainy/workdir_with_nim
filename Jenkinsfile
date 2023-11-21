pipeline {
    agent any
    
    environment {
      //DATE = new Date().format('yy.M')
      DATE = new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'", TimeZone.getTimeZone("UTC")) 
      TAG = "${DATE}.${BUILD_NUMBER}"
      repo = "git@github.com:fchmainy/convertOAS.git"
    }

    stages {
        stage('Get from NIM the Latest Working Configuration for Instance Group') {    
            steps('Get Instance Group UUID') {
                script {
                    INSTANCE_GROUP = params.instance_group
                    echo INSTANCE_GROUP
                    INSTANCE_GROUP_UUID = sh(script: 'curl -sk -H "Authorization: Basic YWRtaW46MDludHl2Nzg4dUxCTFFsZ1daTHh3Vll1RFZ4Yk9Y" https://10.1.1.7/api/platform/v1/instance-groups | jq -r ".items[] | select( .name == \\"' + INSTANCE_GROUP + '\\") | .uid"', returnStdout: true)
                }
   	        }
        }
        
        stage('Show UUID') {
            steps('UUID') {
                echo INSTANCE_GROUP_UUID    
   	        }
        }

        stage('Cloning Git') {
            steps('Checkout') {
                deleteDir()
                dir('templates') {
                    git branch: 'templates', credentialsId: 'github_ssh_key', url: "git@github.com:fchmainy/workdir_with_nim.git"    
                }
                dir('engine') {
                    git branch: 'engine', credentialsId: 'github_ssh_key', url: "git@github.com:fchmainy/workdir_with_nim.git"    
                }
                dir('instance-group') {
                    git branch: INSTANCE_GROUP, credentialsId: 'github_ssh_key', url: "git@github.com:fchmainy/workdir_with_nim.git"    
                }
           	}
        }
        
        
        stage('Config creation') {
            steps('Convert Configurations') {
                withCredentials([usernamePassword(credentialsId: 'nms', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                ansiblePlaybook(
                    playbook: './engine/create.yaml',
                    colorized: true,
                    inventory: 'hosts.ini',
                    extras: '-vvv',
                    extraVars: [
                        nms_user    : USERNAME,
                        nms_password : PASSWORD,
                        server_name : params.server_name,
                        secure      : params.secure,
                        certificate : params.certificate,
                        servers     : params.upstreams
                        ]
                )}
   	        }
        }
        
        stage('Commit') {
            steps('Convert Configurations') {
                dir('instance-group') {
                     withCredentials([sshUserPrivateKey(credentialsId: 'github_ssh_key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
                        withEnv(["GIT_SSH_COMMAND=ssh -i $SSH_KEY -o StrictHostKeyChecking=no"]) {
                            sh '''
                            git config user.email "${SSH_USER}"
                            git add *
                            git commit -am "${TAG}"
                            git log --format="%H" -n 1 > commitHash.txt
                            '''
                        }
                    }
                }
   	        }
        }
        
        stage('Post on NIM') {
            steps('Build and push') {
                dir('instance-group') {
                    script {
                        COMMIT_HASH = readFile('commitHash.txt').trim()
                        IGRP_ID = INSTANCE_GROUP_UUID.trim()
                        echo IGRP_ID
                    }
                    withCredentials([usernamePassword(credentialsId: 'nms', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                        ansiblePlaybook(
                            playbook: '../engine/push.yaml',
                            colorized: true,
                            inventory: 'hosts.ini',
                            extras: '-vvv',
                            extraVars: [
                                nms_user            : USERNAME,
                                nms_password        : PASSWORD,
                                instance_group_name : INSTANCE_GROUP,
                                instance_group_id   : IGRP_ID,
                                currentDate         : DATE,
                                commitHash          : COMMIT_HASH
                                ]
                        )}
                }
            }
        }
        
        stage('Push configuration on Git') {
            steps('Convert Configurations') {
                dir('instance-group') {
                    withCredentials([sshUserPrivateKey(credentialsId: 'github_deploy_key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
                        withEnv(["GIT_SSH_COMMAND=ssh -i $SSH_KEY -o StrictHostKeyChecking=no", "INST_GRP=${INSTANCE_GROUP}"]) {
                            sh '''
                            git push origin "${INST_GRP}"
                            '''
                        }
                    }
                }
   	        }
        }
       
    }
}
