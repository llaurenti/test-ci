pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    stages {
        stage('Init') {
            steps {
                init()
            }
        }
        stage('Checkout') {
            steps {
                echo "==>> BRANCH_NAME: ${branchName}"
                git branch: "${branchName}", credentialsId: "${CREDENTIAL_ID}", url: 'https://github.com/llaurenti/test-ci'
            }
        }
        stage('Login Aplicacao API') {
            steps {
                loginAplicacao()
            }
        }
        stage('Obtem nova versão') {
            steps {
                getVersaoNova()
            }
        }
        stage('Define Versao') {
            steps {
                defineVersao()
            }
        }
        stage('Build war') {
            steps {
                buildWar()
            }
        }
        stage('Create Properties') {
            steps {
                createProperties()
            }
        }
        stage('Upload Properties') {
            steps {
                uploadProperties()
            }
        }
        stage('deploy war') {
            steps {
                deploy()
            }
        }
        stage('restart server') {
            steps {
                restartServer()
            }
        }
    }
}

def branchName
def CREDENTIAL_ID
def SERVER
def BUCKET_NAME
def tokenAplicacaoApi
def versaoNova
def versaoAtual
def sistema

def init() {
    echo ".:: Iniciando o build da API para QA ::."
    CREDENTIAL_ID = env.GITHUB_CREDENTIALS
    SERVER = "server_qa"
    sistema = 1
    if (APLICACAO == "") {
        APLICACAO = "${DATABASE_NAME.toLowerCase()}-api"
        BUCKET_NAME = DATABASE_NAME.toLowerCase().capitalize()
        BUCKET_NAME = DATABASE_NAME.toLowerCase().capitalize()
    } else {
        APLICACAO = "qualidade-api"
        BUCKET_NAME = "Qualidade"
    }
    def temp = "${BRANCH_NAME}"
    branchName = temp.replace('refs/heads/', '')

    echo """.:: Iniciando o build para Teste de Qualidade ::.k
==>>
==>> CREDENTIAL_ID....: ${CREDENTIAL_ID}
==>> BRANCH_NAME......: ${branchName}
==>> DATABASE_NAME....: ${DATABASE_NAME}
==>> APLICACAO........: ${APLICACAO}
==>> BUCKET_NAME......: ${BUCKET_NAME}
==>> SERVER_ADDRESS...: ${SERVER}
==>> """
}


def buildWar() {
    sh("mvn -B -P jenkins-build,doc-api clean package -DskipTests")
    sleep 1
}

def deploy() {
    def tomcatServer = "http://192.168.1.31:8080"
    sh("curl -v -u unosol:segreds -T ./application/target/application*.war \"${tomcatServer}/manager/text/deploy?path=/${APLICACAO}&update=true\"")
}

def restartServer() {
    sendCommand(SERVER, ["sudo service tomcat stop", "sudo service tomcat start"])
}

def stopServer() {
    sendCommand(SERVER, ["sudo service tomcat stop"])
}

def startServer() {
    sendCommand(SERVER, ["sudo service tomcat start"])
}

def sendCommand(serverAddress, commands) {
    def theTransfer = []
    commands.each {
        theTransfer << sshTransfer(
                execCommand: "${it}",
                execTimeout: 120000
        )
    }
    sshPublisher(
            publishers: [
                    sshPublisherDesc(
                            configName: "${serverAddress}",
                            transfers: theTransfer,
                            usePromotionTimestamp: false,
                            useWorkspaceInPromotion: false,
                            verbose: true
                    )
            ]
    )
}

def createProperties() {
    def databaseName = DATABASE_NAME.toLowerCase()

    sh('rm -rf *.yml')

    def application = """spring:
  datasource:
    url: jdbc:mysql://localhost/db_qa_${databaseName}?useTimezone=true&serverTimezone=GMT-3&useSSL=false
    username: unosol
    password: '#prudencia5!'
logging:
  level:
    org.springframework.jdbc.core.JdbcTemplate: trace
    org.springframework.jdbc.core.StatementCreatorUtils: trace
    br.com.unosolucoes.unoerp.*: debug
boi:
  endpoint: 'http://localhost:8080/${databaseName}-boi/api/boi'
uno:
  reset-secret: false
  security:
    jwt:
      access-expiration: 3650d
      refresh-expiration: 3651d"""

    sh("""echo "${application}" > application.yml""")
}

