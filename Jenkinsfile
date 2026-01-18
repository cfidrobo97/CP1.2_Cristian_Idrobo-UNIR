pipeline {
  agent any

  stages {

    stage('Get Code') {
      steps {
        //Descargar el repositorio
        git branch: 'master',
          credentialsId: 'github-pat',
          url: 'https://github.com/cfidrobo97/CP1.2_Cristian_Idrobo-UNIR.git'
      }
    }

    stage('Unit') {
      steps {
        // Ejecución unica de Unit
        catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
          bat '''
            del /q .coverage 2>NUL
            set PYTHONPATH=%WORKSPACE%

            coverage run --branch -m pytest --junitxml=result-unit.xml test\\unit
          '''
        }
        //Publicar los resultados
        junit 'result-unit.xml'

        // Guardamos .coverage para usarlo en Coverage sin re-ejecutar tests
        stash name: 'unit-artifacts', includes: '.coverage, result-unit.xml'
      }
    }

    stage('Rest') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
          bat '''
            set FLASK_APP=app\\api.py
            set FLASK_ENV=development
    
            start /B flask run
            start /B java -jar C:\\WIREMOCK\\wiremock-standalone-3.13.2.jar --port 9090 --root-dir test\\wiremock
          '''
          sleep time: 3, unit: 'SECONDS'
    
          bat '''
            set PYTHONPATH=%WORKSPACE%
            pytest --junitxml=result-rest.xml test\\rest
          '''
        }
        //Publicar los resultados
        junit 'result-rest.xml'
      }
    }

    stage('Quality') {
      parallel {

        stage('Static') {
          steps {
            script {
              // Flake8: el estado se decide por # hallazgos
              bat 'flake8 app --format=default > flake8.txt || exit 0'

              recordIssues(
                tools: [flake8(pattern: 'flake8.txt')],
                name: 'Flake8 Analysis'
              )

              def flake8Out = readFile('flake8.txt').trim()
              def count = flake8Out ? flake8Out.readLines().count { it?.trim() } : 0

              echo "Total de hallazgos detectados (Flake8): ${count}"

              if (count >= 10) {
                currentBuild.result = 'FAILURE'
                echo "Estado: Unhealthy (Rojo) por alcanzar ${count} errores."
              } else if (count >= 8) {
                if (currentBuild.result != 'FAILURE') currentBuild.result = 'UNSTABLE'
                echo "Estado: Unstable (Amarillo) por alcanzar ${count} errores."
              } else {
                echo "Estado: OK (Verde)"
              }
            }
          }
        }

        stage('Security Test') {
          steps {
            script {
              // Bandit: el estado se decide por # hallazgos
              bat 'bandit -r app -f json -o bandit.json || exit 0'
              bat 'bandit -r app -f txt  -o bandit_results.txt || exit 0'

              recordIssues(
                tools: [issues(pattern: 'bandit.json', name: 'Bandit')],
                id: 'bandit-security'
              )

              archiveArtifacts artifacts: 'bandit.json, bandit_results.txt', fingerprint: true

              // Conteo de issues desde el reporte TXT
              def out = readFile('bandit_results.txt')
              def count = out.readLines().count { it.contains('>> Issue:') }

              echo "Total de hallazgos de seguridad (Bandit): ${count}"

              if (count >= 4) {
                currentBuild.result = 'FAILURE'
                echo "Estado: Unhealthy (Rojo) por ${count} vulnerabilidades."
              } else if (count >= 2) {
                if (currentBuild.result != 'FAILURE') currentBuild.result = 'UNSTABLE'
                echo "Estado: Unstable (Amarillo) por ${count} vulnerabilidades."
              } else {
                echo "Estado: OK (Verde)"
              }
            }
          }
        }

      }
    }

    stage('Performance') {
      steps {
        script {
          // Performance: levanta Flask y ejecuta JMeter con el archivo cp12.jmx
          bat '''
            if exist jmeter-report rmdir /s /q jmeter-report
            if exist jmeter-results.jtl del /q jmeter-results.jtl 
         
            set FLASK_APP=app\\api.py
            set FLASK_ENV=development
            start /B flask run
          '''
    

          sleep time: 5, unit: 'SECONDS'
    
          // Ejecutamos JMeter en modo no-GUI
          bat '''
            "C:\\jmeter-apache\\apache-jmeter-5.6.3\\bin\\jmeter.bat" ^
              -n ^
              -t test\\jmeter\\cp12.jmx ^
              -l jmeter-results.jtl ^
              -e -o jmeter-report
          '''
    
          // Evidencia + reporte en Jenkins (plugin performance)
          archiveArtifacts artifacts: 'jmeter-results.jtl, jmeter-report/**', fingerprint: true
          perfReport 'jmeter-results.jtl'
        }
      }
    }

    stage('Coverage') {
      steps {
        // Usa el .coverage sin re-ejecutar unit tests
        unstash 'unit-artifacts'

        script {
          // Genera reportes desde el .coverage ya creado
          bat '''
            coverage json -o coverage.json
            coverage xml -o coverage.xml
          '''
          recordCoverage(
            tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
            id: 'coverage',
            name: 'Coverage Report'
          )

          archiveArtifacts artifacts: 'coverage.json, coverage.xml', fingerprint: true

          def cov = readJSON file: 'coverage.json'

          // Líneas
          def linePct = cov.totals.percent_covered

          // Ramas
          def branchesTotal = cov.totals.num_branches ?: 0
          def branchesCovered = cov.totals.covered_branches ?: 0
          def branchPct = (branchesTotal > 0) ? (branchesCovered * 100.0 / branchesTotal) : 100.0

          echo String.format("Coverage líneas: %.2f%% | Coverage ramas: %.2f%%", linePct, branchPct)

          //Reglas Baremo
          def lineState = (linePct < 85) ? 'RED' : (linePct < 95) ? 'YELLOW' : 'GREEN'
          def branchState = (branchPct < 80) ? 'RED' : (branchPct < 90) ? 'YELLOW' : 'GREEN'

          if (lineState == 'RED' || branchState == 'RED') {
            currentBuild.result = 'FAILURE'
            echo "Estado Coverage: Unhealthy (Rojo)"
          } else if (lineState == 'YELLOW' || branchState == 'YELLOW') {
            if (currentBuild.result != 'FAILURE') currentBuild.result = 'UNSTABLE'
            echo "Estado Coverage: Unstable (Amarillo)"
          } else {
            echo "Estado Coverage: OK (Verde)"
          }
        }
      }
    }
    
  }
}
