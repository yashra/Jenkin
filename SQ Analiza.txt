pipeline {
    agent {
    node {
        label 'slave-0'
    }
	}
	environment {
		mavenHome=tool "Maven3"
		scannerHome = tool 'Sonarqube'
		NODEJS_HOME = "${tool 'NodeJS LTS'}"
		PATH="${env.NODEJS_HOME}/bin:${env.PATH}"
		nexusConfiguration = """\\
    <server>\\
        <id>nexus</id> \\
        <username>nexus-demo</username>\\
        <password>nexus-demo</password>\\
    </server>"""
	}	
    stages {
        stage('Clone repository') { 
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/mwocka/wypozyczalnia.git']]])
            }
        }
        stage('Build using npm') { 
            steps {
                sh "${mavenHome}/bin/mvn --version"
				sh "npm --version"
				sh "npm i @angular/cli"
				sh "ls -la front-end"
            }
        }
        stage('SonarQube analysis') { 
            steps {
				withSonarQubeEnv('Sonarqube') {
				sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=sleroy:sample-project -Dsonar.projectName=SampleProject -Dsonar.projectVersion=1.0 -Dsonar.sources=front-end -Dsonar.sourceEncoding=UTF-8"
				} 
            }
        }
		stage('Clone sec repository') { 
            steps {
				sh "rm -fr *"
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/mwocka/simple-java-maven-app.git']]])
            }
        }
		stage('Deploy artifactory') { 
            steps {
				sh """ grep "<id>nexus</id>" ${mavenHome}/conf/settings.xml \\
					&& grep "<username>nexus-demo</username>" ${mavenHome}/conf/settings.xml \\
					&& grep "<password>nexus-demo</password>" \\
					${mavenHome}/conf/settings.xml &&
					echo "Nexus configuration has been already set" ||
					sed -i '/<\\/servers>/i ${nexusConfiguration}' ${mavenHome}/conf/settings.xml"""
				sh "${mavenHome}/bin/mvn clean install -Dmaven.test.skip=true "
				sh "${mavenHome}/bin/mvn deploy -Dmaven.test.skip=true "
            }
        }
    }
}
