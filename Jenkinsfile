pipeline {
    agent any

    environment {
        USER_UNIQUE_NAME = "ecs-workshop"
    }

    stages {
        stage("terraform-plan-and-apply") {
            steps {
                script {
                    MAPPING = readYaml(file: "mapping.yaml")
                    ENV = MAPPING.env
                    VPC = MAPPING.vpc
                    ALB = MAPPING.alb
                    MAPPING.clusters.each { cluster ->
                        def clusterName = cluster.cluster_name
                        def serviceConfigs = cluster.services.collect { service -> getServiceConfig(service) }
                        def instanceType = getInstanceType(
                                getMininumInstanceCPU(serviceConfigs),
                                getMininumInstanceMemory(serviceConfigs)
                        )
                        print instanceType

                        sh 'rm -rf terraform-repo'
                        dir('terraform-repo') {
                            git url: 'git@github.com:ecsworkshop2018/bifrost-infra-provisioner.git'

                            dir('terraform/cluster-and-services') {
                                setupPythonVirtualEnv()

                                def backendConfigFile = populateBackendConfigFile(
                                        ENV,
                                        USER_UNIQUE_NAME,
                                        clusterName
                                )

                                populateTerraformTfvars(
                                        ENV,
                                        VPC,
                                        ALB,
                                        USER_UNIQUE_NAME,
                                        clusterName,
                                        instanceType
                                )

                                terraformPlan(backendConfigFile)

                                terraformApply()

                                commitBackendConfig(backendConfigFile)
                            }
                        }
                    }
                }
            }
        }
    }
}

def setupPythonVirtualEnv() {
    sh """
        virtualenv venv
        . venv/bin/activate
        pip install -r requirements.txt
        deactivate
       """
}

def getInstanceType(minCpu, minMemory) {
    return "t3.micro"
}

int getMininumInstanceCPU(serviceConfigs) {
    return max(serviceConfigs.collect { serviceConfig -> serviceConfig.cpu })
}

int getMininumInstanceMemory(serviceConfigs) {
    return max(serviceConfigs.collect { serviceConfig -> serviceConfig.memory })
}

int max(intList) {
    return intList.inject(0, { result, i -> result > i ? result : i })
}

def getServiceConfig(service) {
    sh 'rm -rf service-repo'
    dir('service-repo') {
        git url: service.service_repo
        serviceConfig = readYaml(file: service.service_config_path)
        serviceConfig.dockerImage = service.docker_image
        return serviceConfig
    }
}

def populateBackendConfigFile(env, userUniqueName, clusterName) {
    stateBucket = "ecs-workshop-terraform-state-${env}"
    sh "mkdir -p backend-configs"
    backendConfigFile = "backend-configs/${stateBucket}-${userUniqueName}-${clusterName}"
    sh """
        echo "bucket = \\"${stateBucket}\\"" > ${backendConfigFile}
        echo "key    = \\"${userUniqueName}-${clusterName}-cluster.tfstate\\"" >> ${backendConfigFile}
       """
    return "${backendConfigFile}"
}

def populateTerraformTfvars(env, vpc, alb, userUniqueName, clusterName, instanceType) {
    varFile = "terraform.tfvars"
    sh """
        echo "env           = \\"${env}\\"" > ${varFile}
        echo "vpc_name      = \\"${vpc}\\"" >> ${varFile}
        echo "alb_name      = \\"${alb}\\"" >> ${varFile}
        echo "unique_name   = \\"${userUniqueName}\\"" >> ${varFile}
        echo "cluster_name  = \\"${clusterName}\\"" >> ${varFile}
        echo "instance_type = \\"${instanceType}\\"" >> ${varFile}
       """
}

def terraformPlan(backendConfigFile) {
    wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'xterm']) {
        sh """
            . venv/bin/activate
            TF_IN_AUTOMATION=true 
            terraform get -update=true
            terraform init -input=false -backend-config=${backendConfigFile}
            terraform plan -input=false -out=terraform.tfplan
            deactivate
           """
    }
}

def terraformApply() {
    wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'xterm']) {
        sh """
            . venv/bin/activate
            TF_IN_AUTOMATION=true
            terraform apply -input=false terraform.tfplan
            deactivate
           """
    }
}

def commitBackendConfig(backendConfigFile) {
    sh """
        git add ${backendConfigFile}
        git commit -m "Backend config for applied terraform"
        git push origin master
       """
}