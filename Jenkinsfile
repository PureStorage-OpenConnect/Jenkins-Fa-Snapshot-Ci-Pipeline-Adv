def RefreshDatabase(RefreshDatabase, SourceInstance, DestInstance, PfaEndpoint) {
    timeout(time:5, unit:'MINUTES') {
        withCredentials([string(credentialsId: 'PfaCredentialsFile', variable: 'PfaCredentialsFile'),
                         string(credentialsId: 'PfaUser'           , variable: 'PfaUser')]) {
            powershell 'Import-Module -Name PureStorageDbaTools; ' + 
                       '\$Pwd = Get-Content ' + "${PfaCredentialsFile}" + ' | ConvertTo-SecureString;'  +
                       '\$Creds = New-Object System.Management.Automation.PSCredential(\"' + "${PfaUser}" + '\",\$Pwd); ' +
                       'Invoke-PfaDbRefresh -RefreshDatabase ' + "${RefreshDatabase}"       + 
                                          ' -RefreshSource   ' + "${SourceInstance}" + 
                                          ' -DestSqlInstance ' + "${DestInstance}"   + 
                                          ' -PfaEndpoint     ' + "${PfaEndpoint}"    + 
                                          ' -PfaCredentials  \$Creds'
        }
    }
}

def GetScmProjectName() {
    def scmProjectName = scm.getUserRemoteConfigs()[0].getUrl().tokenize('/').last().split("\\.")[0]
    return scmProjectName.trim()
}

def RefresfDevDatabase() {
    parallel (
        dev1: {
	    RefreshDatabase("${params.REFRESH_DATABASE}", "${params.IAT_SQL_INSTANCE}", "${params.DEV1_SQL_INSTANCE}", "${PFA_ENDPOINT}")
        },
        dev2: {
	    RefreshDatabase("${params.REFRESH_DATABASE}", "${params.IAT_SQL_INSTANCE}", "${params.DEV2_SQL_INSTANCE}", "${PFA_ENDPOINT}")
        },
        dev3: {
	    RefreshDatabase("${params.REFRESH_DATABASE}", "${params.IAT_SQL_INSTANCE}", "${params.DEV3_SQL_INSTANCE}", "${PFA_ENDPOINT}")
        },
        dev4: {
	    RefreshDatabase("${params.REFRESH_DATABASE}", "${params.IAT_SQL_INSTANCE}", "${params.DEV4_SQL_INSTANCE}", "${PFA_ENDPOINT}")
        }
    )            
}

