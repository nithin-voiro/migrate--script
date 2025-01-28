pipeline {
    agent any

    environment {
        LANG = "en_US.UTF-8"
        LC_ALL = "en_US.UTF-8"
        BRANCH = "pre-prod"
        UAT_SERVERS = "myntra-uat" // Changed to a string
        MYNTRA_SERVERS = "myntra-qa,qa-automation,myntra-uat" // Changed to a string
    }

    parameters {
        string(name: 'Hosts', description: 'Comma-separated list of hosts to deploy')
        string(name: 'Jobs', description: 'Comma-separated list of jobs to perform')
        string(name: 'BUILD_NUMBER', defaultValue: '1', description: 'Build number')
    }

    stages {
        stage('Validation') {
            steps {
                script {
                    if (!params.Hosts) {
                        error "Please select the boxes to deploy."
                    }

                    if (!params.Jobs) {
                        error "Please select at least 1 job to perform."
                    }
                }
            }
        }

        stage('Preparation') {
            steps {
                script {
                    def hostArray = params.Hosts.split(',')
                    def jobsArray = params.Jobs.split(',')
                    env.HOSTS = hostArray
                    env.JOBS = jobsArray

                    // Parse UAT and MYNTRA server lists
                    def uatServers = UAT_SERVERS.split(',')
                    def myntraServers = MYNTRA_SERVERS.split(',')

                    env.UAT_SERVERS_LIST = uatServers
                    env.MYNTRA_SERVERS_LIST = myntraServers
                }
            }
        }

        stage('Deployment') {
            steps {
                script {
                    env.HOSTS.each { hostname ->
                        def folder = "voiro"
                        def docker = "phoenix"

                        if (hostname in ['dstv-staging', 'rooter-replica']) {
                            folder = "ubuntu"
                            docker = "phoenix_merge"
                        } else if (hostname == 'myntra-uat') {
                            docker = "phoenix_merge"
                        }

                        if (hostname in env.UAT_SERVERS_LIST) {
                            env.BRANCH = "primary-uat"
                        }

                        env.JOBS.each { job ->
                            switch (job) {
                                case "Build and Deploy FE":
                                    buildAndDeployFE(hostname, folder, docker)
                                    break
                                case "Deploy BE":
                                    deployBE(hostname, folder, docker)
                                    break
                                case "Migrate DB":
                                    migrateDB(hostname, docker)
                                    break
                                case "Restart Celery":
                                    restartCelery(hostname, docker)
                                    break
                                case "Restart Server":
                                    restartServer(hostname, docker)
                                    break
                                default:
                                    echo "Unknown job: ${job}"
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}

// Function Definitions

void buildAndDeployFE(String hostname, String folder, String docker) {
    stage("Build and Deploy FE on ${hostname}") {
        steps {
            sh '''
                git reset --hard
                git checkout primary
                git branch -D ${BRANCH} || echo "${BRANCH} does not exist in local. Ignore."
                git checkout ${BRANCH}
                npm install -f
                git reset --hard
                version=${BRANCH}-${BUILD_NUMBER}
                sed -i "s/{{version}}/${version}/g" src/app/app.component.ts
                ng build
                cd dist/voiro-frontend
                mv 3rdpartylicenses.txt browser/3rdpartylicenses.txt
                zip ${version}.zip -r browser
                aws s3 cp ${version}.zip s3://phoenix-fe-builds/ --region ap-south-1
                git reset --hard
            '''
        }
    }
}

void deployBE(String hostname, String folder, String docker) {
    stage("Deploy BE on ${hostname}") {
        steps {
            sh '''
                ssh -i /var/lib/jenkins/.ssh/build.key voiro@${hostname}.voiro.com "cd /home/${folder}/code; git pull origin ${BRANCH} --ff -f"
            '''
        }
    }
}

void migrateDB(String hostname, String docker) {
    stage("Migrate DB on ${hostname}") {
        steps {
            sh '''
                ssh -i /var/lib/jenkins/.ssh/build.key voiro@${hostname}.voiro.com "docker exec ${docker} /bin/bash -c \"cd /code/thinktank_project; export LC_ALL=en_US.UTF-8; export LANG=en_US.UTF-8; yes | python manage.py makemigrations --merge\""
                ssh -i /var/lib/jenkins/.ssh/build.key voiro@${hostname}.voiro.com "docker exec ${docker} /bin/bash -c \"python manage.py migrate\""
            '''
        }
    }
}

void restartCelery(String hostname, String docker) {
    stage("Restart Celery on ${hostname}") {
        steps {
            sh '''
                ssh -i /var/lib/jenkins/.ssh/build.key voiro@${hostname}.voiro.com "docker exec ${docker} /bin/bash -c \"export LC_ALL=en_US.UTF-8; export LANG=en_US.UTF-8; celery -A thinktank_project worker -l info --detach\""
            '''
        }
    }
}

void restartServer(String hostname, String docker) {
    stage("Restart Server on ${hostname}") {
        steps {
            sh '''
                ssh -i /var/lib/jenkins/.ssh/build.key voiro@${hostname}.voiro.com "docker exec ${docker} /bin/bash -c \"export LC_ALL=en_US.UTF-8; export LANG=en_US.UTF-8; apachectl graceful\""
            '''
        }
    }
}


