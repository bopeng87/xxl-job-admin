
pipeline {
  environment {
    mavenRegistry = "registry-vpc.cn-beijing.aliyuncs.com/lemonbox/lemonbox_setting_maven"
    registry = "registry-vpc.cn-beijing.aliyuncs.com/lemonbox/xxl-job-admin"
    registryCredential = "dockerhub"
    parentWorkspace = "$WORKSPACE"
    FAILED_STAGE=""
    CURRENT_ENVIRONMENT=""
    BLUE_OR_GREEN=""
  }
  agent any
  stages {
    stage("Deply Parent") {
        agent {
            docker {
                image mavenRegistry
                registryUrl 'https://registry-vpc.cn-beijing.aliyuncs.com'
                registryCredentialsId registryCredential
                args '-v $HOME/.m2/repository:/root/.m2/repository -u root'
            }
        }
        steps {
            script {
            FAILED_STAGE=env.STAGE_NAME
                sh "mvn -U clean install -Dmaven.test.skip=true"
                // 拷贝到容器外
                sh "cp target/xxl-job-admin-0.0.1-SNAPSHOT.jar " + parentWorkspace +"/xxl-job-admin-0.0.1-SNAPSHOT.jar"
            }
        }
    }
    stage("Building image") {
      steps {
        script {
          FAILED_STAGE=env.STAGE_NAME
          docker.build(registry)
        }
      }
    }
    stage("Deploy Image") {
      steps {
        script {
          FAILED_STAGE=env.STAGE_NAME
          docker.withRegistry('https://registry-vpc.cn-beijing.aliyuncs.com', registryCredential) {
            docker.image(registry).push("$GIT_COMMIT")
          }
        }
      }
    }
    stage("Update k8s cluster") {
      options {
        timeout(time: 8, unit: 'MINUTES')
      }
      steps {
        script {
          FAILED_STAGE = env.STAGE_NAME

          if (env.GIT_BRANCH == "origin/master") {
            sh "kubectl config use aws_nx_data"
            sh "cat k8s/deployment.yml | sed 's/{{GIT_COMMIT}}/$GIT_COMMIT/g' | kubectl apply -f -"
            sh "kubectl rollout status deployment/xxl-job-admin"
          } else {
            sh "kubectl config use test"
            sh "cat k8s/deployment-test.yml | sed 's/{{GIT_COMMIT}}/$GIT_COMMIT/g' | kubectl apply -f -"
            sh "kubectl rollout status deployment/xxl-job-admin-test"
          }
        }
      }
    }
  }
  post {
    failure {
      sh "curl -d '{\"commitId\": \"$GIT_COMMIT\", \"branch\": \"$GIT_BRANCH\", \"build\": \"$BUILD_ID\"}' -H \"Content-Type: application/json\" -X POST http://ec2-54-222-181-255.cn-north-1.compute.amazonaws.com.cn:8888/build/fail"
    }
    aborted {
      script {
        if (FAILED_STAGE == "Update k8s cluster") {
          if (env.GIT_BRANCH == "origin/master") {
            sh "kubectl config use aws_nx_data"
            sh "kubectl rollout undo deployment/xxl-job-admin"
          } else {
            sh "kubectl config use test"
            sh "kubectl rollout undo deployment/xxl-job-admin-test"
          }
        }
      }
    }
  }
}
