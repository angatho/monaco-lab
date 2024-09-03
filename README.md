# Monaco lab
## Monaco Use cases
- Keeping configuration history
  - Version control
  - Backups
- Automating deployment of new applications based on predefined parameters (such as application detection rules, tags, and SLOs)
- Bulk configuration deletion (such as deleting all configuration of a decomissioned application)
- Migrating configuration between environments
  - Commonly used in Managed to SaaS Migrations

## Monaco requirements
To use monaco you need to have a manifest file.

Deployment manifests are YAML files that tell the Dynatrace Monaco CLI what projects (this can be a  configuration or group of configurations) to deploy and where they should be deployed. 

This manifest file provides details on what to deploy and where to deploy it.

The manifest is saved as a YAML file. It has three top-level keys: manifestVersion, projects, and environmentGroups.

A sample manifest.yaml might look like this:

```yaml
manifestVersion: 1.0

projects:
- name: infra
  path: shared/infrastructure
- name: general
  path: general
  type: grouping

environmentGroups:
- name: dev
  environments:
  - name: test-env-1
    url:
      value: https://aaa.bbb.cc
    auth:
      token:
        name: TEST_ENV_TOKEN
  - name: test-env-2
    url:
      value: https://ddd.bbb.cc
    auth:
      token:
        name: TEST_ENV_2_TOKEN
- name: prod
  environments:
  - name: prod-env-1
    url:
      type: environment
      value: PROD_URL
    auth:
      token:
        name: PROD_TOKEN
```

For working with your environment this is the default manifest that you should start with

```yaml
manifestVersion: 1.0

environmentGroups:
  - name: development
    environments:
      - name: "<environment name>"
        url:
          value: "<environment url, for example https://abc12345.dynatrace.com/>"
        auth:
          token:
            name: "<API token environment variable, you can use the name token>"
```

Once you have this manifest with your environment url and an API token handy you are ready to being working with monaco.

## Installing monaco

1. Steps to install monaco can be found here 
https://docs.dynatrace.com/docs/shortlink/configuration-as-code-installation#install-the-dynatrace-monaco-cli

## Downloading environment configuration

1. Ensure you have your manifest file correctly configured, with an environment url and token environment variable
1. Export the API token to the environment variable defined in your manifest ```export token=<API token>```
1. Run the monaco download command, making sure to specify the manifest and the environment name ```monaco download --manifest manifest.yaml -e <environment name>```
1. Once the command has finished you will see a new directory called download followed by the timestamp of the execution, inside this directory will be the downloaded manifest of the configuration. Inside of the project directory will be all the configurations of your environment.

    <img width="1316" alt="monaco_download" src="https://github.com/user-attachments/assets/979e06ee-74fc-42d7-8923-e2b98986d3d1">


1. Take a look at the created yaml and json files. The json files define the setting configuration, whereas the yaml defines which of the setting configurations to be applied. 
> **_Note:_** The configuration used in the json files relies on the settings 2.0 framework

## Pushing configuration
1. Firstly we will be creating our directory structure for our configurations, we will be creating a directory called config, and then 2 directories inside called process-tag and slo.

    <img width="426" alt="monaco_push_directory_list" src="https://github.com/user-attachments/assets/1e05659f-5efc-4aa0-9843-b817b7c31063">


1. For the process-tag directory we will be adding the configuration for an automatic tagging rule which will tag all processes on the environment with the rule **Process_tag**

    <details>
      <summary>JSON configuration for tag (called <strong>process-tag.json</strong>)</summary>

      ```JSON
      {
        "name": "Process_test",
        "rules": [
            {
                "attributeRule": {
                    "conditions": [
                        {
                            "key": "PROCESS_GROUP_NAME",
                            "operator": "EXISTS"
                        }
                    ],
                    "entityType": "PROCESS_GROUP",
                    "pgToHostPropagation": false,
                    "pgToServicePropagation": false
                },
                "enabled": true,
                "type": "ME",
                "valueFormat": null,
                "valueNormalization": "Leave text as-is"
            }
        ]
    }
      ```
    </details>
    
    <br/>

    <details>
      <summary>YAML for applying configuration (called <strong>config.yaml</strong>)</summary>

      ```yaml
      configs:
        - id: process-tagging
          type:
            settings:
              schema: builtin:tags.auto-tagging
              scope: environment
          config:
            template: "process-tag.json"
            name: "process-tag"      
      ```
    </details>

    <br/>