def uploadProperties() {
    def webappsPath = "webapps/"
    def pathFiles = "/var/tomcat/webapps/UCOMMERCE/${BUCKET_NAME}/properties/*.yml"
    def bucket = "/var/tomcat/webapps/UCOMMERCE/${BUCKET_NAME}"
    sendFile(SERVER,
            "${webappsPath}UCOMMERCE/${BUCKET_NAME}/properties/",
            "*.yml",
            "rm -f ${pathFiles}",
            "find ${bucket} -type d -exec chmod 2770 {} \\;;find ${bucket} -type f -exec chmod 660 {} \\;"
    )
}

def sendFile(serverAddress, remoteDirectory, sourceFiles, commandBefore, comandAfter) {
    def theTransfer = []
    theTransfer << sshTransfer(
            execCommand: "${commandBefore}",
            execTimeout: 120000
    )
    theTransfer << sshTransfer(
            cleanRemote: false,
            execCommand: "${comandAfter}",
            execTimeout: 120000,
            flatten: false,
            makeEmptyDirs: true,
            noDefaultExcludes: false,
            patternSeparator: '[, ]+',
            remoteDirectory: "${remoteDirectory}",
            remoteDirectorySDF: false,
            sourceFiles: "${sourceFiles}"
    )
    sshPublisher(
            publishers: [
                    sshPublisherDesc(
                            configName: "${serverAddress}",
                            transfers: theTransfer,
                            usePromotionTimestamp: false,
                            useWorkspaceInPromotion: false,
                            verbose: true
                    )
            ]
    )
}

def defineVersao() {
    echo "==>> versionando o sistema para a versão ${versaoNova}-B${currentBuild.number}"
    sh("mvn versions:set -DnewVersion=${versaoNova}-B${currentBuild.number}")
}

def loginAplicacao() {
    def body = "key=4721693F0037DEBA91B40252B9F52248&secret=qzy5tFaQkh&grant_type=client_credentials"
    def response =  httpRequest(
            consoleLogResponseBody: false,
            contentType: 'APPLICATION_FORM',
            httpMode: 'POST',
            requestBody: body,
            url: 'http://sorocaba.unoerp.com.br:8080/aplicacao-api/auth/token',
            wrapAsMultipart: false
    )
    if (response.status == 200) {
        def jsonSlurper = new groovy.json.JsonSlurper()
        def object = jsonSlurper.parseText(response.content)
        tokenAplicacaoApi = object.access_token
    }
}

def getVersaoNova() {
    def response =  httpRequest(
            consoleLogResponseBody: false,
            contentType: 'APPLICATION_JSON',
            httpMode: 'GET',
            customHeaders: [[name: "Authorization", value: "Bearer ${tokenAplicacaoApi}"]],
            url: "http://sorocaba.unoerp.com.br:8080/aplicacao-api/sistema/${sistema}/versao-nova",
            wrapAsMultipart: false
    )
    if (response.status == 200) {
        def jsonSlurper = new groovy.json.JsonSlurper()
        def object = jsonSlurper.parseText(response.content)
        versaoNova = object.versao
    }
}

def getVersaoAtual() {
    def response =  httpRequest(
            consoleLogResponseBody: false,
            contentType: 'APPLICATION_JSON',
            httpMode: 'GET',
            customHeaders: [[name: "Authorization", value: "Bearer ${tokenAplicacaoApi}"]],
            url: "http://sorocaba.unoerp.com.br:8080/aplicacao-api/sistema/${sistema}/versao-atual",
            wrapAsMultipart: false
    )
    versaoAtual = "0.0.0";
    if (response.status == 200) {
        def jsonSlurper = new groovy.json.JsonSlurper()
        def object = jsonSlurper.parseText(response.content)
        versaoNova = object.versao
    }
}

def gravaVersao() {
    def jsonBody = new groovy.json.JsonBuilder(
            versao: versaoNova,
            criticidade: 4,
            descricao : "Nononon on o no no no no n on o n on o no n on onoononon onon o no non",
            alert : "Versão está disponível para atualização",
            revisao : currentBuild.number
    ).toPrettyString()
    def response =  httpRequest(
            consoleLogResponseBody: false,
            contentType: 'APPLICATION_JSON',
            httpMode: 'GET',
            customHeaders: [[name: "Authorization", value: "Bearer ${tokenAplicacaoApi}"]],
            url: "http://sorocaba.unoerp.com.br:8080/aplicacao-api/sistema/${sistema}/versao-atual",
            wrapAsMultipart: false
    )
    echo "====>>>> Dados da nova versão enviados status: ${response.status}"
}