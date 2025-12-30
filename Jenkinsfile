pipeline {
    agent any

    environment {
        // Maven e Java
        MAVEN_HOME = '/usr/share/maven'
        JAVA_HOME  = '/usr/lib/jvm/java-17-openjdk-amd64'

        // Git
        GIT_REPO_URL   = 'https://github.com/DiogoAntunes1211165/lms-authnusers.git'
        GIT_BRANCH    = 'main'
        CREDENTIALS_ID = 'password_for_github_tiago'

        // Docker Registry
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_REGISTRY_CREDENTIALS = 'dockerhub-credentials'
        DOCKER_REGISTRY_NAMESPACE = 'tiagomiguel55'

        // Docker
        IMAGE_NAME = 'lmsauthnusers'
        IMAGE_TAG  = "${GIT_COMMIT}"
        FULL_IMAGE_NAME = "${DOCKER_REGISTRY}/${DOCKER_REGISTRY_NAMESPACE}/${IMAGE_NAME}"

        // Swarm stacks
        STACK_NAME_DEV     = 'lmsauthnusers-dev'
        STACK_NAME_STAGING = 'lmsauthnusers-staging'
        STACK_NAME_PROD    = 'lmsauthnusers-prod'

        // Email
        EMAIL_RECIPIENT = 'your.email@gmail.com'
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Deployment environment'
        )
        booleanParam(
            name: 'SKIP_DEPLOY',
            defaultValue: false,
            description: 'Skip deployment'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: "${GIT_BRANCH}",
                    url: "${GIT_REPO_URL}",
                    credentialsId: "${CREDENTIALS_ID}"
            }
        }

        stage('Build & Package') {
            when {
                expression { params.ENVIRONMENT != 'prod' }
            }
            steps {
                sh """
                    ${MAVEN_HOME}/bin/mvn clean package -DskipTests
                """
            }
        }

        stage('Build Docker Image') {
            when {
                expression { params.ENVIRONMENT != 'prod' }
            }
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:${params.ENVIRONMENT}
                """
            }
        }

        stage('Push Docker Image to Registry') {
            when {
                expression { params.ENVIRONMENT == 'dev' || params.ENVIRONMENT == 'staging' }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh """
                            echo "=========================================="
                            echo "Pushing image to registry..."
                            echo "Environment: ${params.ENVIRONMENT}"
                            echo "Registry: ${DOCKER_REGISTRY}"
                            echo "Namespace: ${DOCKER_REGISTRY_NAMESPACE}"
                            echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
                            echo "=========================================="

                            echo "\$DOCKER_PASSWORD" | docker login ${DOCKER_REGISTRY} -u "\$DOCKER_USERNAME" --password-stdin

                            echo ""
                            echo "Tagging for registry..."
                            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}:${IMAGE_TAG}
                            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}:${params.ENVIRONMENT}

                            echo ""
                            echo "Pushing image..."
                            docker push ${FULL_IMAGE_NAME}:${IMAGE_TAG}
                            docker push ${FULL_IMAGE_NAME}:${params.ENVIRONMENT}

                            echo ""
                            echo "=========================================="
                            echo "✅ Image pushed to registry!"
                            echo "   ${FULL_IMAGE_NAME}:${IMAGE_TAG}"
                            echo "   ${FULL_IMAGE_NAME}:${params.ENVIRONMENT}"
                            echo "=========================================="
                        """
                    }
                }
            }
        }

        stage('Pull Docker Image from Registry') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh """
                            echo "=========================================="
                            echo "Pulling image from registry for Production..."
                            echo "Registry: ${DOCKER_REGISTRY}"
                            echo "Image: ${FULL_IMAGE_NAME}:${IMAGE_TAG}"
                            echo "=========================================="

                            echo "\$DOCKER_PASSWORD" | docker login ${DOCKER_REGISTRY} -u "\$DOCKER_USERNAME" --password-stdin

                            echo ""
                            echo "Pulling image..."
                            docker pull ${FULL_IMAGE_NAME}:${IMAGE_TAG}

                            echo ""
                            echo "Tagging as prod and local name..."
                            docker tag ${FULL_IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:${IMAGE_TAG}
                            docker tag ${FULL_IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}:prod

                            echo ""
                            echo "Pushing prod tag to registry..."
                            docker push ${FULL_IMAGE_NAME}:prod

                            echo ""
                            echo "=========================================="
                            echo "✅ Production image ready!"
                            echo "   Local: ${IMAGE_NAME}:${IMAGE_TAG}"
                            echo "   Registry: ${FULL_IMAGE_NAME}:prod"
                            echo "=========================================="
                        """
                    }
                }
            }
        }

        stage('Initialize Docker Swarm') {
            when {
                expression { params.SKIP_DEPLOY == false }
            }
            steps {
                sh '''
                    docker info | grep -q "Swarm: active" || docker swarm init || true
                    docker network ls | grep -q lms_network || \
                    docker network create --driver overlay --attachable lms_network || true
                '''
            }
        }

        stage('Deploy Shared Infrastructure') {
            when {
                expression { params.SKIP_DEPLOY == false }
            }
            steps {
                sh """
                    echo "=========================================="
                    echo "Deploying Shared Infrastructure (PostgreSQL + RabbitMQ)"
                    echo "Environment: ${params.ENVIRONMENT}"
                    echo "=========================================="

                    ENVIRONMENT="${params.ENVIRONMENT}"
                    STACK_NAME="lms_shared_\${ENVIRONMENT}"

                    # Define environment-specific ports
                    case "\${ENVIRONMENT}" in
                        dev)
                            POSTGRES_PORT=5432
                            RABBITMQ_PORT=5672
                            RABBITMQ_MGMT_PORT=15672
                            POSTGRES_VOLUME="postgres_data_dev"
                            RABBITMQ_VOLUME="rabbitmq_data_dev"
                            ;;
                        staging)
                            POSTGRES_PORT=5433
                            RABBITMQ_PORT=5673
                            RABBITMQ_MGMT_PORT=15673
                            POSTGRES_VOLUME="postgres_data_staging"
                            RABBITMQ_VOLUME="rabbitmq_data_staging"
                            ;;
                        prod)
                            POSTGRES_PORT=5434
                            RABBITMQ_PORT=5674
                            RABBITMQ_MGMT_PORT=15674
                            POSTGRES_VOLUME="postgres_data_prod"
                            RABBITMQ_VOLUME="rabbitmq_data_prod"
                            ;;
                        *)
                            echo "❌ ERROR: Unknown environment: \${ENVIRONMENT}"
                            exit 1
                            ;;
                    esac

                    echo "Environment: \${ENVIRONMENT}"
                    echo "Stack Name: \${STACK_NAME}"
                    echo "PostgreSQL Port: \${POSTGRES_PORT}"
                    echo "RabbitMQ Port: \${RABBITMQ_PORT}"
                    echo "RabbitMQ Management Port: \${RABBITMQ_MGMT_PORT}"
                    echo ""

                    # Generate environment-specific docker-compose file
                    COMPOSE_FILE="${WORKSPACE}/docker-compose-shared-\${ENVIRONMENT}-generated.yml"

                    cat > "\${COMPOSE_FILE}" <<EOF
