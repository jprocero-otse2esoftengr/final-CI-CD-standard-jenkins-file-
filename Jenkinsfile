 pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '1'))
        disableConcurrentBuilds()
    }

    environment {
        REGTEST_JAR = 'jarfiles/RegTestRunner-8.10.5.jar'
    }

    triggers {
        githubPush()  // Trigger on GitHub webhook
    }

    parameters {
        choice(name: 'XUMLC', choices: ['jarfiles/xumlc-7.20.0.jar'], description: 'Location of the xUML Compiler')
        choice(name: 'REGTEST', choices: ['jarfiles/RegTestRunner-8.10.5.jar'], description: 'Location of the Regression Test Runner')
        string(name: 'BRIDGE_HOST', defaultValue: 'ec2-52-74-183-0.ap-southeast-1.compute.amazonaws.com', description: 'Bridge host address')
        string(name: 'BRIDGE_USER', defaultValue: 'jprocero', description: 'Bridge username')
        password(name: 'BRIDGE_PASSWORD', defaultValue: 'jprocero', description: 'Bridge password')
        string(name: 'BRIDGE_PORT', defaultValue: '11165', description: 'Bridge port')
        string(name: 'CONTROL_PORT', defaultValue: '21179', description: 'Control port for deployment')
    }

    stages {
        stage('Validate Control Port') {
            steps {
                script {
                    if (!params.CONTROL_PORT?.trim()) {
                        error("CONTROL_PORT is missing or empty. Please update MagicDraw or specify the control port.")
                    }
                }
            }
        }
        stage('Build') {
            steps {
                dir('.') {
                    bat """
                        java -jar ${params.XUMLC} -uml uml/BuilderUML.xml
                        if errorlevel 1 exit /b 1
                        echo Build completed successfully
                        dir repository\\BuilderUML\\*.rep
                    """
                    archiveArtifacts artifacts: 'repository/BuilderUML/*.rep'
                }
            }
        }
        stage('Deploy') {
            steps {
                dir('.') {
                    bat """
                        echo Checking for repository files...
                        if not exist repository\\BuilderUML\\regtestlatest.rep (
                            echo ERROR: regtestlatest.rep not found!
                            exit /b 1
                        )
                        echo All repository files found, starting deployment...
                        npx e2e-bridge-cli deploy repository/BuilderUML/regtestlatest.rep -h ${params.BRIDGE_HOST} -u ${params.BRIDGE_USER} -P ${params.BRIDGE_PASSWORD} -o overwrite
                    """
                }
            }
        }
        stage('List Test Suites') {
            steps {
                dir('regressiontest') {
                    bat """
                        echo Listing available test suites...
                        java -jar ${params.REGTEST} -project . -list
                        echo.
                        echo Checking project structure...
                        dir /s testsuite
                        echo.
                        echo Checking if testsuite.xml exists...
                        if exist testsuite\\testsuite.xml (
                            echo testsuite.xml found
                            type testsuite\\testsuite.xml | findstr "testcase"
                        ) else (
                            echo testsuite.xml not found
                        )
                    """
                }
            }
        }
        stage('Test') {
            steps {
                dir('.') {
                    bat """
                        echo Starting regression tests...
                        echo Using RegTest jar: ${params.REGTEST}
                        java -jar "${params.REGTEST}" -project . -host ${params.BRIDGE_HOST} -port ${params.BRIDGE_PORT} -username ${params.BRIDGE_USER} -password ${params.BRIDGE_PASSWORD} -controlport ${params.CONTROL_PORT} -logfile regressiontest/result.xml
                        echo Checking if result.xml was created...
                        if exist regressiontest\\result.xml (
                            echo result.xml found, displaying contents:
                            type regressiontest\\result.xml
                        )
                    """
                }
            }
        }
    }
}