pipeline{
  agent any 

  environment {
    GIT_REPO = "https://gitee.com/kormondor/blog-hexo"
    CONFIG_FILE = "_config.yml"
  }

  stages{
    stage("update source code"){
      steps{
        echo "Git repo: ${GIT_REPO}"
        script {
          if (fileExists (file: "${CONFIG_FILE}")){
            echo "project existed, will pull remote respository"
            sh "git pull origin master"
          }else{
            echo "project not exist, will clone respository"
            git "${GIT_REPO}"
          }
        }
      }
    }
    stage("Download dependency"){
        steps{
          echo "downloading dependency"
          sh "yarn install"
        }
    }

    stage("Build"){
        steps{
          echo "Building start"
          sh "yarn clean & yarn build"
        }
    }
  }
}