pipeline {
    agent any

    environment {
        SCM_PROJECT        = GetScmProjectName()
	IAT_CONNECT_STRING = "server=${params.IAT_SQL_INSTANCE};database=${params.REFRESH_DATABASE}"
	SQLPACKAGE_EXE     = "C:\\Program Files (x86)\\Microsoft SQL Server\\140\\DAC\\bin\\sqlpackage.exe"
    }
    
    parameters {
        string(name: 'PFA_ENDPOINT'     , defaultValue: '10.225.112.10'              , description: 'Flash array end point')
        string(name: 'REFRESH_DATABASE' , defaultValue: 'tpch-no-compression'        , description: 'Database that is refresh source/target')
        string(name: 'PROD_SQL_INSTANCE', defaultValue: 'Z-STN-WIN2016-A\\DEVOPSPRD' , description: 'Database that is refresh source/target')
        string(name: 'IAT_SQL_INSTANCE' , defaultValue: 'Z-STN-WIN2016-A\\DEVOPSIAT' , description: 'Database that is refresh source/target')
        string(name: 'DEV1_SQL_INSTANCE', defaultValue: 'Z-STN-WIN2016-A\\DEVOPSDEV1', description: 'Database that is refresh source/target')
        string(name: 'DEV2_SQL_INSTANCE', defaultValue: 'Z-STN-WIN2016-A\\DEVOPSDEV2', description: 'Database that is refresh source/target')
        string(name: 'DEV3_SQL_INSTANCE', defaultValue: 'Z-STN-WIN2016-A\\DEVOPSDEV3', description: 'Database that is refresh source/target')
        string(name: 'DEV4_SQL_INSTANCE', defaultValue: 'Z-STN-WIN2016-A\\DEVOPSDEV4', description: 'Database that is refresh source/target')
        booleanParam(name: 'HAPPY_PATH' , defaultValue: true                         , description: 'Toggle to send tests down happy/unhappy path')
    }
    
    stages {	  
        stage('git checkout') {
	    steps {
                timeout(time:1, unit:'MINUTES') {
                    checkout scm
                }
	    }
        }
	        
        stage('Build Dacpac from SQLProj') {
	    steps {
                timeout(time:5, unit:'MINUTES') {
                    bat "\"${tool name: 'Default', type: 'msbuild'}\" /p:Configuration=Release"
                    stash includes: '"${SCM_PROJECT}"\\bin\\Release\\"${SCM_PROJECT}".dacpac', name: 'theDacpac'
                }
	    }
        }        
        
        stage('refresh IAT from prod') {
	    steps {		   
                RefreshDatabase("${params.REFRESH_DATABASE}", "${params.PROD_SQL_INSTANCE}", "${params.IAT_SQL_INSTANCE}", "${PFA_ENDPOINT}")
	    }
	}
        
        stage('Deploy Dacpac to SQL Server')
        {
	    steps {
                timeout(time:2, unit:'MINUTES') {
                    unstash 'theDacpac'			
		    bat "sqlcmd -S ${params.IAT_SQL_INSTANCE} -Q \"EXEC sp_configure 'show advanced option', '1';RECONFIGURE\""
                    bat "sqlcmd -S ${params.IAT_SQL_INSTANCE} -Q \"EXEC sp_configure 'clr enabled', 1;RECONFIGURE\""
                    bat "sqlcmd -S ${params.IAT_SQL_INSTANCE} -Q \"EXEC sp_configure 'clr strict security', 0;RECONFIGURE\""			
		    bat "\"${SQLPACKAGE_EXE}\" /Action:Publish /SourceFile:\"${SCM_PROJECT}\\bin\\Release\\${SCM_PROJECT}.dacpac\" /TargetConnectionString:\"${IAT_CONNECT_STRING}\""
		}        
	    }
        }        

        stage('run tests (Happy path)') {
            when {
                expression {
                    return params.HAPPY_PATH
                }
            }
            steps {
                bat "sqlcmd -S ${params.IAT_SQL_INSTANCE} -d ${params.REFRESH_DATABASE} -Q \"EXEC tSQLt.Run \'tSQLtHappyPath\'\""
                bat "sqlcmd -S ${params.IAT_SQL_INSTANCE} -d ${params.REFRESH_DATABASE} -y0 -Q \"SET NOCOUNT ON;EXEC tSQLt.XmlResultFormatter\" -o \"${WORKSPACE}\\${SCM_PROJECT}.xml\"" 
                junit "${SCM_PROJECT}.xml"
            }
        }

        stage('run tests (Un-happy path)') {
            when {
                expression {
                    return !(params.HAPPY_PATH)
                }
            }
            steps {
                bat "sqlcmd -S ${params.IAT_SQL_INSTANCE} -d ${params.REFRESH_DATABASE} -Q \"EXEC tSQLt.Run \'tSQLtUnHappyPath\'\""
                bat "sqlcmd -S ${params.IAT_SQL_INSTANCE} -d ${params.REFRESH_DATABASE} -y0 -Q \"SET NOCOUNT ON;EXEC tSQLt.XmlResultFormatter\" -o \"${WORKSPACE}\\${SCM_PROJECT}.xml\"" 
                junit "${SCM_PROJECT}.xml"
            }
        }
	    
	stage('build status') {
	    steps {
		print "${currentBuild.result}"
	    }
	}	    
	    
	stage('BUILD IS STABLE => refresh dev databases') {
            when {
                expression {
		    return (${currentbuild.currentresult} == "SUCCESS")
                }
            }
            steps {
		parallel (
		    dev1: {
			RefreshDatabase("${params.REFRESH_DATABASE}", "${params.IAT_SQL_INSTANCE}", "${params.DEV1_SQL_INSTANCE}", "${PFA_ENDPOINT}")
		    },
		    dev2: {
			RefreshDatabase("${params.REFRESH_DATABASE}", "${params.IAT_SQL_INSTANCE}", "${params.DEV2_SQL_INSTANCE}", "${PFA_ENDPOINT}")
		    },
		    dev3: {
			RefreshDatabase("${params.REFRESH_DATABASE}", "${params.IAT_SQL_INSTANCE}", "${params.DEV3_SQL_INSTANCE}", "${PFA_ENDPOINT}")
		    },
		    dev4: {
			RefreshDatabase("${params.REFRESH_DATABASE}", "${params.IAT_SQL_INSTANCE}", "${params.DEV4_SQL_INSTANCE}", "${PFA_ENDPOINT}")
		    }
		)            
	    }
        }

    }
    post {
        always {
            print 'post: Always'
        }
        success {
            print 'post: Success'
	    RefresfDevDatabase()
        }
        unstable {
            print 'post: Unstable'
        }
        failure {
            print 'post: Failure'
        }
    }
}
