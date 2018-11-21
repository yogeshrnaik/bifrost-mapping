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

                        sh 'rm -rf terraform-repo'
                        dir('terraform-repo') {
                            git url: 'git@github.com:ecsworkshop2018/bifrost-infra-provisioner.git'

                            dir('terraform/cluster-and-services') {
                                setupPythonVirtualEnv()

                                def backendConfigPath = populateBackendConfigFile(
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
                                        instanceType,
                                        backendConfigPath
                                )

                                terraformPlan(backendConfigPath)

                                terraformApply()

                                commitBackendConfig(backendConfigPath)
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
    backendConfigPath = "backend-configs/${env}-${userUniqueName}-${clusterName}"
    sh "mkdir -p ${backendConfigPath}"
    backendConfigFile = "${backendConfigPath}/backend.config"
    backenConfigs = [
        terraformStringVar("bucket", stateBucket),
        terraformStringVar("key", "${userUniqueName}-${clusterName}-cluster.tfstate")
    ]
    writeFile encoding: "UTF-8", file: "${backendConfigFile}", text: backenConfigs.join("\n")
    return "${backendConfigPath}"
}

def populateTerraformTfvars(env, vpc, alb, userUniqueName, clusterName, instanceType, serviceConfigs, backendConfigPath) {
    varFile = "${backendConfigPath}/terraform.tfvars"
    vars = [
        terraformStringVar("env", env),
        terraformStringVar("vpc_name", vpc),
        terraformStringVar("alb_name", alb),
        terraformStringVar("unique_name", userUniqueName),
        terraformStringVar("cluster_name", clusterName),
        terraformStringVar("instance_type", instanceType),
        terraformListVar("service_names", serviceConfigs.collect { serviceConfig -> serviceConfig.name }),
        terraformListVar("service_memories", serviceConfigs.collect { serviceConfig -> serviceConfig.memory }),
        terraformListVar("service_cpus", serviceConfigs.collect { serviceConfig -> serviceConfig.cpu }),
        terraformListVar("docker_images", serviceConfigs.collect { serviceConfig -> serviceConfig.docker_image })
    ]
    writeFile encoding: "UTF-8", file: "${varFile}", text: vars.join("\n")
}

def terraformStringVar(key, value) {
    return "${key} = " + """ "${value}" """
}

def terraformListVar(key, values) {
    return "${key} = [" + values.collect({ value -> """ "${value}" """ }).join(",") + "]"
}

def terraformPlan(backendConfigPath) {
    wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'xterm']) {
        sh """
            . venv/bin/activate
            TF_IN_AUTOMATION=true 
            terraform get -update=true
            terraform init -input=false -backend-config=${backendConfigPath}/backend.config
            terraform plan -input=false -out=terraform.tfplan -var-file=${backendConfigPath}/terraform.tfvars
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

def commitBackendConfig(backendConfigPath) {
    sh """
        git add ${backendConfigPath}/backend.config
        git add ${backendConfigPath}/terraform.tfvars
        if [ ! \$(git status -s --untracked-files=no | wc -l) -eq 0 ]; then
            git commit -m "Backend config for applied terraform"
            git push origin master
        fi
       """
}