version: '3.8'

services:
  postgres:
    image: postgres:latest
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - \${POSTGRES_VOLUME}:/var/lib/postgresql
    ports:
      - "\${POSTGRES_PORT}:5432"
    networks:
      - lms_network
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - \${RABBITMQ_VOLUME}:/var/lib/rabbitmq
    ports:
      - "\${RABBITMQ_PORT}:5672"
      - "\${RABBITMQ_MGMT_PORT}:15672"
    networks:
      - lms_network
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  lms_network:
    external: true

volumes:
  \${POSTGRES_VOLUME}:
  \${RABBITMQ_VOLUME}:
EOF

                    echo "✅ Generated compose file: \${COMPOSE_FILE}"
                    echo ""

                    # Deploy or update stack
                    if docker stack ls | grep -q "\${STACK_NAME}"; then
                        echo "✅ \${STACK_NAME} stack already deployed - updating..."
                        docker stack deploy -c "\${COMPOSE_FILE}" \${STACK_NAME}
                        echo "Waiting 15 seconds for update to stabilize..."
                        sleep 15
                    else
                        echo "Deploying \${STACK_NAME} stack..."
                        docker stack deploy -c "\${COMPOSE_FILE}" \${STACK_NAME}
                        echo "Waiting 30 seconds for services to start..."
                        sleep 30
                    fi

                    echo ""
                    echo "=========================================="
                    echo "✅ Shared Infrastructure deployed for \${ENVIRONMENT}"
                    echo "   - PostgreSQL: localhost:\${POSTGRES_PORT}"
                    echo "   - RabbitMQ: localhost:\${RABBITMQ_PORT}"
                    echo "   - RabbitMQ Management: http://localhost:\${RABBITMQ_MGMT_PORT}"
                    echo "   - Service names: postgres, rabbitmq"
                    echo "=========================================="
                """
            }
        }

        stage('Deploy to Dev') {
            when {
                allOf {
                    expression { params.ENVIRONMENT == 'dev' }
                    expression { params.SKIP_DEPLOY == false }
                }
            }
            steps {
                sh """
                    echo "=========================================="
                    echo "Deploying to DEV environment"
                    echo "=========================================="

                    # Generate dev-specific docker-compose file
                    COMPOSE_FILE="${WORKSPACE}/docker-compose-swarm-dev-generated.yml"

                    cat > "\${COMPOSE_FILE}" <<EOF
