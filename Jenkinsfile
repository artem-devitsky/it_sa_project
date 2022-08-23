pipeline {
    agent none
    stages {
        stage("Send start notification"){
            steps {
                notifyEvents message: '<p>Build <a href="$PROJECT_URL">$PROJECT_NAME</a>: #<a href="$BUILD_URL">$BUILD_NUMBER</a> status: <b>Started</b></p>', token: '${env.push_tocken}'
            }
        }
        stage("Cleanup") {            
            parallel {
                stage('Cleanup node1') {
                    agent {label 'kube-instance1' }
                    steps {
                        sh '''#!/bin/bash
                            rm -rf ~/workload_wp/*
                            rm -rf ~/workload_db/*
                        '''                    
                    }
                } 
                stage('Cleanup node2') {
                    agent {label 'kube-instance2' }
                    steps {
                        sh '''#!/bin/bash
                            rm -rf ~/workload_wp/*
                            rm -rf ~/workload_db/*
                        '''                    
                    }
                }  
            }
        }
        stage('Build images') {
            parallel {
                stage('Build WordPress image'){
                    agent {label 'kube-instance1' }
                    steps {
                        sh '''#!/bin/bash
                            branch="6.0.1"
                            #branch="6.0-branch"
                            commit_id=""
                            img_name="adziavitski/wp-front"
                            prev_img_id=$(docker images |grep "${img_name}"|grep "latest" | awk '{ print $3 }'|head -n 1)
                            prev_img_name=$(docker images |grep "${prev_img_id}" |grep -v latest | awk '{ print $1":"$2 }'|head -n 1)
                            mkdir -p ~/workload_wp
                            cd ~/workload_wp
                            if ! grep github.com ~/.ssh/known_hosts > /dev/null
                            then
                                ssh-keyscan github.com >> ~/.ssh/known_hosts
                            fi
                            git clone --depth=1 git@github.com:artem-devitsky/it-sa-project-private.git
                            #ln -sf ~/workspace/project_sa-it_main ./it-sa-project-private
                            cd it-sa-project-private/manifests/wordpress/
                            git clone --depth=1 --branch "${branch}" --single-branch git@github.com:artem-devitsky/project-WordPress.git ./app
                            rm -rf ./app/.git
                            cp -f wp-config.php ./app
                            commit_id=$(git log -1 | grep ^commit | awk '{ print $2 }')
                            if (docker build -t "${img_name}":latest .)
                            then {
                                docker tag "${img_name}":latest "${img_name}":"${commit_id}"
                                echo "=====================Comparing images==============================="
                                if [ $(docker-diff "${img_name}":latest "${prev_img_name}" |grep -v "Permission denied" |wc -l) -eq 0 ]
                                then {
                                    echo "Image ${img_name}:latest equal to ${prev_img_name}. No need to push."
                                    echo "0" > ~/workload_wp/semaphore_kube_deploy.file
                                }
                                else {
                                    echo "Image ${img_name}:latest different from ${prev_img_name}. Push started."
                                    docker push "${img_name}":"${commit_id}"    
                                    docker push "${img_name}":latest
                                    echo "1" > ~/workload_wp/semaphore_kube_deploy.file
                                }
                                fi
                            }
                            else {
                                echo "Error while building WordPress image. Look at logs"
                                exit 1
                            }
                            fi
                            '''
                    }

                }
                stage('Build DB image') {
                    agent {label 'kube-instance2' }
                    steps {
                        sh '''#!/bin/bash
                            commit_id=""
                            img_name="adziavitski/wp-db"
                            prev_img_id=$(docker images |grep "${img_name}"|grep "latest" | awk '{ print $3 }'|head -n 1)
                            prev_img_name=$(docker images |grep "${prev_img_id}" |grep -v latest | awk '{ print $1":"$2 }'|head -n 1)
                            mkdir -p ~/workload_db
                            cd ~/workload_db
                            if ! grep github.com ~/.ssh/known_hosts > /dev/null
                            then {
                                ssh-keyscan github.com >> ~/.ssh/known_hosts
                            }
                            fi
                            git clone --depth=1 git@github.com:artem-devitsky/it-sa-project-private.git
                            #ln -sf ~/workspace/project_sa-it_main ./it-sa-project-private
                            cd it-sa-project-private/manifests/db/
                            commit_id=$(git log -1 | grep ^commit | awk '{ print $2 }')
                            if (docker build -t "${img_name}":latest .)
                            then {
                                docker tag "${img_name}":latest adziavitski/wp-db:"${commit_id}"
                                if [ $(docker-diff "${img_name}":latest "${prev_img_name}" |grep -v "Permission denied" |wc -l) -eq 0 ]
                                then {
                                    echo "Image ${img_name}:latest equal to ${prev_img_name}. No need to push."
                                    echo "0" > ~/workload_db/semaphore_kube_deploy.file
                                }
                                else {
                                    echo "Image ${img_name}:latest different from ${prev_img_name}. Push started."
                                    docker push "${img_name}":"${commit_id}"    
                                    docker push "${img_name}":latest
                                    echo "1" > ~/workload_db/semaphore_kube_deploy.file
                                }
                                fi
                            }
                            else {
                                echo "Error while building db image. Look at logs"
                                exit 1
                            }
                            fi
                            '''
                    }
                }
            }
        }
        stage('Deploy to kubernetes') {
            parallel{
                stage('Deploy WordPress to kubernetes') {
                    agent {label 'kube-instance1' }
                    steps {
                        sh '''#!/bin/bash
                            if [ -f ~/workload_wp/semaphore_kube_deploy.file ]
                            then {
                                if [ $(cat ~/workload_wp/semaphore_kube_deploy.file) -eq 1 ]
                                then {
                                    echo "Updating kubernetes WordPress deployment from manifests"
                                    cd ~/workload_wp/it-sa-project-private/manifests/k8s/wordpress/
                                    kubectl apply -k ./
                                    echo "Deploying new WordPress-frontend image on kubernetes"
                                    kubectl rollout restart deploy wordpress -n project
                                }
                                else {
                                    echo "Updating kubernetes WordPress deployment from manifests"
                                    cd ~/workload_wp/it-sa-project-private/manifests/k8s/wordpress/
                                    kubectl apply -k ./
                                    echo "No need to deploy WordPress-frontend image om kubernetes"
                                }
                                fi
                            }
                            else {
                                echo "No need to update WordPress-frontend on kubernetes"
                            }
                            fi
                        '''
                    }
                }
                stage('Deploy DB to kubernetes') {
                    agent {label 'kube-instance2' }
                    steps {
                        sh '''#!/bin/bash
                            if [ -f ~/workload_db/semaphore_kube_deploy.file ]
                            then {
                                if [ $(cat ~/workload_db/semaphore_kube_deploy.file) -eq 1 ]
                                then {
                                   echo "Updating kubernetes WordPress-MySQL deployment from manifests"
                                   cd ~/workload_wp/it-sa-project-private/manifests/k8s/wordpress/
                                   kubectl apply -k ./
                                   echo "Deploying new WordPress-MySQL image on kubernetes"
                                   kubectl rollout restart deploy wordpress-mysql -n project
                                }
                                else {
                                    echo "Updating kubernetes WordPress-MySQL deployment from manifests"
                                    cd ~/workload_wp/it-sa-project-private/manifests/k8s/wordpress/
                                    kubectl apply -k ./
                                    echo "No need to deploy WordPress-MySQL image on kubernetes"
                                }
                                fi
                            }
                            else {
                                echo "No need to update WordPress-MySQL on kubernetes"
                            }
                            fi
                        '''
                    }
                }
            }
        }
        stage("Cleanup at the end") {
            parallel {
                stage('Cleanup node1') {
                    agent {label 'kube-instance1' }
                    steps {
                        sh '''#!/bin/bash
                            rm -rf ~/workload_wp/*
                            rm -rf ~/workload_db/*
                        '''                    
                    }
                } 
                stage('Cleanup node2') {
                    agent {label 'kube-instance2' }
                    steps {
                        sh '''#!/bin/bash
                            rm -rf ~/workload_wp/*
                            rm -rf ~/workload_db/*
                        '''                    
                    }
                }  
            }
        }
    }
    post {
        always {
            notifyEvents message: '<p>Build <a href="$PROJECT_URL">$PROJECT_NAME</a>: #<a href="$BUILD_URL">$BUILD_NUMBER</a> status: <b>$BUILD_STATUS</b></p><p><a href="$BUILD_URL/console">Build log</a></p>', token: '${env.push_tocken}'
        }
    }
}
