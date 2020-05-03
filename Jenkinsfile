pipeline {
   agent any

   stages {
      stage('Build') {
        steps {
            sh "hugo"
        }
      }
      stage('Deploy') {
        steps {
            s3Upload(
                profileName: "jenkins_s3publisher",
                entries: [[
                    bucket: "egedev",
                    sourceFile: "public/**",
                    selectedRegion: "eu-central-1"
                ]],
                dontWaitForConcurrentBuildCompletion: false,
                dontSetBuildResultOnFailure: false,
                pluginFailureResultConstraint: "FAILURE",
                consoleLogLevel: "INFO",
                userMetadata: []
            )
        }
      }
   }
}
