# Red Hat Developer Hub with Jenkins Integration Showcase

[Red Hat Developer Hub](https://developers.redhat.com/rhdh/overview) is an enterprise-grade Internal Developer Portal, combined with Red Hat OpenShift, Red Hat Developer Hub allows platform engineering teams to offer software templates, pre-architected and supported approaches to maximize developer skills, ease onboarding, and increase development productivity, and focus on writing great code by reducing friction and frustration for development teams. Red Hat Developer Hub also offers Software Templates to simplify the development process, which can  reduce friction and frustration for development teams, boosting their productivity and increasing an organization's competitive advantage.

Red Hat provides OpenShift Pipelines based on Tekton for CI/CD implementation on OpenShift. Customers who are using OpenShift can utilize the OpenShift Pipelines for integrated CI/CD implementation. However, many organizations are still using Jenkins for their CI/CD engine, this document outlines the necessary steps to enable the Red Hat Developer Hub and Jenkins integration.

## Installing Red Hat Developer Hub

You can install Red Hat Developer Hub using Operator and [Helm Chart](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.3/html/installing_red_hat_developer_hub_on_openshift_container_platform/index#assembly-install-rhdh-ocp-helm). This guide will follow the steps in Red Hat product documentation to install Red Hat Developer Hub using Operator.

Please follow the [Installing Red Hat Developer Hub on OpenShift Container Platform with the Operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.3/html/installing_red_hat_developer_hub_on_openshift_container_platform/index#assembly-install-rhdh-ocp-operator) to install the Red Hat Developer Hub.

## Installing and Configuring Jenkins

1. Clone the this repo into your local directory.

    ```
    git clone https://github.com/maarten-vandeperre/developer-hub-documentation.git
    ```

2. Login into OpenShift from command prompt

3. Change directory to `developer-hub-documentation` and run the following script to install Jenkins and configured it with `BuildConfig`.
    
    ```
    ./script/script_install_jenkins.sh
    ```

    Once the installation success and running, you should be able to see the following result:

    <br>

    ![BuildConfig](/img/demo-app-pipeline-1.png)

## Configure Red Hat Developer Hub

**Adding Custom Configuration File**

To quickly setup the demo, we going to enable guest access RHDH.

1. Generate a base64 encoded string as a value. Use a unique value for each Red Hat Developer Hub instance. For example, you can use the following command to generate a key from your terminal:

    ```
    node -p 'require("crypto").randomBytes(24).toString("base64")'
    ```

    Use this base64 string in your next step for `BACKEND_SECRET`

2. Apply the following `secret` in the same project as your Developer Hub intance. Remember to change the following values according to your actual environment:

    - namespace
    - rhdh.redhat.com/backstage-name
    - BACKEND_SECRET

    ```yaml
    kind: Secret
    apiVersion: v1
    metadata:
        name: secrets-rhdh
        namespace: rhdh
    labels:
        rhdh.redhat.com/ext-config-sync: 'true'
    annotations:
        rhdh.redhat.com/backstage-name: developer-hub
    data:
        BACKEND_SECRET: WG1oL3NhblhhcUQ1aVlMdEl0eEhCV3Bhb0h5M1hVTU0=
    type: Opaque
    ```

2. Create a YAML file with the following content. Make sure to change the following values to reflect your actual setup:

    - namespace
    - rhdh.redhat.com/backstage-name
    - baseUrl
    - origin

    ```yaml
    kind: ConfigMap
    apiVersion: v1
    metadata:
    namespace: rhdh
    name: app-config-rhdh
    labels:
        rhdh.redhat.com/ext-config-sync: 'true'
    annotations:
        rhdh.redhat.com/backstage-name: developer-hub
    data:
        app-config-rhdh.yaml: |
            app:
                title: Red Hat Developer Hub
                baseUrl: https://backstage-developer-hub-rhdh.apps.sno.ocp.internal
            auth:
                environment: development
                providers:
                    guest:
                    dangerouslyAllowOutsideDevelopment: true
            backend:
                auth:
                    externalAccess:
                    - type: legacy
                        options:
                        subject: legacy-default-config
                        secret: "${BACKEND_SECRET}" 
            baseUrl: https://backstage-developer-hub-rhdh.apps.sno.ocp.internal
            cors:
                origin: https://backstage-developer-hub-rhdh.apps.sno.ocp.internal
    ```

3. Apply the custom configuration: 

    ```
    oc apply -f deployment/app-config-rhdh.yaml
    ```

4. Wait for the RHDH pod to restart and ready. Navigate to the URL and click on `Guest` button to login

    <br>

    ![RHDH Guest account](/img/rhdh-guest.png)


**Enable Jenkins Plugin**

The Jenkins plugin is pre-installed with RHDH but it is disabled by default. You need to enable it.

1. Generate base64 code string for Jenkins server URL, Jenkins username, Jenkins token using. The following show the example command to generate the Jenkins URL

    ```
    echo -n 'https://jenkins-demo-project.apps.sno.ocp.internal' | base64
    ```

    Use these generated base64 code string for the next step.

2. Update the OpenShift secret created earlier `secrets-rhdh` with the base63 code string for Jenkins:

    ```yaml
    kind: Secret
    apiVersion: v1
    metadata:
        name: secrets-rhdh
        namespace: rhdh
    labels:
        rhdh.redhat.com/ext-config-sync: 'true'
    annotations:
        rhdh.redhat.com/backstage-name: developer-hub
    data:
        BACKEND_SECRET: WG1oL3NhblhhcUQ1aVlMdEl0eEhCV3Bhb0h5M1hVTU0=
        JENKINS_TOKEN: MTE4YWI2NTQxNDY5YzU2NDUxY2UwZWZhYjcwM2Y3Njc0YQ==
        JENKINS_URL: aHR0cHM6Ly9qZW5raW5zLWRlbW8tcHJvamVjdC5hcHBzLnNuby5vY3AuaW50ZXJuYWw=
        JENKINS_USERNAME: YWRtaW4=
    type: Opaque
    ```

2. Create the following ConfigMap to enable Jenkins plugin and apply it in the same project as the RHDH. 

    > Note: 
    > - For RHDH, environmental variables in the configuration must be configured in OpenShift secret as per the previous step for `secrets-rhdh`.
    > - Make sure you change the Jenkins instance's name to `default-jenkins`. This is to ensure it match the sample `Catalog` that we are going to create in the next step

    ```yaml
    kind: ConfigMap
    apiVersion: v1
    metadata:
        name: dynamic-plugins-rhdh
        namespace: rhdh
        labels:
            rhdh.redhat.com/ext-config-sync: 'true'
        annotations:
            rhdh.redhat.com/backstage-name: developer-hub  
    data:
        dynamic-plugins.yaml: |
            includes:
                - dynamic-plugins.default.yaml
            plugins:
                - package: './dynamic-plugins/dist/backstage-plugin-jenkins-backend-dynamic'
                  disabled: false
                  pluginConfig:
                    jenkins:
                        instances:
                        - name: default-jenkins
                          baseUrl: ${JENKINS_URL}
                          username: ${JENKINS_USERNAME}
                          apiKey: ${JENKINS_TOKEN}
                - package: './dynamic-plugins/dist/backstage-plugin-jenkins'
                  disabled: false
                  pluginConfig:
                    dynamicPlugins:
                        frontend:
                            backstage.plugin-jenkins:
                                mountPoints:
                                - mountPoint: entity.page.ci/cards
                                  importName: EntityJenkinsContent
                                  config:
                                    layout:
                                        gridColumn: "1 / -1"
                                  if:
                                    allOf:
                                        - isJenkinsAvailable

    ```

3. Wait for the RHDH pod to restart and get ready. Access to the RHDH > Administrator > Plugins. Search for Jenkins. You should be able to see the plugins are enabled.

    <br>

    ![RHDH Guest account](/img/rhdh-jenkins-plugins.png)

## Create the Sample Catalog

1. Goto RHDH > Catalog and click on `Create` button

2. On the Software Template page, click on `Register Existing Component` button. Enter the following url and follow through the wizard with default settings

    ```
    https://github.com/maarten-vandeperre/dev-hub-test-demo/blob/master/catalog-info.yaml
    ```

3. Navigate to `Catalog` page, click on the `dev-hub-test-demo` that we had just created. On the detail page, click on the `CI` tab. You should be able to see that it is integrated to Jenkins with the latest build information.
     <br>

    ![RHDH Guest account](/img/rhdh-catalog-jenkins.png)

## References

- [Adding a custom application configuration file to Red Hat OpenShift Container Platform](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.3/html-single/administration_guide_for_red_hat_developer_hub/index?extIdCarryOver=true&intcmp=7013a000003SwrYAAS&sc_cid=701f2000001Css5AAC#assembly-add-custom-app-file-openshift_admin-rhdh)
- [Installing dynamic plugins with the Red Hat Developer Hub Operator ](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.3/html/installing_and_viewing_dynamic_plugins/index#proc-config-dynamic-plugins-rhdh-operator_title-plugins-rhdh-about)