1. For the slo directory we will be adding the configuration for a slo which will evaluate the service level avaiability for all services on the environment

    <details>
      <summary>JSON configuration a slo (called <strong>slo.json</strong>)</summary>

      ```JSON
      {
          "customDescription": null,
          "enabled": true,
          "errorBudgetBurnRate": {
              "burnRateVisualizationEnabled": true,
              "fastBurnThreshold": 10
          },
          "evaluationType": "AGGREGATE",
          "evaluationWindow": "-1w",
          "filter": "type(SERVICE)",
          "metricExpression": "(100)*(builtin:service.errors.server.successCount:splitBy())/(builtin:service.requestCount.server:splitBy())",
          "metricName": "all_services_95__target",
          "name": "All services 95% target",
          "targetSuccess": 95,
          "targetWarning": 97
      }
      ```
    </details>
    
    <br/>

    <details>
      <summary>YAML for applying configuration (called <strong>config.yaml</strong>)</summary>

      ```yaml
      configs:
        - id: slo
          type:
            settings:
              schema: builtin:monitoring.slo
              scope: environment
          config:
            name: "All services 95% target"
            template: slo.json
      ```
    </details>

    <br/>

1. Either download the files or copy the config into the respective json and config files.

1. Create the manifest. The manifest must contain the environment, token. Additionally you can also specify a project, this would make it easier when working with multiple configuration items (Such as SLOs, tags, application detection rules, etc ...)

    <details>
      <summary>Manifest file (called <strong>manifest.yaml</strong>)</summary>
      
      ```yaml
      manifestVersion: 1.0

      projects:
        - name: config
          path: config
          type: grouping

      environmentGroups:
      - name: development
        environments:
          - name: "<environment name>"
            url:
              value: "<environment url, for example https://abc12345.dynatrace.com/>"
            auth:
              token:
                name: "<API token environment variable>"
      ```
    </details>

    <br/>

1. You should have a directory stucture looking like this, where you have your manifest.yaml then a directory called config, then 2 more directories inside of config called process_tag and slo which then contains your relevant json and yaml files.

    <img width="383" alt="monaco_push_directory_structure" src="https://github.com/user-attachments/assets/9ee9ca6b-1e31-4ba3-9317-09967b3fa939">


1. To deploy the configuration make sure you either export your API token to the console via ```export token=<API token>```, navigate to the directory where your manifest file is located then run this command ```monaco deploy manifest.yaml -e <environment name>```

    The output should look like this, you will be able to see that the deployment of our tag and slo was successful

    <img width="1473" alt="monaco_deployment" src="https://github.com/user-attachments/assets/137dde08-62ef-4d35-af59-d08326d2f6fd">


    We can also validate this by checking out the slo and tag configuration

    **SLO configuration**
    <img width="1137" alt="slo_deployment_check" src="https://github.com/user-attachments/assets/812ce9cf-8b0d-4294-9271-a012a56b78c0">


    **Process tag configuration**
    <img width="1113" alt="process_tag_deployment_check" src="https://github.com/user-attachments/assets/124934a1-a263-4228-8127-d9f61ee68b6a">


1. Once you have done this and can see that these configurations have been deployed in your environment you have successfully used monaco to deploy configurations!

## Deleting configuration
1. Now we will be deleting the configuration we have just applied through moncaco
1. To delete configuration via monaco we will need something called a delete file. Luckily monaco comes equiped with a handy generate command that can create the required delete file.
1. We can run this command ```monaco generate deletefile manifest.yaml``` in the same directory as our manifest file (that has been named manifest.yaml)
    <img width="769" alt="generate_deletefile" src="https://github.com/user-attachments/assets/abd6a866-892b-4e05-8334-86967258439c">