version: '3.8'

services:
  lmsauthnusers:
    image: ${IMAGE_NAME}:${IMAGE_TAG}
    environment:
      - SPRING_PROFILES_ACTIVE=bootstrap
      - spring.datasource.url=jdbc:postgresql://postgres:5432/postgres
      - spring.datasource.username=postgres
      - spring.datasource.password=password
      - spring.datasource.driver-class-name=org.postgresql.Driver
      - spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
      - spring.rabbitmq.host=rabbitmq
      - bootstrap.mode=static
    volumes:
      - uploaded_files_dev:/tmp
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 5
        window: 120s
    ports:
      - "8090:8080"
    networks:
      - lms_network

networks:
  lms_network:
    external: true

volumes:
  uploaded_files_dev:
EOF

                    docker stack deploy -c "\${COMPOSE_FILE}" ${STACK_NAME_DEV}

                    echo ""
                    echo "Waiting for services to stabilize..."
                    sleep 20

                    echo ""
                    echo "=========================================="
                    echo "✅ DEV deployment completed"
                    echo "   Access: http://localhost:8090"
                    echo "   Swagger: http://localhost:8090/swagger-ui"
                    echo "=========================================="
                """
            }
        }

        stage('Deploy to Staging') {
            when {
                allOf {
                    expression { params.ENVIRONMENT == 'staging' }
                    expression { params.SKIP_DEPLOY == false }
                }
            }
            steps {
                sh """
                    echo "=========================================="
                    echo "Deploying to STAGING environment"
                    echo "=========================================="

                    # Generate staging-specific docker-compose file
                    COMPOSE_FILE="${WORKSPACE}/docker-compose-swarm-staging-generated.yml"

                    cat > "\${COMPOSE_FILE}" <<EOF
version: '3.8'

services:
  lmsauthnusers:
    image: ${IMAGE_NAME}:${IMAGE_TAG}
    environment:
      - SPRING_PROFILES_ACTIVE=bootstrap
      - spring.datasource.url=jdbc:postgresql://postgres:5432/postgres
      - spring.datasource.username=postgres
      - spring.datasource.password=password
      - spring.datasource.driver-class-name=org.postgresql.Driver
      - spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
      - spring.rabbitmq.host=rabbitmq
      - bootstrap.mode=static
    volumes:
      - uploaded_files_staging:/tmp
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 5
        window: 120s
    ports:
      - "8091:8080"
    networks:
      - lms_network

networks:
  lms_network:
    external: true

volumes:
  uploaded_files_staging:
