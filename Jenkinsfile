@Library('dynatrace@master') _

def tagMatchRules = [
  [
    meTypes: [
      [meType: 'SERVICE']
    ],
    tags : [
      [context: 'CONTEXTLESS', key: 'environment', value: 'production']
    ]
  ]
]

pipeline {
  agent {
    label 'kubegit'
  }
  stages {
    stage('Update production versions with latest versions from staging') {
      steps {
        container('kubectl') {
          sh "sed -i \"s#image: .*#image: `kubectl -n staging get deployment -o jsonpath='{.items[*].spec.template.spec.containers[0].image}' --field-selector=metadata.name=carts`#\" carts.yml"
          sh "sed -i \"s#image: .*#image: `kubectl -n staging get deployment -o jsonpath='{.items[*].spec.template.spec.containers[0].image}' --field-selector=metadata.name=catalogue`#\" catalogue.yml"
          sh "sed -i \"s#image: .*#image: `kubectl -n staging get deployment -o jsonpath='{.items[*].spec.template.spec.containers[0].image}' --field-selector=metadata.name=front-end`#\" front-end.yml"
          sh "sed -i \"s#image: .*#image: `kubectl -n staging get deployment -o jsonpath='{.items[*].spec.template.spec.containers[0].image}' --field-selector=metadata.name=orders`#\" orders.yml"
          sh "sed -i \"s#image: .*#image: `kubectl -n staging get deployment -o jsonpath='{.items[*].spec.template.spec.containers[0].image}' --field-selector=metadata.name=payment`#\" payment.yml"
          sh "sed -i \"s#image: .*#image: `kubectl -n staging get deployment -o jsonpath='{.items[*].spec.template.spec.containers[0].image}' --field-selector=metadata.name=queue-master`#\" queue-master.yml"
          sh "sed -i \"s#image: .*#image: `kubectl -n staging get deployment -o jsonpath='{.items[*].spec.template.spec.containers[0].image}' --field-selector=metadata.name=shipping`#\" shipping.yml"
          sh "sed -i \"s#image: .*#image: `kubectl -n staging get deployment -o jsonpath='{.items[*].spec.template.spec.containers[0].image}' --field-selector=metadata.name=user`#\" user.yml"
          script{
            env.CARTS_DT_PROPS = "${sh(script:'kubectl -n staging get deployment  -o jsonpath=\'{.items[*].spec.template.spec.containers[0].env[?(@.name==\"DT_CUSTOM_PROP\")].value}\' --field-selector=metadata.name=carts', returnStdout: true)}"
          }
          sh "printenv | sort"
          sh "sed -i 's#value: \"DT_CUSTOM_PROP_PLACEHOLDER\".*#value: \"${env.CARTS_DT_PROPS}\"#' carts.yml"
          sh "cat carts.yml"
          sh "kubectl -n production apply -f ."
        }
      }
    }
    stage('DT Deploy Event') {
      steps {
        container("curl") {
          script {
            def status = pushDynatraceDeploymentEvent (
              tagRule : tagMatchRules,
              customProperties : [
                [key: 'Jenkins Build Number', value: "${env.BUILD_ID}"],
                [key: 'Git commit', value: "${env.GIT_COMMIT}"]
              ]
            )
          }
        }
      }
    }
    stage('DT Create Application') {
      steps {
        container("kubectl"){
          script{
            env.FRONT_END_IP = "${sh(script:'kubectl get svc front-end-ext -n production -o jsonpath=\'{.status.loadBalancer.ingress[0].ip}\'', returnStdout: true)}"
          }
        }
        container("curl") {
          script {
            def status = dt_createUpdateAppDetectionRule (
              dtAppName : "sockshop.production",
              pattern : "http://${env.FRONT_END_IP}:8080",
              applicationMatchType: "CONTAINS",
              applicationMatchTarget: "URL"
            )
          }
        }
      }
    }
  }
}
