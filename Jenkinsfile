#!groovy
node("slave") {
    stage "Получение исходных кодов"
    //git url: 'https://github.com/silverbulleters/vanessa-behavior-new.git'
    
    
    checkout scm
    if (env.DISPLAY) {
        println env.DISPLAY;
    } else {
        env.DISPLAY=":1"
    }
    //env.RUNNER_ENV="production";

    if (isUnix()) {sh 'git config --system core.longpaths'} else {bat "git config --system core.longpaths"}

    if (isUnix()) {sh 'git submodule update --init'} else {bat "git submodule update --init"}
    
    stage "Контроль технического долга"

    if (env.QASONAR) {
        println env.QASONAR;
        def sonarcommand = "@\"./../../../tools/hudson.plugins.sonar.SonarRunnerInstallation/Main_Classic/bin/sonar-scanner\""
        withCredentials([[$class: 'StringBinding', credentialsId: env.SonarOAuthCredentianalID, variable: 'SonarOAuth']]) {
            sonarcommand = sonarcommand + " -Dsonar.host.url=http://sonar.silverbulleters.org -Dsonar.login=${env.SonarOAuth}"
        }
        
        // Get version
        def configurationText = readFile encoding: 'UTF-8', file: 'vanessa-behavior/VanessaBehavior/Ext/ObjectModule.bsl'
        def configurationVersion = (configurationText =~ /Версия = "(.*)";/)[0][1]
        sonarcommand = sonarcommand + " -Dsonar.projectVersion=${configurationVersion}"

        def makeAnalyzis = true
        if (env.BRANCH_NAME == "develop") {
            echo 'Analysing develop branch'
        } else if (env.BRANCH_NAME.startsWith("release/")) {
            sonarcommand = sonarcommand + " -Dsonar.branch=${BRANCH_NAME}"
        } else if (env.BRANCH_NAME.startsWith("PR-")) {
            // Report PR issues           
            def PRNumber = env.BRANCH_NAME.tokenize("PR-")[0]
            def gitURLcommand = 'git config --local remote.origin.url'
            def gitURL = ""
            if (isUnix()) {
                gitURL = sh(returnStdout: true, script: gitURLcommand).trim() 
            } else {
                gitURL = bat(returnStdout: true, script: gitURLcommand).trim() 
            }
            def repository = gitURL.tokenize("/")[2] + "/" + gitURL.tokenize("/")[3]
            repository = repository.tokenize(".")[0]
            withCredentials([[$class: 'StringBinding', credentialsId: env.GithubOAuthCredentianalID, variable: 'githubOAuth']]) {
                sonarcommand = sonarcommand + " -Dsonar.analysis.mode=issues -Dsonar.github.pullRequest=${PRNumber} -Dsonar.github.repository=${repository} -Dsonar.github.oauth=${env.githubOAuth}"
            }
        } else {
            makeAnalyzis = false
        }
        if (makeAnalyzis) {
            if (isUnix()) {
                sh '${sonarcommand}'
            } else {
                bat "${sonarcommand}"
            }
        }
    } else {
        echo "QA runner not installed"
    }

    stage "Подготовка окружения"

    def srcpath = "./lib/CF/83NoSync";
    if (env.SRCPATH){
        srcpath = env.SRCPATH;
    }
    def v8version = "";
    if (env.V8VERSION) {
        v8version = "--v8version ${env.V8VERSION}"
    }
    def command = "oscript -encoding=utf-8 tools/init.os init-dev ${v8version} --src "+srcpath
    timestamps {
        if (isUnix()){
            sh "${command}"
        } else {
            bat "chcp 1251\n${command}"
        }
    }
    
    stage "Сборка поставки"
	
    echo "build catalogs"
    command = """oscript -encoding=utf-8 tools/runner.os compileepf ${v8version} --ibname /F"./build/ib" ./ ./build/out/ """
    if (isUnix()) {sh "${command}"} else {bat "chcp 1251\n${command}"}       
    
    stage "Проверка поведения BDD"
    def testsettings = "VBParams837UF.json";
    if (env.PATHSETTINGS) {
        testsettings = env.PATHSETTINGS;
    }
    
    // TODO:
    // Придумать, как это сделать красиво и с учетом того, что задано в VBParams837UF.json
    // Стр = Стр + " /Execute " + ПараметрыСборки["EpfДляИнициализацияБазы"] + " /C""InitDataBase;VBParams=" + ПараметрыСборки["ПараметрыДляИнициализацияБазы"] + """";
    def VBParamsPath = pwd().replaceAll("%", "%%") + "/build/out/tools/epf/init.json"
    command = """oscript -encoding=utf-8 tools/runner.os run ${v8version} --ibname /F"./build/ib" --execute "./build/out/tools/epf/init.epf" --command "InitDataBase;VBParams=${VBParamsPath}" """
    def errors = []
    try{
        if (isUnix()){
            sh "${command}"
            
        } else {
            bat "chcp 1251\n${command}"
        }
    } catch (e) {
         errors << "BDD status : ${e}"
    }

    command = """oscript -encoding=utf-8 tools/runner.os vanessa ${v8version} --ibname /F"./build/ib" --path ./build/out/vanessa-behavior.epf --pathsettings ./tools/JSON/${testsettings} """
    try{
        if (isUnix()){
            sh "${command}"
            
        } else {
            env.VANESSA_commandscreenshot='nircmd.exe savescreenshot '
            bat "chcp 1251\n${command}"
        }
    } catch (e) {
         errors << "BDD status : ${e}"
    }

    command = """allure generate ./build/out/allurereport -o ./build/htmlpublish"""
    if (isUnix()){ sh "${command}" } else {bat "chcp 1251\n${command}"}
    publishHTML(target:[allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: './build/htmlpublish', reportFiles: 'index.html', reportName: 'Allure report'])

    if (errors.size() > 0) {
        currentBuild.result = 'UNSTABLE'
        for (int i = 0; i < errors.size(); i++) {
            echo errors[i]; 
        }
    } else {
        step([$class: 'ArtifactArchiver', artifacts: '**/build/out/*.epf', fingerprint: true])
        step([$class: 'ArtifactArchiver', artifacts: '**/build/out/features/Libraries/**/*.epf', fingerprint: true])
        step([$class: 'ArtifactArchiver', artifacts: '**/build/out/features/Libraries/**/*.feature', fingerprint: true])    
    }

    stage "Публикация релизов"

    echo "stable if master, pre-release if have release, nigthbuild if develop"

}