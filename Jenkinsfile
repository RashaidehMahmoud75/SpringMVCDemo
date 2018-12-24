// change the project version (don't rely on version from pom.xml)
env.BN = VersionNumber([
        versionNumberString : '${BUILD_MONTH}.${BUILDS_TODAY}.${BUILD_NUMBER}', 
        projectStartDate : '2018-12-24', 
        versionPrefix : 'v1.'
    ])
		
node ("master") {                  
    stage('Provision') {                
        echo 'PIPELINE STARTED'
        
        echo 'Checkout source code from GitHub ...'
        retry(5){
            git 'https://github.com/RashaidehMahmoud75/SpringMVCDemo'
		
        }
        
        echo 'Change the project version ...'
        def W_M2_HOME = tool 'Maven'
        sh "${W_M2_HOME}/bin/mvn versions:set -DnewVersion=$BN -DgenerateBackupPoms=false"                
        
        echo "Create a new branch with name release_${BN} ..."
        def W_GIT_HOME = tool 'Git'
        sh "${W_GIT_HOME} checkout -b release_${BN}"
        
        echo 'Stash the project source code ...'
        stash includes: '**', excludes: '**/TestPlan.jmx', name: 'SOURCE_CODE'
    }
}
 node ("TestMachine-ut") {
        // we can also use: withEnv(['M2_HOME=/usr/share/maven', 'JAVA_HOME=/usr']) {}
        env.MAVEN_HOME = '/usr/share/maven'
        env.M2_HOME = '/usr/share/maven'
        env.JAVA_HOME = '/usr'	 
      
        echo 'Preparing Artifactory to resolve dependencies ...'          
        def server = Artifactory.server('artifactory')       
        def rtMaven = Artifactory.newMavenBuild()
        rtMaven.opts = '-Xms1024m -Xmx4096m'
        rtMaven.resolver server: server, releaseRepo: 'virtual-repo', snapshotRepo: 'virtual-repo'
        
        stage('Run-ut') {   
            try{		
                echo 'Unstash the project source code ...'
                unstash 'SOURCE_CODE'	                                                       
                                
                echo 'Run the unit tests (and Jacoco) ...'
                // sh "'${M2_HOME}/bin/mvn' clean test-compile jacoco:prepare-agent test -Djacoco.destFile=target/jacoco.exec"   
                rtMaven.run pom: 'pom.xml', goals: 'clean test-compile jacoco:prepare-agent test -Djacoco.destFile=target/jacoco.exec'

                echo 'Run the Jacoco code coverage report for unit tests ...'
                step([$class: 'JacocoPublisher', canComputeNew: false, defaultEncoding: '', healthy: '', 
                        pattern: '**/target/jacoco.exec', unHealthy: ''])
			
                echo 'Stash Jacoco-ut exec ...'
                stash includes: '**/target/jacoco.exec', name: 'JACOCO_UT' 
            
                echo 'jUnit report (surefire) ...'
                junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                currentBuild.result='SUCCESS'
            }catch (any){
                currentBuild.result='FAILURE'
                step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: 'sample@org.com', sendToIndividuals: false])
                throw any
            } finally {                
                // ...
            }
        }
    }