EOF

                    docker stack deploy -c "\${COMPOSE_FILE}" ${STACK_NAME_STAGING}

                    echo ""
                    echo "Waiting for services to stabilize..."
                    sleep 20

                    echo ""
                    echo "=========================================="
                    echo "✅ STAGING deployment completed"
                    echo "   Access: http://localhost:8091"
                    echo "   Swagger: http://localhost:8091/swagger-ui"
                    echo "=========================================="
                """
            }
        }

        stage('Manual Approval for Production') {
            when {
                allOf {
                    expression { params.ENVIRONMENT == 'prod' }
                    expression { params.SKIP_DEPLOY == false }
                }
            }
            steps {
                script {
                    echo "=========================================="
                    echo "PRODUCTION DEPLOYMENT APPROVAL REQUIRED"
                    echo "=========================================="
                    echo "Project: LMS AuthN and Users Service"
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "Image Tag: ${IMAGE_TAG}"
                    echo "Environment: PRODUCTION"
                    echo "Deployment Mode: Docker Swarm - Rolling Update (3 replicas)"
                    echo ""
                    echo "Rolling Update Strategy:"
                    echo "  - Update 1 container at a time"
                    echo "  - 10-second delay between updates"
                    echo "  - Health checks before proceeding"
                    echo "  - Automatic rollback on failure"
                    echo "=========================================="

                    try {
                        mail(
                            to: "${EMAIL_RECIPIENT}",
                            subject: "Jenkins: Production Deployment Approval Required - Build #${BUILD_NUMBER}",
                            body: """
                                Production Deployment Approval Required
                                =======================================

                                Project: LMS AuthN and Users Service
                                Build Number: ${BUILD_NUMBER}
                                Image Tag: ${IMAGE_TAG}
                                Environment: PRODUCTION
                                Deployment Mode: Docker Swarm - Rolling Update (3 replicas)

                                Rolling Update Strategy:
                                - Update 1 container at a time
                                - 10-second delay between updates
                                - Health checks before proceeding
                                - Automatic rollback on failure

                                Please review and approve/reject the deployment to production.

                                Approve or reject at: ${BUILD_URL}input/

                                View full build details: ${BUILD_URL}
                            """,
                            mimeType: 'text/plain'
                        )
                        echo "✅ Email notification sent successfully to ${EMAIL_RECIPIENT}"
                    } catch (Exception e) {
                        echo "⚠️ WARNING: Could not send email notification"
                        echo "Email error: ${e.message}"
                        echo "Continuing with manual approval process..."
                    }

                    timeout(time: 30, unit: 'MINUTES') {
                        input message: 'Approve Production Deployment?', ok: 'Deploy'
                    }
                }
            }
        }

        stage('Deploy to Production (Rolling Update + Rollback)') {
            when {
                allOf {
                    expression { params.ENVIRONMENT == 'prod' }
                    expression { params.SKIP_DEPLOY == false }
                }
            }
            steps {
                script {
                    echo "Starting PROD rolling update with image ${IMAGE_NAME}:${IMAGE_TAG}"

                    sh """
                        # Generate production docker-compose file
                        COMPOSE_FILE="${WORKSPACE}/docker-compose-swarm-prod-generated.yml"

                        cat > "\${COMPOSE_FILE}" <<EOF
version: '3.8'

services:
  lmsauthnusers:
    image: ${IMAGE_NAME}:${IMAGE_TAG}
    environment:
      - SPRING_PROFILES_ACTIVE=bootstrap
      - spring.datasource.url=jdbc:postgresql://postgres:5432/postgres
      - spring.datasource.username=postgres
      - spring.datasource.password=password
      - spring.datasource.driver-class-name=org.postgresql.Driver
      - spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
      - spring.rabbitmq.host=rabbitmq
      - bootstrap.mode=static
    volumes:
      - uploaded_files_prod:/tmp
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
    ports:
      - "8092:8080"
    networks:
      - lms_network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

networks:
  lms_network:
    external: true

volumes:
  uploaded_files_prod:
