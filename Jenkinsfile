pipeline {
    // 스테이지 별로 다른 거
    agent any

    triggers {
        pollSCM('*/3 * * * *')
    }

    environment {
      AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId')
      AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey')
      AWS_DEFAULT_REGION = 'ap-northeast-2'
      HOME = '.' // Avoid npm root owned
    }

    stages {
        // 레포지토리를 다운로드 받음
        stage('Prepare') {
            agent any
            
            steps {
                echo 'Clonning Repository'

                git url: 'https://github.com/78won96/jenkins.git',
                    branch: 'main',
                    credentialsId: 'jenkins_token_github'
            }

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success {
                    echo 'Successfully Cloned Repository'
                }

                always {
                  echo "i tried..."
                }

                cleanup {
                  echo "after all other post condition"
                }
            }
        }
        
        // aws s3 에 파일을 올림
        stage('Deploy Frontend') {
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance profile 을 등록해야함.
            dir ('./website'){
                sh '''
                aws s3 sync ./ s3://jenkins-test-jwlee
                '''
            }
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {
                  echo 'Successfully Cloned Repository'

              }

              failure {
                  echo 'I failed :('

                  
              }
          }
        }
        
        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능!
            agent {
              docker {
                image 'node:latest'
              }
            }
            
            steps {
              dir ('./server'){
                  sh '''
                  npm install&&
                  npm run lint
                  '''
              }
            }
        }
        
        stage('Test Backend') {
          agent {
            docker {
              image 'node:latest'
            }
          }
          steps {
            echo 'Test Backend'

            dir ('./server'){
                sh '''
                npm install
                npm run test
                '''
            }
          }
        }
        stage('Scan') {
          steps {
              // Scan the image
              prismaCloudScanImage ca: '',
              cert: '',
              dockerAddress: 'unix:///var/run/docker.sock',
              image: 'server*',
              key: '',
              logLevel: 'info',
              podmanPath: '',
              project: '',
              resultsFile: 'prisma-cloud-scan-results.json',
              ignoreImageBuildTime:true
              } 
          
          post {
             always {
            // The post section lets you run the publish step regardless of the scan results
            prismaCloudPublish resultsFilePattern: 'prisma-cloud-scan-results.json'
             } 
          }
        }
        
        stage('Bulid Backend') {
          agent any
          steps {
            echo 'Build Backend'

            dir ('./server'){
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }

          post {
            failure {
              error 'This pipeline stops here...'
            }
          }
        }

        // stage('Scan') {
        //   steps {
        //       // Scan the image
        //       prismaCloudScanImage ca: '',
        //       cert: '',
        //       dockerAddress: 'unix:///var/run/docker.sock',
        //       image: 'server*',
        //       key: '',
        //       logLevel: 'info',
        //       podmanPath: '',
        //       project: '',
        //       resultsFile: 'prisma-cloud-scan-results.json',
        //       ignoreImageBuildTime:true
        //       } 
          
        //   post {
        //      always {
        //     // The post section lets you run the publish step regardless of the scan results
        //     prismaCloudPublish resultsFilePattern: 'prisma-cloud-scan-results.json'
        //      } 
        //   }
        // }

        stage('Deploy Backend') {
          agent any

          steps {
            echo 'Build Backend'

            dir ('./server'){
                sh '''
                docker rm -f $(docker ps -aq)
                docker run -p 80:80 -d server
                '''
            }
          }

          post {
            success {
              mail  to: 'frontalnh@gmail.com',
                    subject: "Deploy Success",
                    body: "Successfully deployed!"
                  
            }
          }
        }
    }
}
