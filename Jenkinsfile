pipeline {
        agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: hybris
    image: signet/hybris-ant:6.6.0.15_3.0.3-UAT
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

    stages {
        
	stage('Build') {
            steps {
			
		container('hybris') {
			
                    sh '''
                        #!/bin/bash
                        java -version
                        pwd
                        
                        mkdir -p /hybris-commerce-suite/hybris/bin/custom/training/trainingstorefront/
			
                        cp -R /$WORKSPACE/bin/custom/training/trainingstorefront/ /hybris-commerce-suite/hybris/bin/custom/training/trainingstorefront/
			cd /hybris-commerce-suite/hybris/bin/custom/training/trainingstorefront/
			ls
                        cd /hybris-commerce-suite/hybris/bin/platform 
                        . ./setantenv.sh
                        ant clean -Dinput.template=develop
			ant customize -Dinput.template=develop
			ant build -Dinput.template=develop
			ant production -Dproduction.legacy.mode=true -Dproduction.include.tomcat=true
                    '''
                 } 
            }   
        }
		
    }
}
