pipeline{
  agent any 

  environment {
    GIT_REPO = "https://gitee.com/kormondor/blog-hexo"
    CONFIG_FILE = "_config.yml"
  }

  stages{
    stage("获取更新代码"){
      steps{
        echo "Git仓库代码: ${GIT_REPO}"
        script {
          if (fileExists (file: "${CONFIG_FILE}")){
            echo "项目已存在 准备更新代码"
            sh "git pull origin master"
          }else{
            echo "项目不存在，准备拉取代码"
            git "${GIT_REPO}"
          }
        }
      }
      stage("下载依赖"){
        steps{
          echo "开始下载依赖"
          sh "yarn install"
        }
      }

      stage("打包"){
        steps{
          echo "开始打包"
          sh "yarn clean & yarn build"
        }
      }
    }
  }
}