1. Now that we have our delete file by running the monaco delete command we will remove everything included within our delete file.
1. We can delete our applied configuration with this command, making sure that we specify our manifest, delete file, and environment. ```monaco delete --manifest <manifest.yaml> --file <delete.yaml> -e <environment-name>```
    <img width="1451" alt="monaco_delete" src="https://github.com/user-attachments/assets/d9f38a86-ad95-4e62-b730-b38ae9ef3b6a">

1. Once you have run this command you will have deleted the configuration that we had applied earlier.

> We can also validate that this took affect not just from the command line but by viewing the tag and slo configurations where we will find our previously applied configurations no longer present.


## Pushing only certain configurations
1. You may be wondering what happens if you have multiple configurations/schemas in your directory structure but you only want to deploy a single configuration. This can be achieved and it's why we used the specific directory structure and set the project type to grouping. This will allow us to choose individual directories within our project.
1. You could either choose make a separate manifest file which only includes, the individual configuration that you want to push. However, the monaco manifest comes equiped with a method to deal with this called projects. We can use the type grouping, under the projects making sure to specify the name and path of our project as well which can be used to include all configuration items/directories within the specific project directory.
    
    <details>
      <summary>Manifest file (called <strong>manifest.yaml</strong>)</summary>
      
      ```yaml
      manifestVersion: 1.0
      
      #Projects section where we specify the project name, path, and type
      projects:
        - name: config
          path: config
          type: grouping
      
      environmentGroups:
      - name: development
        environments:
          - name: "<environment name>"
            url:
              value: "<environment url, for example https://abc12345.dynatrace.com/>"
            auth:
              token:
                name: "<API token environment variable>"
      ```
    </details>

    <br/>

    <details>
      <summary>Directory structure</summary>
      
      <img width="383" alt="monaco_push_directory_structure" src="https://github.com/user-attachments/assets/45d9651c-be8e-4ec7-ad89-07fcbe11104a">

    </details>

    <br/>
  
  1. Now we want to only push our tag configuration without deploying the slo configuration as well. We can do this by using the monaco deploy command, but this time we can specify the project and the sub directory which contains the chosen configuration denoted with a peroid (.) in between the directories. ```monaco deploy manifest.yaml  <environment name>  -p config.slo```

      From the output we can see that only the slo configuration was pushed to your Dynatrace environment

      <img width="1038" alt="monaco_push_single_config" src="https://github.com/user-attachments/assets/017119e6-2925-431c-bd92-7d13febe6be7">


## Deleting only certain configurations
1. Similar to the above scenario, we can delete only individual configurations instead of everything within the project. 
1. We can either do this by generating a deletefile only for the specific configuration items in our project, or by manually modifying the full delete file. I will be generating a delete file which will be used to only delete the slo configuration we applied earlier ```monaco generate deletefile manifest.yaml --file slo_delete.yaml -p config.slo```

    <img width="915" alt="generate_slo_deletefile" src="https://github.com/user-attachments/assets/bff80015-7689-412f-8d94-f4e1a83215d9">


    <details>
      <summary>Generated delete file (called <strong>slo_delete.yaml</strong>)</summary>
      
      ```yaml
      delete:
      - project: config.slo
        type: builtin:monitoring.slo
        id: slo
      ```
    </details>    

    <br>

1. When we have our delete file created, we can run ```monaco delete --manifest manifest.yaml --file slo_delete.yaml -e <environment name>```

    From the output we can see that only the slo configuration was deleted from your Dynatrace environment
    <img width="1051" alt="monaco_delete_slo" src="https://github.com/user-attachments/assets/f6eb84e4-89ed-4509-ba75-4546c30f3efc">


## Task 1: Pushing configuration
Create an automated tagging rule which will tag all hosts in your environment with the tag monaco_lab

## Tast 2: Deleting configuration
Delete the created tagging rule from the environment without deleting anything else
