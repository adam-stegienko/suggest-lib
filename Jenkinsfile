pipeline {
    agent any
    tools {
        maven 'Maven 3.6.2'
    }
    environment {
        GITLAB = credentials('a31843c7-9aa6-4723-95ff-87a1feb934a1')
    }
    stages {
        stage('Setup parameters') {
            steps {
                script {
                    properties([
                        disableConcurrentBuilds(), 
                        gitLabConnection(gitLabConnection: 'GitLab API Connection', jobCredentialId: ''), 
                        [$class: 'GitlabLogoProperty', repositoryName: 'adam/suggest-lib'], 
                        parameters([
                            validatingString(
                                description: 'Put "release" and a 2-digit value meaning the branch you would like to build your version on, as in the following examples: "release/1.0", "release/1.1", "release/1.2", etc. For either master or feature branch to choose, type "feature" or "master".', 
                                failedValidationMessage: 'Parameter format is not valid. Try again with valid parameter format.', 
                                name: 'Version', 
                                regex: '^master$|^feature$|release\\/[0-9]{1,}\\.[0-9]{1,}$'
                            )
                        ]), 
                    ])
                }
            }
        }
        stage('Release branch versioning') {
            when { branch "release/*" }
            steps{
                script {
                    BRANCH = params.Version
                    echo "This is the branch you chose to operate on: $BRANCH"
                    sh"""
                        git checkout master
                        git remote set-url origin http://\"$GITLAB\"@ec2-3-125-51-254.eu-central-1.compute.amazonaws.com/adam/suggest-lib.git
                        git pull --rebase
                    """
                    BRANCH_EXISTING = sh(
                        script: "(git ls-remote -q | grep -w $BRANCH) || BRANCH_EXISTING=False",
                        returnStdout: true,
                    )
                    if (BRANCH_EXISTING) {
                        echo "The $BRANCH branch is already existing."
                        sh "git checkout $BRANCH && git pull --rebase && git fetch --tags"
                    } else {
                        echo "The $BRANCH branch is not exsiting yet and needs to be created."
                        sh"""
                        git branch $BRANCH
                        git checkout $BRANCH
                        git remote set-url origin http://\"$GITLAB\"@ec2-3-125-51-254.eu-central-1.compute.amazonaws.com/adam/suggest-lib.git
                        git fetch --tags
                        """
                        MINOR_VERSION = BRANCH.tokenize("/")[1]
                        LATEST_TAG = sh(
                            script: "git describe --tags --abbrev=0 | grep -w $BRANCH || true"
                        )
                        echo "This is the expected version without patch: $MINOR_VERSION"
                        echo "This is the latest tag found that is related to given branch: $LATEST_TAG"
                    }
                }
            }
        }
        stage('*TEST* Release branch presentation') {
            when { branch "release/*" }
            steps {
                script {
                    BRANCH = params.Version
                    echo "You are currently on $BRANCH branch."
                }
            }
        }
        stage('*TEST* Master branch presentation') {
            when { branch "master" }
            steps {
                script {
                    BRANCH = params.Version
                    echo "You are currently on $BRANCH branch."
                }
            }
        }
        stage('*TEST* Feature branch presentation') {
            when { branch "feature" }
            steps {
                script {
                    BRANCH = params.Version
                    echo "You are currently on $BRANCH branch."
                }
            }
        }
        stage('Non-release branch build') {
            when {
                anyOf {
                    branch "release/*"
                    branch "master"
                    branch "feature"
                }
            }
            steps {
                script {
                    sh "mvn clean verify"
                }
            }
            post {
                success {
                    archiveArtifacts 'target/*.jar'
                }
            }
        }
        stage('Master branch publishing as SNAPSHOT') {
            when {
                anyOf {
                    branch "master"
                }
            }
            steps {
                script {
                    configFileProvider([configFile(fileId: 'fc3e184d-fd76-4262-a1e7-9a5671ebd340', variable: 'MAVEN_SETTINGS_XML')]) {
                        sh "mvn  -Dmaven.test.failure.ignore=true -DskipTests -s $MAVEN_SETTINGS_XML deploy"
                    }
                    echo 'SNAPSHOT version has been published in artifactory.'
                }
            }
        }
    }
}
