pipeline {
        agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: hybris
    image: signet/hybris-ant:1811_21-V8
    command: 
    - /bin/bash
    tty: true	
  - name: maven
    image: maven:3.6.1-slim
    command: 
    - cat
    tty: true
'''
            label 'sample-java-app'
            idleMinutes 10
            defaultContainer 'jnlp'
        }
    }  

	environment{
		scannerHome = tool 'Sonarqube'
		CCV2CMD="/app/sap_cli/bin"
		ccv2_env="d1"
		ccv2_temp_branch="ccv2_deploy_${ccv2_env}"
	}
    	stages {
        
		stage('Build') {
            		steps {
			
				container('hybris') {
					script{
						propfile = readProperties(file: './devops.properties')
					}
			
                    			sh '''
                        			#!/bin/bash
						export JAVA_HOME=/app/sapjvm8/sapjvm_8/
                        			java -version
                        			mkdir -p /hybris-commerce-suite/hybris/bin/custom/training/trainingstorefront/
                        			cp -R /$WORKSPACE/bin/custom/training/trainingstorefront/ /hybris-commerce-suite/hybris/bin/custom/training/trainingstorefront/
						cd /hybris-commerce-suite/hybris/bin/custom/training/trainingstorefront/
						ls
                        			cd /hybris-commerce-suite/hybris/bin/platform 
                        			. ./setantenv.sh
						java -version
						#ant addonuninstall -Daddonnames=assistedservicestorefront,smarteditaddon,captchaaddon,profiletagaddon -DaddonStorefront.yacceleratorstorefront=signetstorefront
						#ant addoninstall -Daddonnames=assistedservicestorefront,smarteditaddon,captchaaddon,profiletagaddon -DaddonStorefront.yacceleratorstorefront=signetstorefront
                        			ant customize clean build -Dinput.template=develop
						ant production -Dproduction.legacy.mode=true -Dproduction.include.tomcat=true
            		
		                    	'''
		    			echo "Performing npm build..."
		    			sh '''
						cd /$WORKSPACE/bin/custom/training/trainingstorefront/web/
						pwd
                   				npm install -g grunt-cli
						npm install grunt --save-dev
						npm install grunt-contrib-less --save-dev
						npm install grunt-contrib-watch --save-dev
						npm install grunt-sync --save-dev
						grunt
		    		
		    			'''	   
                 		} 
            		}   
        	}
		
		stage('Unit Test') {
            		steps {
				container('hybris') {

                    			sh '''
		    	
						export JAVA_HOME=/app/sapjvm8/sapjvm_8/
			
                        			cd /hybris-commerce-suite/hybris/bin/platform 
                        			. ./setantenv.sh
                        			ant unittests
			
					'''
			
					jacoco( 
			      			execPattern: '**/*.exec',
			      			classPattern: '**/*.class',
			      			sourcePattern: '**/*.java',
			      			exclusionPattern: '**/test*'
					)
			
                 		} 
            		}   
        	}
		
		stage('Code Quality') {
            		steps {
				container('maven') {
                			withSonarQubeEnv(installationName:'Sonarqube') {
                    				sh ''' $scannerHome/bin/sonar-scanner -X -Dsonar.projectName=hybris_${BRANCH_NAME} -Dsonar.projectKey=hybris_sample -Dsonar.projectVersion=1.0 -Dsonar.qualityGate=Hybris_Sonar -Dsonar.extensions=trainingstorefront-Dsonar.host.url='https://sonarqube.sgnt.devops.accentureanalytics.com/' -Dsonar.exclusions=file:**/gensrc/**,**/*demo.html,web/webroot/**/web.xml,web/webroot/WEB-INF/config/**/*,web/webroot/WEB-INF/lib/**/*,web/webroot/WEB-INF/views/welcome.jsp,web/webroot/index.jsp,**/*BeforeViewHandler*.java,web/webroot/static/bootstrap/js/*.js,web/webroot/static/theme/js/*.js,web/webroot/signetsmarteditmodule/js/*.js,**/*Constants.java,**/jalo/**,**/email/context/**,**/*Form*.java,web/src/**,**/platform/**,src/com/hybris/yprofile/**,resources/apache-nutch-1.16-custom-code/apache-nutch-1.16/**,**/*.java
                    				'''
                			}
                			timeout(time: 10, unit: 'MINUTES') {
						//sleep(60)
                				waitForQualityGate abortPipeline: true
                			}
				}
            		}
        	}
	
		stage('Create Temp Branch') {
		
			steps {
				echo "Create temp branch for cloud"
			}
		
		}
	    
		stage('Deploy') {
			when { expression {env.GIT_BRANCH == 'dev' || env.GIT_BRANCH == 'release'|| propfile['feature_deploy'] == "true" }}
            		steps {
				container('hybris') {
					
					echo "I am executing Deploy to target environment."
					sh '''	
					cd $CCV2CMD
					export JAVA_HOME=/app/sapmachine-jdk-11.0.10/
					./sapccm --help
					'''
					
					/*
					  	sapccm config set auth-credentials {TOKEN_VALUE}
					  	echo "Create a Build"
					  	sapccm build create –application-code=commerce-cloud  --branch=BRANCH_NAME –name=BUILD_NAME –no-wait –subscription-code= SUBSCRIPTION_CODE
					  
					  	echo "Check a Build"
					  	sapccm  build show –subscription-code=SUBSCRIPTION_CODE  --build-code=BUILD_CODE
					  	echo "Create a Deployment"
					  	sapccm deployment create –build-code=BUILD_CODE  --subscription-code=SUBSCRIPTION_CODE  --database-update-mode=NONE/UPDATE/INITIALIZE  --environment-code=d1/s1/p1  --strategy=CREATE/ROLLING_UPDATE  --subscriptioni-code=SUBSCRIPTION_CODE
					  	echo "Check Deployment" 
					  	sapccm deployment list –subscription-code=SUBSCRIPTION_CODE 
					*/
					
				}
            		}
        	}

		stage('Post Deploy Tests') {
			when { expression {env.GIT_BRANCH == 'dev' || env.GIT_BRANCH == 'release'|| propfile['feature_deploy'] == "true" }}
			parallel {
				stage('Smoke Test') {
					steps {
					
						script {
							
							if (propfile['smoke_test'] == "true") {
								echo "I am executing Smoke Test on target dev environment post deployment"
				
							/*RESP=`curl -X GET "${bamboo.uri}/RequestsRunning" -H "accept: application/xml" -H "authorization: bearer lR0AA2qfq7v9Ry96vDAgqcer1GPVd5yStmv1_aJVFS43rk06EytB7WsS0_owoiXIgpOXmZVEfkY4ST0JwHtRBk7RH0QRaldWtQT8udC0VdimdGx38RddY2sGaeeF0t9Aflr5rh1Jc_EUfkNK8YrKVxQ6kxB05aCe46CD2fkognv7TiJATmht-ycUjEsd_oy8jH5EK9fmn9eL-wXavNTQcEdsUmFm3-2r3IJDzMK7XCa74qu353yOKLvVyZ1yYQBnc1_fY5GS1BDrFLUZprxpAS30lGEu-d_JTTOQ989UJtIEB3cZzDkIQzeqdYBGCsiDdjdHo2DC1FK2kVPyBITTbQ"`
							echo "The response for current execution status is: $RESP"
							if [ "$RESP" != "[]" ];
							then
							echo "There is a test executing currently in Worksoft. Hence, not proceeding with the execution of Worksoft test cases."
							exit 1
							else
							echo "There are no tests executing right now. Hence, proceeding with Worksoft test execution"
							fi
							# To abort the request before attempting to re-run, uncomment and run below line.
							# abort=$(curl -X PUT -H "Authorization: Bearer $token" -d "" -H "id: ${bamboo.RequestID}" ${bamboo.uri}/Request/${bamboo.RequestID}/abort/)
							guid=$(curl -X PUT -H "Authorization: Bearer $token" -d "" -H "parameters: {TestEnv}{${bamboo.stage_name}}" -H "id: ${bamboo.RequestID}" ${bamboo.uri}/ExecuteRequest/ | tr -d \")
							echo "The GUID is: $guid"
							status=$(curl -X GET -H "Authorization: Bearer $token" -d "" -H "APIRequestID: $guid" ${bamboo.uri}/ExecutionStatus/ | awk -F':' '{print $2}' | tr -d \" | tr -d \})
							echo "The status is: $status"
							while [[ $status != *"Completed"* ]]
							do
							status=$(curl -X GET -H "Authorization: Bearer $token" -d "" -H "APIRequestID: $guid" ${bamboo.uri}/ExecutionStatus/ | awk -F':' '{print $2}' | tr -d \" | tr -d \})
							echo "The status is: $status"
							sleep 15
							done	
							status=$(curl -X GET -H "Authorization: Bearer $token" -d "" -H "APIRequestID: $guid" ${bamboo.uri}/ExecutionStatus/)
							echo "The status is: $status"
							execstatus=$(curl -X GET -H "Authorization: Bearer $token" -d "" -H "APIRequestID: $guid" ${bamboo.uri}/ExecutionStatus/ | awk -F':' '{print $3}' | tr -d \" | tr -d \})
							echo "The exec status is: $execstatus"
							if [[ $execstatus != *Passed* ]];
							then
							echo "Failed"
							exit 1
							else
							echo "Passed"
							exit
							fi
							exit */
						}
					}
				}
				}
				stage('Security Test') {
					steps {
						echo 'I am running Security Test here'
					}
				}
			}
		}    
  	}
  	post {
	  
		always {
			script {
				if (propfile['javadoc'] == "true") {
					javadoc(javadocDir: "/$WORKSPACE", keepAll: true)
        			}
		  	}
	  	}
	  
        	failure {
            	/*mail bcc: '', 
	            	 body: "<b>Example</b><br>\n<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", 
	            	 cc: '', 
        	    	 charset: 'UTF-8', 
            		 from: '', 
            	 	mimeType: 'text/html', 
            	 	replyTo: '', 
            	 	subject: "ERROR CI: Project name -> ${env.JOB_NAME}", 
            	 	to: "foo@foomail.com";*/
			echo 'I am sending a notification with failure'
        	}
		success {
			echo 'I am sending a notification with success'
		}
   	}
}
