pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Ambiente de despliegue')
    }

    environment {
        AWS_CRED_ID    = 'hgis-credentials-aws'
        AWS_REGION     = 'us-east-1'
        STACK_NAME     = "hgis-backend-${params.ENVIRONMENT}"
    }

    stages {

        stage('4. Static Security Scan (SAST & IaC)') {
            steps {
                dir('./manual') {                              // ← cambia el directorio para todos los sh
                    echo "Ejecutando análisis estático de seguridad..."
                    
                    sh 'cfn-lint template.yaml || true'
                    sh 'sam validate'
                    
                    echo "Escaneando template de infraestructura..."
                    sh 'checkov -f template.yml --soft-fail'
                }
            }
        }

        stage('6. Deploy Backend (AWS SAM)') {
            steps {
                dir('./manual') {                              // ← mismo fix aquí
                    withAWS(credentials: "${AWS_CRED_ID}", region: "${AWS_REGION}") {
                        echo "Desplegando en: ${params.ENVIRONMENT}..."
                        sh """
                            sam deploy \
                            --template-file template.yml \
                            --stack-name ${STACK_NAME} \
                            --resolve-s3 \
                            --parameter-overrides Environment=${params.ENVIRONMENT} \
                            --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
                            --no-confirm-changeset
                        """
                    }
                }
            }
        }

    }

    post {
        success {
            echo "¡Despliegue ejecutado exitosamente en ${params.ENVIRONMENT}!"
            // Aquí podrías agregar alertas a Slack o Teams para cumplir con la observabilidad
        }
        failure {
            echo "El pipeline falló. Iniciando alertas y revisión de logs..."
            // Si el despliegue de SAM falla, AWS CloudFormation ejecuta un Rollback automático del stack (Punto 3)
        }
    }
}