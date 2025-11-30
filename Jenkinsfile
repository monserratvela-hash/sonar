pipeline {
  agent any
  environment {
    WORKDIR = 'trufflehog-output'
    TARGET_DIR = '.'   // directorio a escanear; si clonaste repo, usar workspace root
    TRUFFLE_IMAGE = 'trufflesecurity/trufflehog'
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh "mkdir -p ${WORKDIR}"
      }
    }

    stage('(Opcional) Inyectar secreto de prueba') {
      steps {
        // crear un secreto intencional para probar (elimina en producción)
        sh '''
          mkdir -p trufflehog-test
          cat > trufflehog-test/fake_secret.txt <<'EOF'
          AWS_ACCESS_KEY_ID=AKIAFAKEKEYEXAMPLE
          AWS_SECRET_ACCESS_KEY=FAKESECRET1234567890
          PASSWORD=myfakepassword123
          EOF
        '''
      }
    }

    stage('TruffleHog scan (filesystem)') {
      steps {
        script {
          // Ejecutar trufflehog en modo filesystem sobre workspace
          sh """
            docker run --rm -v \$(pwd):/src ${TRUFFLE_IMAGE} filesystem --directory /src/${TARGET_DIR} --json > ${WORKDIR}/trufflehog_report.json || true
          """
          // Nota: usamos '|| true' para que el step no falle automáticamente; controlamos después con el script de chequeo
        }
      }
      post {
        always {
          archiveArtifacts artifacts: "${WORKDIR}/**", fingerprint: true
        }
      }
    }

    stage('Analizar reporte y aplicar umbrales') {
      steps {
        // crear el script python dinámicamente (si no está en repo)
        writeFile file: 'trufflehog_check.py', text: '''
import json,sys
REPORT = sys.argv[1] if len(sys.argv)>1 else 'trufflehog-output/trufflehog_report.json'
MIN_FINDINGS = int(sys.argv[2]) if len(sys.argv)>2 else 1
try:
    with open(REPORT,'r',encoding='utf-8') as f:
        data = json.load(f)
except Exception as e:
    print("Error leyendo reporte:", e)
    sys.exit(2)
if isinstance(data, dict) and 'matches' in data:
    findings = data['matches']
elif isinstance(data, list):
    findings = data
else:
    findings = []
    if isinstance(data, dict):
        for k in ('results','matches','leaks'):
            if k in data and isinstance(data[k], list):
                findings.extend(data[k])
count = len(findings)
print("Encontradas",count,"coincidencias.")
if count < MIN_FINDINGS:
    print("UMBRAL NO CUMPLIDO: falla el build.")
    sys.exit(1)
print("OK. Umbral cumplido.")
sys.exit(0)
'''
        sh 'python3 -V || true'
        sh 'python3 trufflehog_check.py trufflehog-output/trufflehog_report.json 1'
      }
    }
  }
  post {
    always {
      echo "Pipeline finalizado. Revisar artefactos en la build."
    }
    failure {
      echo "Pipeline marcado como FALLIDO: revisar reporte trufflehog en los artefactos."
    }
  }
}