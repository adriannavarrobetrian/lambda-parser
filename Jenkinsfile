def functionName = 'MoviesParser'
def imageName = 'adriannavarro/parser'
def bucket = 'deployment-packages-watchlist.test123456'
def region = 'eu-west-1'

node('jenkins_agent'){
    try{
        stage('Checkout'){
            checkout scm
        }

        def imageTest= docker.build("${imageName}-test", "-f Dockerfile.test .")

        stage('Pre-integration Tests'){
            parallel(
                'Quality Tests': {
                    imageTest.inside{
                        sh 'golint'
                    }
                },
                'Unit Tests': {
                    imageTest.inside{
                        sh 'go test'
                    }
                },
                'Security Tests': {
                    imageTest.inside('-u root:root'){
                        sh 'nancy /go/src/github/adriannavarro/parser/Gopkg.lock'
                    }
                }
            )
        }

        stage('Build'){
            sh """
                docker build -t ${imageName} .
                containerName=\$(docker run -d ${imageName})
                docker cp \$containerName:/go/src/github.com/adriannavarro/parser/main main
                docker rm -f \$containerName
                zip -r ${commitID()}.zip main
            """
        }

        stage('Push'){
            sh "aws s3 cp ${commitID()}.zip s3://${bucket}/${functionName}/"
        }

        stage('Deploy'){
            sh "aws lambda update-function-code --function-name ${functionName} \
                    --s3-bucket ${bucket} --s3-key ${functionName}/${commitID()}.zip \
                    --region ${region}"

            sh "aws lambda publish-version --function-name ${functionName} \
                    --description ${commitID()} --region ${region}"
        }
    } catch(e){
        currentBuild.result = 'FAILED'
        throw e
    } finally {
        notifySlack(currentBuild.result)
        sh "rm -rf ${commitID()}.zip"
    }
}

def commitAuthor(){
    sh 'git show -s --pretty=%an > .git/commitAuthor'
    def commitAuthor = readFile('.git/commitAuthor').trim()
    sh 'rm .git/commitAuthor'
    commitAuthor
}

def commitID() {
    sh 'git rev-parse HEAD > .git/commitID'
    def commitID = readFile('.git/commitID').trim()
    sh 'rm .git/commitID'
    commitID
}

def commitMessage() {
    sh 'git log --format=%B -n 1 HEAD > .git/commitMessage'
    def commitMessage = readFile('.git/commitMessage').trim()
    sh 'rm .git/commitMessage'
    commitMessage
}