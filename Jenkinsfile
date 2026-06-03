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
                echo "Ejecutando análisis estático de seguridad..."
                // 1. Validación estructural y semántica del template SAM/CloudFormation (Punto 2)
                sh 'cfn-lint template.yaml || true' // Linter de CloudFormation
                sh 'sam validate' // Validación semántica de AWS SAM
                
                // 2. Escaneo de seguridad del código y dependencias (DevSecOps)
                // Nota: Reemplazar con las herramientas reales instaladas en tu Jenkins (ej: Checkov, cfn-nag, Trufflehog)
                echo "Escaneando template de infraestructura buscando vulnerabilidades o datos sensibles expuestos..."
                sh 'checkov -f template.yaml --soft-fail' 
            }
        }

        stage('6. Deploy Backend (AWS SAM)') {
            steps {
                withAWS(credentials: "${AWS_CRED_ID}", region: "${AWS_REGION}") {
                    echo "Iniciando despliegue de infraestructura Serverless en ambiente: ${params.ENVIRONMENT}..."
                    
                    // Despliegue automatizado con estrategia Canary gestionada por el template/CodeDeploy
                    sh """
                        sam deploy \
                        --template-file template.yaml \
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