properties([
  parameters([
    string(name: 'PARAM1', defaultValue: 'default', description: 'Example Parameter'),
    booleanParam(name: 'ENABLE_FEATURE', defaultValue: false, description: 'Enable Feature')
  ])
])
pipeline {
    agent any
    environment {
        LANGUAGE = 'en_US.en'
        LANG = 'en_US.UTF-8'
        LC_ALL = 'en_US.UTF-8'
    }
    parameters {
        string(name: 'Hosts', defaultValue: '', description: 'Comma-separated list of hosts to deploy')
        string(name: 'Jobs', defaultValue: '', description: 'Comma-separated list of jobs to perform')
    }
    stages {
        stage('Validation') {
            steps {
                script {
                    if (!params.Hosts?.trim()) {
                        error("Please select the boxes to deploy.")
                    }
                    if (!params.Jobs?.trim()) {
                        error("Please select at least one job to perform.")
                    }
                }
            }
        }
        stage('Setup') {
            steps {
                script {
                    // Parse Hosts and Jobs into arrays
                    def hostArray = params.Hosts.split(',').collect { it.trim() }
                    def jobArray = params.Jobs.split(',').collect { it.trim() }

                    // Initialize variables
                    branch = 'pre-prod'
                    uatServers = ['myntra-uat']
                    myntraServers = ['myntra-qa', 'qa-automation', 'myntra-uat']

                    // Pass parsed arrays to the next stages
                    env.HOST_ARRAY = hostArray.join(',')
                    env.JOB_ARRAY = jobArray.join(',')
                    env.UAT_SERVERS = uatServers.join(',')
                    env.MYNTRA_SERVERS = myntraServers.join(',')
                }
            }
        }
        stage('Deploy to Hosts') {
            steps {
                script {
                    def hostArray = env.HOST_ARRAY.split(',')
                    def jobArray = env.JOB_ARRAY.split(',')

                    // Iterate over each host
                    for (hostname in hostArray) {
                        def folder = 'voiro'
                        def docker = 'phoenix'
                        def isUatServer = env.UAT_SERVERS.contains(hostname)
                        def isMyntraServer = env.MYNTRA_SERVERS.contains(hostname)

                        // Set branch and docker configurations
                        if (hostname == 'dstv-staging' || hostname == 'rooter-replica') {
                            folder = 'ubuntu'
                            docker = 'phoenix_merge'
                        }
                        if (hostname == 'myntra-uat') {
                            docker = 'phoenix_merge'
                        }
                        if (isUatServer) {
                            branch = 'primary-uat'
                        }

                        // Execute jobs for each host
                        for (job in jobArray) {
                            echo "Executing job '${job}' on host '${hostname}' with branch '${branch}'"

                            switch (job) {
                                case 'Build and Deploy FE':
                                    echo "Building and deploying FE..."
                                    // Simulate Windows commands for build and deploy
                                    bat """
                                        git reset --hard
                                        git checkout primary
                                        git branch -D ${branch} || echo Branch does not exist locally. Ignore.
                                        git checkout ${branch}
                                        npm install -f
                                        git reset --hard
                                        set VERSION=${branch}-${BUILD_NUMBER}
                                        sed -i "s/{{version}}/%VERSION%/g" src\\app\\app.component.ts
                                        ng build
                                        cd dist\\voiro-frontend
                                        zip -r %VERSION%.zip browser
                                        aws s3 cp %VERSION%.zip s3://phoenix-fe-builds/
                                    """
                                    break

                                case 'Deploy BE':
                                    echo "Deploying BE..."
                                    if (hostname == 'dstv-staging') {
                                        branch = 'dstv_staging'
                                    }
                                    bat """
                                        git pull origin ${branch} --ff -f
                                    """
                                    break

                                case 'Migrate DB':
                                    echo "Migrating database..."
                                    bat """
                                        docker exec ${docker} /bin/bash -c "python manage.py makemigrations --merge"
                                        docker exec ${docker} /bin/bash -c "python manage.py migrate"
                                    """
                                    break

                                case 'Restart Celery':
                                    echo "Restarting Celery..."
                                    if (isMyntraServer) {
                                        // Custom commands for Myntra servers
                                        bat """
                                            docker exec ${docker} /bin/bash -c "cat /var/run/celery/thinktank_project_others.pid | xargs kill -term" || echo No PID found.
                                            docker exec ${docker} /bin/bash -c "celery -A thinktank_project worker -Q general_queue --detach"
                                        """
                                    } else {
                                        bat """
                                            docker exec ${docker} /bin/bash -c "celery worker --beat --detach"
                                        """
                                    }
                                    break

                                case 'Restart Server':
                                    echo "Restarting server..."
                                    bat """
                                        docker exec ${docker} /bin/bash -c "apachectl graceful"
                                    """
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
            echo "Deployment completed."
        }
        failure {
            echo "Deployment failed. Check logs for details."
        }
    }
}
