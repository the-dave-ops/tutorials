# Gitlab CI file for creating a automated gitlab release.
This GitLab CI configuration file is designed for managing releases within a project repository that oversees multiple microservices. Each microservice is assigned its own version tag, and when initiating a new release, the CI pipeline executes to collect the current version tags of all microservices. Subsequently, it updates the Helm chart with the new version information and proceeds to create and deploy the release onto Kubernetes. The main stages of this CI process include:

1. Calculating the version tag for the new release.
2. Gthering information on the latest version tags of all microservices.
3. Updating the Helm chart.
5. Creating a new release.
6. Deploying the release to Kubernetes.

While this process may seem complex, the provided GitLab CI configuration streamlines these tasks efficiently.

the file uses modules, but this is the shortend version of the full file, so it is easy to understand the logic behind the process.

```
stages:         
  - calculate_release_tag
  - get_all_microservices_tags
  - build
  - create_chart
  - commit
  - release
  - deploy

include: #######
  - project: devops/modules/regular-modules/release-control
    file: create_release_of_all_microservices.yaml
    ref: master
  - project: devops/modules/regular-modules/trigger
    file: trigger_devops_repo_from_microservice_develop_branch.yaml 
    ref: master
  - project: devops/modules/regular-modules/release-control
    file: calculate_new_tag.yaml 
    ref: master
  - project: devops/modules/regular-modules/conditions
    file: devops_repo_deployment.yaml
    ref: master
  - project: devops/modules/regular-modules/connection
    file: connect_to_okd_cluster.yaml
    ref: master
  - project: devops/modules/regular-modules/deploy
    file: deploy_to_kubernetes-getapp.yaml
    ref: master


#==========================================================#
## this first job only calculate's this current release tag ######
#==========================================================#
calculate_new_release_tag: 
  stage: calculate_release_tag
  variables:
    MICROSERVICE_PROJECT_ID: "51"
    OTHER_PROJECT_ID: "51"
  script:
    - echo "the new release tag is ${NEW_TAG}"
    - echo GETAPP_RELEASE_TAG=${NEW_TAG} > .env
    - cat .env
    - mv NEW_TAG_RELEASE.txt stuff/NEW_TAG_RELEASE.txt
  extends:
    .calculate_new_release_tag
  artifacts:
    paths:
      - .env
      - stuff/NEW_TAG_RELEASE.txt
    #   # temp
    #   - .env-release
    reports:
      dotenv: .env-release                                ## Use artifacts:reports:dotenv to expose the variables to other jobs
  except:
    - tags
  only:
    - main

# #==========================================================#
# # get the tag's of all the images in this release
# #==========================================================#

# get_last_repo_tag:
#   stage: get_all_microservices_tags
#   script:
#     - |
#       ACCESS_TOKEN="${GITLAB_ACCESS_TOKEN}"
#       GITLAB_DOMAIN="gitlab.getapp.sh"
#       # api
#       API_TAG=$(curl -s --header "PRIVATE-TOKEN: $ACCESS_TOKEN" "https://$GITLAB_DOMAIN/api/v4/projects/20/repository/tags" | awk -F'"' '/name/{print $4; exit}')
#       echo "the api tag is $API_TAG"
#       echo API_TAG=${API_TAG} >> .env

#       # delivery
#       DELIVERY_TAG=$(curl -s --header "PRIVATE-TOKEN: $ACCESS_TOKEN" "https://$GITLAB_DOMAIN/api/v4/projects/39/repository/tags" | awk -F'"' '/name/{print $4; exit}')
#       echo "the delivery tag is $DELIVERY_TAG"
#       echo DELIVERY_TAG=${DELIVERY_TAG} >> .env

#       # deploy
#       DEPLOY_TAG=$(curl -s --header "PRIVATE-TOKEN: $ACCESS_TOKEN" "https://$GITLAB_DOMAIN/api/v4/projects/38/repository/tags" | awk -F'"' '/name/{print $4; exit}')
#       echo "the deploy tag is $DEPLOY_TAG"
#       echo DEPLOY_TAG=${DEPLOY_TAG} >> .env

#       # Discovery 
#       DISCOVERY_TAG=$(curl -s --header "PRIVATE-TOKEN: $ACCESS_TOKEN" "https://$GITLAB_DOMAIN/api/v4/projects/42/repository/tags" | awk -F'"' '/name/{print $4; exit}')
#       echo "the discovery tag is $DISCOVERY_TAG"
#       echo DISCOVERY_TAG=${DISCOVERY_TAG} >> .env

#       # offering
#       OFFERING_TAG=$(curl -s --header "PRIVATE-TOKEN: $ACCESS_TOKEN" "https://$GITLAB_DOMAIN/api/v4/projects/42/repository/tags" | awk -F'"' '/name/{print $4; exit}')
#       echo "the offering tag is $OFFERING_TAG"
#       echo OFFERING_TAG=${OFFERING_TAG} >> .env

#       # Project Management 
#       PROJECT_MANAGMENT_TAG=$(curl -s --header "PRIVATE-TOKEN: $ACCESS_TOKEN" "https://$GITLAB_DOMAIN/api/v4/projects/43/repository/tags" | awk -F'"' '/name/{print $4; exit}')
#       echo "the project_management tag is $PROJECT_MANAGMENT_TAG"
#       echo PROJECT_MANAGMENT_TAG=${PROJECT_MANAGMENT_TAG} >> .env

#       # Upload 
#       UPLOAD_TAG=$(curl -s --header "PRIVATE-TOKEN: $ACCESS_TOKEN" "https://$GITLAB_DOMAIN/api/v4/projects/40/repository/tags" | awk -F'"' '/name/{print $4; exit}')
#       echo "the upload tag is $UPLOAD_TAG"
#       echo UPLOAD_TAG=${UPLOAD_TAG} >> .env

#       # getmap-node 
#       GETMAP_TAG=$(curl -sSL "https://api.github.com/repos/getappsh/getmap/tags" | jq -r '.[0].name')
#       echo "the tag is - $GETMAP_TAG"
#       echo GETMAP_TAG=${GETMAP_TAG} >> .env

#       # dashboard 
#       DASHBOARD_TAG=$(curl -s --header "PRIVATE-TOKEN: $ACCESS_TOKEN" "https://$GITLAB_DOMAIN/api/v4/projects/40/repository/tags" | awk -F'"' '/name/{print $4; exit}')
#       echo "the dashboard tag is $DASHBOARD_TAG"
#       echo DASHBOARD_TAG=${DASHBOARD_TAG} >> .env
#       cat .env
#   artifacts:
#     paths:
#       - .env
#   dependencies:
#     - calculate_new_release_tag
#   except:
#     - tags
#   tags:
#     - shell
#   only:
#     - main

# #==========================================================#
# # a stage to build a custom env file with no automation
# #==========================================================#

create_custom_env_file:
  stage: get_all_microservices_tags
  script:
    - |
      echo API_TAG=1.2.37-even-V2 >> .env
      echo DELIVERY_TAG=1.1.41-even-V2 >> .env
      echo DEPLOY_TAG=1.1.28-even-v2 >> .env
      echo DISCOVERY_TAG=1.1.36-even-v2 >> .env
      echo OFFERING_TAG=1.1.28-even-v2 >> .env
      echo PROJECT_MANAGMENT_TAG=1.1.15-even-v2 >> .env
      echo UPLOAD_TAG=1.1.22-even-v2 >> .env
      echo GETMAP_TAG=1.0.46-even-v2 >> .env
      echo DASHBOARD_TAG=1.2.12 >> .env
  only:
    - main
  artifacts:
    paths:
      - .env
  dependencies:
    - calculate_new_release_tag
  except:
    - tags
  tags:
    - shell


#==========================================================#
# check that the file is OK, before creating the release.
#==========================================================#

create_full_tag_file:
  stage: build
  script:
    - |
      #!/bin/bash
      cat .env

        # Specify the path to your .env file
      env_file=".env"

        # Check if the .env file exists
      if [ -f "$env_file" ]; then
        # Read the .env file line by line and export each variable
        while IFS= read -r line; do
          export "$line"
        done < "$env_file"
        echo "Variables exported from $env_file"
      else
        echo "$env_file does not exist."
      fi
      envsubst < stuff/getapp-release-info-tamplate.txt > stuff/getapp-release-info.txt
      envsubst < stuff/images-list-template.txt > getapp-images-list.txt
      cat stuff/getapp-release-info.txt
      cat getapp-images-list.txt
      chmod +x test-existens-in-harbor.sh
      cat stuff/getapp-release-info.txt > readme.md
      echo $CI_JOB_URL > stuff/url_job_link.txt
        # create a commit with the new files
      project_url=$(echo $CI_PROJECT_URL | sed 's/https:\/\///')
      echo $project_url
      #./test-existens-in-harbor.sh
  dependencies:
    #- get_last_repo_tag
    - create_custom_env_file
  tags:
    - shell
  artifacts:
    paths:
      - .env
      - getapp-release-info.txt
      - stuff/url_job_link.txt
      - readme.md
  except:
    - tags
  only:
    - main


#====================================================#
# update the new helm chart with the new values
#====================================================#

update_helm_chart: 
  stage: create_chart
  tags:
    - shell 
  variables:
    NAMESPACE: ""
    GITLAB_TOKEN: "$GITLAB_TOKEN"
    SSL_ISSUER: $SSL_ISSUER
    CONFIGMAP: $NAMESPACE
    HELM_PROJECT: $NAMESPACE
    replicaCount: 1
  before_script:
    - |
      #!/bin/bash
      set -x
      pwd
      cat .env
        # Specify the path to your .env file
      env_file=".env"
      # Check if the .env file exists
      if [ -f "$env_file" ]; then
        # Read the .env file line by line and export each variable
        while IFS= read -r line; do
          export "$line"
        done < "$env_file"
        echo "Variables exported from $env_file"
      else
        echo "$env_file does not exist."
      fi
      sed -E -i "s,api1":" .*,api":" ${API_TAG},g" integration-chart/values.yml
      sed -E -i "s,delivery":" .*,delivery":" ${DELIVERY_TAG},g" integration-chart/values.yml
      sed -E -i "s,deploy":" .*,deploy":" ${DEPLOY_TAG},g" integration-chart/values.yml
      sed -E -i "s,discovery":" .*,discovery":" ${DISCOVERY_TAG},g" integration-chart/values.yml
      sed -E -i "s,offering":" .*,offering":" ${OFFERING_TAG},g" integration-chart/values.yml
      sed -E -i "s,projectmanagment":" .*,projectmanagment":" ${PROJECT_MANAGMENT_TAG},g" integration-chart/values.yml
      sed -E -i "s,upload":" .*,upload":" ${UPLOAD_TAG},g" integration-chart/values.yml
      sed -E -i "s,dashboard":" .*,dashboard":" ${DASHBOARD_TAG},g" integration-chart/values.yml
      sed -E -i "s,getmap":" .*,getmap":" ${GETMAP_TAG},g" integration-chart/values.yml
      sed -E -i "s,gitlabrelease":" .*,gitlabrelease":" ${GETAPP_RELEASE_TAG},g" integration-chart/values.yml

      # getapp and getmap #
      sed -E -i "s,configmap":" .*,configmap":" ${CONFIGMAP},g" integration-chart/values.yml
      sed -E -i "s,routePrefix":" .*,routePrefix":" ${CONFIGMAP},g" integration-chart/values.yml
      sed -E -i "s,dbName":" .*,dbName":" ${CONFIGMAP},g" integration-chart/values.yml
      sed -E -i "s,openshiftProjectName":" .*,openshiftProjectName":" ${CONFIGMAP},g" integration-chart/values.yml
      sed -E -i "s,replicaCount":" .*,replicaCount":" ${replicaCount},g" integration-chart/values.yml
      cat integration-chart/values.yml
  artifacts:
    paths:
      - integration-chart/Values.yaml
      - integration-chart/Chart.yaml
      - integration-chart/charts/getapp/Chart.yaml
  except:
    - tags
  extends:
    .update_helm_chart
  only:
    - main


#====================================================#
# commit file to git
#====================================================#

commit_files:
  stage: commit
  script:
    - |
      #!/bin/bash
      cat .env
      echo test >> stuff/nothing
      git config user.email "david@linnovate.net"
      git config user.name "ci-bot"
      git remote add gitlab4 https://oauth2:${GIT_COMMIT_TOKEN}@gitlab.getapp.sh/getapp/getapp-version-control.git || true
      git add .
      git tag $GETAPP_RELEASE_TAG || true
      git commit -am "push back from pipeline"
      git push gitlab4 HEAD:main -o ci.skip
      NEW_COMMIT_SHA=$(git rev-parse HEAD)
      echo the new commit sha is $NEW_COMMIT_SHA
      echo ${NEW_COMMIT_SHA} > commit_sha.txt
  dependencies:
    - create_full_tag_file
    - calculate_new_release_tag
    - update_helm_chart
  tags:
    - shell
  artifacts:
    paths:
      - .env
      - stuff/getapp-release-info.txt
      - stuff/url_job_link.txt
      - readme.md
      - commit_sha.txt
  except:
    - tags
  only:
    - main

#==========================================================#
## create the release. ###
#==========================================================#

create_release:
  stage: release
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  services:
    - name: registry.gitlab.com/gitlab-org/release-cli:latest
      alias: docker
  tags:
    - docker
  needs:
    - job: create_full_tag_file
      artifacts: true
    - job: calculate_new_release_tag
      artifacts: true
    - job: commit_files
      artifacts: true
  variables:
    TAG: $(cat commit_sha.txt)
    NEW_TAG: $(cat stuff/NEW_TAG_RELEASE.txt)-develop
  script:
    - cat stuff/NEW_TAG_RELEASE.txt
  release:
    #name: 'Release ${NEW_TAG}-even-prod-v2'
    name: 'Release ${NEW_TAG}'
    tag_name: ${NEW_TAG}
    ref: '$TAG'
    description: readme.md
    milestones: {}
    assets:
      links:
        - name: "getapp_release-${NEW_TAG}.zip"
          url: "${CI_JOB_URL}/artifacts/download"
  except:
    - tags
  dependencies:
    - create_full_tag_file
    - calculate_new_release_tag
    - commit_files
  artifacts:
    paths:
      - .env
      - stuff/getapp-release-info.txt
  only:
    - main

#====================================================#
# Deploy ###
#====================================================#

deploy_to_okd: 
  stage: deploy
  tags:
    - shell
  variables:
    NAMESPACE: ""
    GITLAB_TOKEN: "$GITLAB_TOKEN"
    SSL_ISSUER: $SSL_ISSUER
    CONFIGMAP: $NAMESPACE
    HELM_PROJECT: $NAMESPACE
  except:
    - tags
  extends:
    .kubernetes_deploy_with_repo_values_file
  only:
    - main



```