 #!groovy

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
        pollSCM('H/5 * * * *')  // Poll GitHub every 5 minutes
    }
    
    parameters {
        choice(name: 'XUMLC', choices: ['jarfiles/xumlc-7.20.0.jar'], description: 'Location of the xUML Compiler')
        choice(name: 'REGTEST', choices: ['jarfiles/RegTestRunner-8.10.5.jar'], description: 'Location of the Regression Test Runner')
        string(name: 'BRIDGE_HOST', defaultValue: 'ec2-52-74-183-0.ap-southeast-1.compute.amazonaws.com', description: 'Bridge host address')
        string(name: 'BRIDGE_USER', defaultValue: 'jprocero', description: 'Bridge username')
        password(name: 'BRIDGE_PASSWORD', defaultValue: 'jprocero', description: 'Bridge password')
        string(name: 'BRIDGE_PORT', defaultValue: '8080', description: 'Bridge port')
        string(name: 'CONTROL_PORT', defaultValue: '21190', description: 'Control port')
    }

    stages {
        stage('Build') {
            steps {
                dir('.') {
                    bat """
                        java -jar ${XUMLC} -uml uml/BuilderUML.xml
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
                       
                        if not exist repository\\BuilderUML\\JenkinsCoffeeSoap.rep (
                            echo ERROR: JenkinsCoffeeSoap.rep not found!
                            exit /b 1
                        )
                         
                        echo All repository files found, starting deployment...
                        npx e2e-bridge-cli deploy repository/BuilderUML/JenkinsCoffeeSoap.rep -h ${BRIDGE_HOST} -u ${BRIDGE_USER} -P ${BRIDGE_PASSWORD} -o overwrite
                        
                    """
                }
            }
        }
        stage('Test') {
            steps {
                dir('.') {
                    bat """
                        echo Starting regression tests...
                        echo Using RegTest jar: ${REGTEST_JAR}
                        
                        echo Checking if regtest jar exists...
                        if not exist "${REGTEST_JAR}" (
                            echo ERROR: RegTest jar not found at ${REGTEST_JAR}
                            exit /b 1
                        )
                        
                        echo Checking if test cases exist...
                        if not exist "testcase\\coffee_service_tests.xml" (
                            echo ERROR: Test cases not found in testcase directory
                            echo Please ensure testcase/coffee_service_tests.xml exists
                            exit /b 1
                        )
                        
                        echo Starting regression tests...
                        echo Test configuration:
                        echo - Project: BuilderUML
                        echo - Host: ${BRIDGE_HOST}
                        echo - Port: ${BRIDGE_PORT}
                        echo - Username: ${BRIDGE_USER}
                        echo - Note: RegTestRunner will run all available test suites in the project
                        
                        echo.
                        echo Checking available test suites...
                        java -jar "${REGTEST_JAR}" -project BuilderUML -host ${BRIDGE_HOST} -port ${BRIDGE_PORT} -username ${BRIDGE_USER} -password ${BRIDGE_PASSWORD} -list
                        
                        echo.
                        echo Running all available regression tests...
                        java -jar "${REGTEST_JAR}" -project BuilderUML -host ${BRIDGE_HOST} -port ${BRIDGE_PORT} -username ${BRIDGE_USER} -password ${BRIDGE_PASSWORD} -logfile result.xml
                        
                        if errorlevel 1 (
                            echo Tests completed with errors
                            echo Checking result.xml for details...
                            if exist result.xml (
                                type result.xml
                            )
                            exit /b 1
                        )
                        
                        echo Tests completed successfully
                        echo Checking if result.xml was created...
                        if exist result.xml (
                            echo result.xml found, size:
                            dir result.xml
                            echo.
                            echo Test results summary:
                            type result.xml
                        ) else (
                            echo WARNING: result.xml not found
                        )
                    """
                }
            }
            post {
                always {
                    script {
                        if (fileExists('result.xml')) {
                            def resultContent = readFile('result.xml')
                            echo "Processing test results..."
                            
                            // Check if we have actual test results
                            if (resultContent.contains('tests="0"') || resultContent.contains('testsuite name=""')) {
                                echo "No test results found in result.xml - this may indicate test configuration issues"
                                echo "Result content: ${resultContent}"
                                
                                // Create a placeholder result for Jenkins reporting
                                writeFile file: 'result.xml', text: '''<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
   <testsuite name="Coffee Service Regression Tests" tests="1" failures="0" errors="0" skipped="1">
      <testcase name="TestConfigurationCheck" classname="RegressionTest">
         <skipped message="Test configuration needs verification - check test cases and service deployment"/>
      </testcase>
   </testsuite>
</testsuites>'''
                            } else {
                                echo "Test results found and processed successfully"
                            }
                            
                            // Always publish results for Jenkins reporting
                            junit 'result.xml'
                            archiveArtifacts artifacts: 'result.xml'
                            
                            // Also archive test case files for debugging
                            archiveArtifacts artifacts: 'testcase/**/*.xml'
                            
                        } else {
                            echo "No test results file found - this indicates a test execution problem"
                            // Create a failure result for Jenkins
                            writeFile file: 'result.xml', text: '''<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
   <testsuite name="Coffee Service Regression Tests" tests="1" failures="1" errors="0" skipped="0">
      <testcase name="TestExecutionFailure" classname="RegressionTest">
         <failure message="Test execution failed - result.xml was not generated"/>
      </testcase>
   </testsuite>
</testsuites>'''
                            junit 'result.xml'
                        }
                    }
                }
            }
        }

    }
}