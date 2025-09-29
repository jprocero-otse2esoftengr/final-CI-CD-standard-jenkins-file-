#!groovy

pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '1'))
        disableConcurrentBuilds()
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
        string(name: 'CONTROL_PORT', defaultValue: '21178', description: 'Control port')
    }


     
    stages {
         
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
                        echo Checking if service already exists...
                        npx e2e-bridge-cli status regtestlatest -h ${params.BRIDGE_HOST} -u ${params.BRIDGE_USER} -P ${params.BRIDGE_PASSWORD} 2>nul || echo "Service not found, proceeding with deployment"
                        echo Deploying service...
                        npx e2e-bridge-cli deploy repository/BuilderUML/regtestlatest.rep -h ${params.BRIDGE_HOST} -u ${params.BRIDGE_USER} -P ${params.BRIDGE_PASSWORD} -o overwrite
                        echo Starting the service...
                        npx e2e-bridge-cli start regtestlatest -h ${params.BRIDGE_HOST} -u ${params.BRIDGE_USER} -P ${params.BRIDGE_PASSWORD}
                        
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
                        
                        echo Checking if regtest jar exists...
                        if not exist "${params.REGTEST}" (
                            echo ERROR: RegTest jar not found at ${params.REGTEST}
                            exit /b 1
                        )
                        
                        echo Checking if test cases exist...
                        if not exist "regressiontest\\testsuite\\testsuite.xml" (
                            echo ERROR: Test cases not found in regressiontest directory
                            echo Please ensure regressiontest/testsuite/testsuite.xml exists
                            exit /b 1
                        )
                        
                        echo Starting regression tests...
                        echo Test configuration:
                        echo - Project: .
                        echo - Host: ${params.BRIDGE_HOST}
                        echo - Port: ${params.BRIDGE_PORT}
                        echo - Control Port: ${params.CONTROL_PORT}
                        echo - Username: ${params.BRIDGE_USER}
                        echo - Note: RegTestRunner will run all available test suites in the project
                        
                        echo.
                        echo DEBUG: Checking bridge connection...
                        echo DEBUG: Testing connection to ${params.BRIDGE_HOST}:${params.BRIDGE_PORT}
                        echo DEBUG: Control port will be: ${params.CONTROL_PORT}
                        
                        echo.
                        echo Checking available test suites...
                        java -jar ${params.REGTEST} -project . -host ${params.BRIDGE_HOST} -port ${params.BRIDGE_PORT} -username ${params.BRIDGE_USER} -password ${params.BRIDGE_PASSWORD} -controlport ${params.CONTROL_PORT} -list
                        
                        echo.
                        echo Running all available regression tests...
                        echo Command: java -jar ${params.REGTEST} -project . -host ${params.BRIDGE_HOST} -port ${params.BRIDGE_PORT} -username ${params.BRIDGE_USER} -password ${params.BRIDGE_PASSWORD} -controlport ${params.CONTROL_PORT} -logfile regressiontest/result.xml
                        java -jar ${params.REGTEST} -project . -host ${params.BRIDGE_HOST} -port ${params.BRIDGE_PORT} -username ${params.BRIDGE_USER} -password ${params.BRIDGE_PASSWORD} -controlport ${params.CONTROL_PORT} -logfile regressiontest/result.xml
                        
                        echo.
                        echo ========================================
                        echo REGRESSION TEST RESULTS SUMMARY
                        echo ========================================
                        
                        if exist regressiontest\\result.xml (
                            echo.
                            echo ✓ Test results file created successfully
                            echo.
                            echo DETAILED TEST RESULTS:
                            echo ========================================
                            type regressiontest\\result.xml
                            echo ========================================
                            
                            echo.
                            echo ANALYZING TEST RESULTS...
                            echo ========================================
                            
                            REM Count total tests
                            for /f "tokens=2 delims==\"" %%i in ('findstr "tests=" regressiontest\\result.xml') do (
                                echo Total Tests: %%i
                            )
                            
                            REM Count errors
                            for /f "tokens=2 delims==\"" %%i in ('findstr "errors=" regressiontest\\result.xml') do (
                                echo Errors: %%i
                            )
                            
                            REM Show individual test results
                            echo.
                            echo INDIVIDUAL TEST RESULTS:
                            echo ========================================
                            findstr "testcase" regressiontest\\result.xml
                            
                        ) else (
                            echo.
                            echo ❌ ERROR: result.xml was not created!
                            echo This indicates the RegTestRunner failed to execute tests
                            echo.
                            echo CHECKING FOR ERROR LOGS...
                            if exist regressiontest\\*.log (
                                echo Found log files:
                                dir regressiontest\\*.log
                                echo.
                                echo Latest log content:
                                for %%f in (regressiontest\\*.log) do (
                                    echo ========================================
                                    echo Log file: %%f
                                    echo ========================================
                                    type "%%f"
                                    echo ========================================
                                )
                            ) else (
                                echo No log files found in regressiontest directory
                            )
                        )
                        
                        echo.
                        echo ========================================
                        if errorlevel 1 (
                            echo ❌ TESTS COMPLETED WITH ERRORS - Exit Code: 1
                            echo ========================================
                        ) else (
                            echo ✅ TESTS COMPLETED SUCCESSFULLY - Exit Code: 0
                            echo ========================================
                        )
                    """
                }
            }
            post {
                always {
                    script {
                        if (fileExists('regressiontest/result.xml')) {
                            def resultContent = readFile('regressiontest/result.xml')
                            echo "Processing test results..."
                            
                            // Check if we have actual test results
                            if (resultContent.contains('tests="0"') || resultContent.contains('testsuite name=""')) {
                                echo "No test results found in result.xml - this may indicate test configuration issues"
                                echo "Result content: ${resultContent}"
                                echo "This usually means the RegTestRunner couldn't find or execute any tests"
                            } else {
                                echo "Test results found and processed successfully"
                                echo "Result content: ${resultContent}"
                            }
                            
                            // Always publish results for Jenkins reporting
                            junit 'regressiontest/result.xml'
                            archiveArtifacts artifacts: 'regressiontest/result.xml'
                            
                            // Also archive test case files for debugging
                            archiveArtifacts artifacts: 'regressiontest/.$output/**/*'
                            
                        } else {
                            echo "No test results file found - this indicates a test execution problem"
                            // Create a failure result for Jenkins
                            writeFile file: 'regressiontest/result.xml', text: '''<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
   <testsuite name="BuilderUML Regression Tests" tests="1" failures="1" errors="0" skipped="0">
      <testcase name="TestExecutionFailure" classname="RegressionTest">
         <failure message="Test execution failed - result.xml was not generated"/>
      </testcase>
   </testsuites>
</testsuites>'''
                            junit 'regressiontest/result.xml'
                        }
                    }
                }
            }
        }
    }
}