EOF

                        # Check if service already exists
                        SERVICE_NAME=\$(docker service ls --filter "label=com.docker.stack.namespace=${STACK_NAME_PROD}" --filter "name=${STACK_NAME_PROD}_lmsauthnusers" --format "{{.Name}}" 2>/dev/null || echo "")

                        if [ -z "\$SERVICE_NAME" ]; then
                            echo "=========================================="
                            echo "⚠️ Service does not exist yet - First deployment"
                            echo "Creating stack for the first time..."
                            echo "=========================================="

                            docker stack deploy -c "\${COMPOSE_FILE}" ${STACK_NAME_PROD}

                            echo "Waiting for services to start..."
                            sleep 30

                            SERVICE_NAME=\$(docker service ls --filter "label=com.docker.stack.namespace=${STACK_NAME_PROD}" --filter "name=${STACK_NAME_PROD}_lmsauthnusers" --format "{{.Name}}")

                            if [ -z "\$SERVICE_NAME" ]; then
                                echo "❌ ERROR: Service was not created!"
                                exit 1
                            fi

                            EXPECTED_REPLICAS=\$(docker service inspect \$SERVICE_NAME --format '{{.Spec.Mode.Replicated.Replicas}}')
                            echo "Expected replicas: \$EXPECTED_REPLICAS"

                            MAX_WAIT=180
                            ELAPSED=0

                            while [ \$ELAPSED -lt \$MAX_WAIT ]; do
                                RUNNING_REPLICAS=\$(docker service ps \$SERVICE_NAME --filter "desired-state=running" --format "{{.CurrentState}}" 2>/dev/null | grep -c "Running")
                                echo "Running replicas: \$RUNNING_REPLICAS/\$EXPECTED_REPLICAS (elapsed: \${ELAPSED}s)"

                                if [ \$RUNNING_REPLICAS -ge \$EXPECTED_REPLICAS ]; then
                                    echo "✅ All replicas are running - first deployment successful!"
                                    break
                                fi

                                sleep 10
                                ELAPSED=\$((ELAPSED + 10))
                            done

                            if [ \$RUNNING_REPLICAS -lt \$EXPECTED_REPLICAS ]; then
                                echo "❌ Not all replicas started in time"
                                docker service logs --tail 50 \$SERVICE_NAME || true
                                exit 1
                            fi

                        else
                            echo "=========================================="
                            echo "✓ Service exists - Performing Rolling Update"
                            echo "Service name: \$SERVICE_NAME"
                            echo "=========================================="

                            OLD_IMAGE=\$(docker service inspect \$SERVICE_NAME --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}')
                            echo "Old image: \$OLD_IMAGE"
                            echo "New image: ${IMAGE_NAME}:${IMAGE_TAG}"

                            docker stack deploy -c "\${COMPOSE_FILE}" ${STACK_NAME_PROD}

                            echo "Docker Swarm is performing rolling update..."
                            echo "Monitoring service update progress..."

                            sleep 20

                            EXPECTED_REPLICAS=\$(docker service inspect \$SERVICE_NAME --format '{{.Spec.Mode.Replicated.Replicas}}')
                            echo "Expected replicas: \$EXPECTED_REPLICAS"

                            MAX_WAIT=300
                            ELAPSED=0
                            UPDATE_COMPLETE=false

                            while [ \$ELAPSED -lt \$MAX_WAIT ]; do
                                STATUS=\$(docker service inspect \$SERVICE_NAME --format '{{.UpdateStatus.State}}' 2>/dev/null || echo "")
                                RUNNING_REPLICAS=\$(docker service ps \$SERVICE_NAME --filter "desired-state=running" --format "{{.CurrentState}}" 2>/dev/null | grep -c "Running" || echo "0")

                                echo "Update status: \${STATUS:-none} | Running replicas: \$RUNNING_REPLICAS/\$EXPECTED_REPLICAS | Elapsed: \${ELAPSED}s"

                                if [ "\$STATUS" = "completed" ]; then
                                    echo "✅ Rolling update completed successfully!"
                                    UPDATE_COMPLETE=true
                                    break
                                elif [ \$RUNNING_REPLICAS -ge \$EXPECTED_REPLICAS ] && [ \$ELAPSED -gt 30 ]; then
                                    echo "✅ All replicas running - deployment successful!"
                                    UPDATE_COMPLETE=true
                                    break
                                elif [ "\$STATUS" = "rollback_completed" ]; then
                                    echo "❌ Swarm automatic rollback completed!"
                                    docker service logs --tail 50 \$SERVICE_NAME || true
                                    exit 1
                                elif [ "\$STATUS" = "paused" ]; then
                                    echo "❌ Update paused - rolling back manually to \$OLD_IMAGE..."
                                    docker service update --image \$OLD_IMAGE \$SERVICE_NAME
                                    docker service logs --tail 50 \$SERVICE_NAME || true
                                    exit 1
                                fi

                                docker service ps \$SERVICE_NAME --filter "desired-state=running" --format "table {{.Name}}\t{{.CurrentState}}\t{{.Error}}" 2>/dev/null || true

                                sleep 10
                                ELAPSED=\$((ELAPSED + 10))
                            done

                            if [ "\$UPDATE_COMPLETE" = "false" ]; then
                                echo "⚠️ Rolling update timed out. Checking final status..."
                                FINAL_RUNNING=\$(docker service ps \$SERVICE_NAME --filter "desired-state=running" --format "{{.CurrentState}}" 2>/dev/null | grep -c "Running" || echo "0")

                                if [ \$FINAL_RUNNING -ge \$EXPECTED_REPLICAS ]; then
                                    echo "✅ All replicas confirmed running - deployment successful!"
                                    UPDATE_COMPLETE=true
                                else
                                    echo "❌ Not all replicas running. Rolling back to \$OLD_IMAGE..."
                                    docker service update --image \$OLD_IMAGE \$SERVICE_NAME
                                    sleep 20
                                    docker service logs --tail 50 \$SERVICE_NAME || true
                                    docker service ps \$SERVICE_NAME || true
                                    exit 1
                                fi
                            fi
                        fi

                        echo "=========================================="
                        echo "Production deployment completed successfully!"
                        echo "All replicas running version ${IMAGE_TAG}"
                        echo "=========================================="

                        docker stack services ${STACK_NAME_PROD}
                        docker stack ps ${STACK_NAME_PROD}
                    """
                }
            }
        }

        stage('Post-Deployment Verification') {
            when {
                expression { params.SKIP_DEPLOY == false }
            }
            steps {
                script {
                    def stackName = params.ENVIRONMENT == 'dev' ? env.STACK_NAME_DEV :
                                   (params.ENVIRONMENT == 'staging' ? env.STACK_NAME_STAGING : env.STACK_NAME_PROD)

                    sh """
                        echo "=========================================="
                        echo "POST-DEPLOYMENT VERIFICATION"
                        echo "Environment: ${params.ENVIRONMENT}"
                        echo "=========================================="

                        echo ""
                        echo "Waiting 15 seconds for Docker Swarm to update replica status..."
                        sleep 15

                        echo ""
                        echo "Current service status:"
                        docker stack services ${stackName}

                        echo ""
                        echo "Service tasks:"
                        docker stack ps ${stackName} --no-trunc

                        echo ""
                        echo "=========================================="
                        echo "✅ Post-deployment verification completed"
                        echo "=========================================="
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                echo "=========================================="
                echo "Pipeline completed successfully!"
                echo "Environment: ${params.ENVIRONMENT}"
                echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
                echo "=========================================="

                try {
                    emailext(
                        subject: "SUCCESS: LMS AuthN and Users - ${params.ENVIRONMENT} - Build #${BUILD_NUMBER}",
                        body: """
                            <h2>Build Successful</h2>
                            <p><strong>Project:</strong> LMS AuthN and Users Service</p>
                            <p><strong>Environment:</strong> ${params.ENVIRONMENT}</p>
                            <p><strong>Build Number:</strong> ${BUILD_NUMBER}</p>
                            <p><strong>Image Tag:</strong> ${IMAGE_TAG}</p>
                            <p><strong>Status:</strong> ✅ SUCCESS</p>
                            <br>
                            <p><a href="${BUILD_URL}">View Build Details</a></p>
                        """,
                        to: "${EMAIL_RECIPIENT}",
                        mimeType: 'text/html'
                    )
                    echo "✅ Success notification email sent to ${EMAIL_RECIPIENT}"
                } catch (Exception e) {
                    echo "⚠️ WARNING: Could not send success email notification"
                    echo "Email error: ${e.message}"
                }
            }
        }

        failure {
            script {
                echo "=========================================="
                echo "Pipeline failed!"
                echo "Environment: ${params.ENVIRONMENT}"
                echo "=========================================="

                try {
                    emailext(
                        subject: "FAILURE: LMS AuthN and Users - ${params.ENVIRONMENT} - Build #${BUILD_NUMBER}",
                        body: """
                            <h2>Build Failed</h2>
                            <p><strong>Project:</strong> LMS AuthN and Users Service</p>
                            <p><strong>Environment:</strong> ${params.ENVIRONMENT}</p>
                            <p><strong>Build Number:</strong> ${BUILD_NUMBER}</p>
                            <p><strong>Status:</strong> ❌ FAILURE</p>
                            <br>
                            <p><a href="${BUILD_URL}console">View Console Output</a></p>
                        """,
                        to: "${EMAIL_RECIPIENT}",
                        mimeType: 'text/html'
                    )
                    echo "✅ Failure notification email sent to ${EMAIL_RECIPIENT}"
                } catch (Exception e) {
                    echo "⚠️ WARNING: Could not send failure email notification"
                    echo "Email error: ${e.message}"
                }
            }
        }

        always {
            script {
                echo "Cleaning up old Docker images..."
                sh """
                    docker image prune -f --filter "until=24h" || true
                """
            }
        }
    }
}

