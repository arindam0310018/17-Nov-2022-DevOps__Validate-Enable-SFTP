# SFTP WITH AZ DEVOPS

Greetings to my fellow Technology Advocates and Specialists.

In this Session, I will demonstrate __SFTP with Azure DevOps.__

I had the Privilege to talk on this topic in __ONE__ Azure Communities:-

| __NAME OF THE AZURE COMMUNITY__ | __TYPE OF SPEAKER SESSION__ |
| --------- | --------- |
| __Festive Tech Calendar 2022__ | __Virtual__ |


| __LIVE RECORDED SESSION:-__ |
| --------- |
| __LIVE DEMO__ was Recorded as part of my Presentation in __FESTIVE TECH CALENDAR 2022__ Forum/Platform |
| Duration of My Demo = __1 Hour 05 Mins 08 Secs__ |
| [![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/pcIVKO2dlEI/0.jpg)](https://www.youtube.com/watch?v=pcIVKO2dlEI&t=80s) |


| __THIS IS HOW IT LOOKS:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/03tp0azw78ki2bfy294h.jpg) |


| __AUTOMATION OBJECTIVE:-__ |
| --------- |
| Validate if provided Resource Group exists. If Not, Pipeline will __FAIL__. |
| Validate if Storage Account exists inside the specified Resource Group. If Not, Pipeline will __FAIL__. |
| Validate if Hierarchical Namespace is Enabled in the specified Storage Account. If Not, Pipeline will __FAIL__. |
| Validate if Key Vault exists inside the specified Resource Group. If Not, Pipeline will __FAIL__. |
| Validate if SFTP is enabled in the specified Storage Account. If No, it will enable SFTP and Proceed to Next Validation. If Yes, It will skip and and Proceed to Next Validation.  |
| Validate if SFTP Local User Home Directory Container exists. If Yes, Pipeline will __FAIL__. |
| Validate If SFTP Local User Exists. If Yes, Pipeline will __FAIL__. |
| If all of the above validation is __SUCCESSFUL__, SFTP will be Enabled or Skipped in the Storage Account (Depending upon the Status at the time), Local SSH User will be created and Password will be Generated. Finally, Local SSH Username, Password and Connection String will be stored in Key Vault. |


| __IMPORTANT NOTE:-__ |
| --------- |
The YAML Pipeline is tested on __WINDOWS BUILD AGENT__ Only!!!


| __REQUIREMENTS:-__ |
| --------- |

1. Azure Subscription.
2. Azure DevOps Organisation and Project.
3. Service Principal with Required RBAC ( __Contributor__) applied on Subscription or Resource Group(s). 
5. Azure Resource Manager Service Connection in Azure DevOps.


| __HOW DOES MY CODE PLACEHOLDER LOOKS LIKE:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yyl4imogo0q7f2czygzf.png) |


| PIPELINE CODE SNIPPET:- | 
| --------- |

| AZURE DEVOPS YAML PIPELINE (azure-pipelines-storage-account-enable-sftp-v1.0.yml):- | 
| --------- |

```
trigger:
  none

######################
#DECLARE PARAMETERS:-
######################
parameters:
- name: SUBSCRIPTIONID
  displayName: Subscription ID Details Follow Below:-
  type: string
  default: 210e66cb-55cf-424e-8daa-6cad804ab604
  values:
  - 210e66cb-55cf-424e-8daa-6cad804ab604

- name: RGNAME
  displayName: Please Provide the Resource Group Name:-
  type: object
  default: 

- name: STORAGEACCOUNTNAME
  displayName: Please Provide the Storage Account Name:-
  type: object
  default:

- name: KVNAME
  displayName: Please Provide the Keyvault Name:-
  type: object
  default: 

- name: SFTP
  displayName: Enable or Disable SFTP:-
  type: string
  default: Enable
  values:
  - Enable 

- name: SFTPUSER
  displayName: Please Provide the SFTP Username:-
  type: object
  default:

######################
#DECLARE VARIABLES:-
######################
variables:
  ServiceConnection: amcloud-cicd-service-connection
  BuildAgent: windows-latest
  Permissions: rwdlc 
  Service: blob
  SSHPasswd: true

#########################
# Declare Build Agents:-
#########################
pool:
  vmImage: $(BuildAgent)

###################
# Declare Stages:-
###################

stages:

- stage: VALIDATE_RG_STORAGE_ACCOUNT_HIERARCHICAL_NAMESPACE_AND_KV 
  jobs:
  - job: VALIDATE_RG_STORAGE_ACCOUNT_HIERARCHICAL_NAMESPACE_AND_KV 
    displayName: VALIDATE RG STORAGE ACCOUNT HIERARCHICAL_NAMESPACE & KV
    steps:
    - task: AzureCLI@2
      displayName: SET AZURE ACCOUNT
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          az --version
          az account set --subscription ${{ parameters.SUBSCRIPTIONID }}
          az account show  
    - task: AzureCLI@2
      displayName: VALIDATE RG STORAGE ACCOUNT HIERARCHICAL_NAMESPACE & RG
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |      
          $i = az group exists -n ${{ parameters.RGNAME }}
            if ($i -eq "true") {
              echo "#####################################################"
              echo "Resource Group ${{ parameters.RGNAME }} exists!!!"
              echo "#####################################################"
              $j = az storage account check-name --name ${{ parameters.STORAGEACCOUNTNAME }} --query "reason" --out tsv
                if ($j -eq "AlreadyExists") {
                  echo "###################################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} exists!!!"
                  echo "###################################################################"
                  $k = az storage account show -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --query [isHnsEnabled] --output tsv
                    if ($k -eq "true") {
                      echo "###################################################################"
                      echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} has Hierarchical Namespace Enabled!!!"
                      echo "###################################################################"
                      $l = az keyvault list --resource-group ${{ parameters.RGNAME }} --query [].name -o tsv		
                        if ($l -eq "${{ parameters.KVNAME }}") {
                          echo "###################################################################"
                          echo "Key Vault ${{ parameters.KVNAME }} exists!!!"
                          echo "###################################################################"
                        }
                        else {
                          echo "###################################################################################################"
                          echo "Key Vault ${{ parameters.KVNAME }} DOES NOT EXISTS in Resource Group ${{ parameters.RGNAME }}!!!"
                          echo "###################################################################################################"
                          exit 1
                        }
                    }  
                    else {
                      echo "#######################################################################################################################"
                      echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} DOES NOT have Hierarchical Namespace Enabled!!!!!!"
                      echo "#######################################################################################################################"
                      exit 1
                    }              
                }
                else {
                  echo "#######################################################################################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} DOES NOT EXISTS in Resource Group ${{ parameters.RGNAME }}!!!"
                  echo "#######################################################################################################################"
                  exit 1
                }
            }
            else {
              echo "#############################################################"
              echo "Resource Group ${{ parameters.RGNAME }} DOES NOT EXISTS!!!"
              echo "#############################################################"
              exit 1
            }

- stage: SFTP_ENABLE
  condition: |
     and(succeeded(),
       eq('${{ parameters.SFTP }}', 'Enable')
     ) 
  jobs:
  - job: SFTP_ENABLE 
    displayName: ENABLE SFTP & STORE CREDENTIALS IN KV
    steps:
    - task: AzureCLI@2
      displayName: ENABLE SFTP & STORE CREDENTIALS IN KV
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $i = az storage account show -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --query [isSftpEnabled] --output tsv
            if ($i -eq "false") {
              az storage account update -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --enable-sftp=true
              echo "#####################################################"
              echo "SFTP Enabled for Storage Account ${{ parameters.STORAGEACCOUNTNAME }} in the Resource Group ${{ parameters.RGNAME }}!!!"
              echo "#####################################################"
              echo "Validating if SFTP Local User Home Directory Exists!!!"
              echo "#####################################################"
              $j = az storage container exists --account-name ${{ parameters.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --name ${{ parameters.SFTPUSER }}-dir --query "exists" --out tsv 
                if ($j -ne "true") {
                  az storage container create --name ${{ parameters.SFTPUSER }}-dir --account-key $(az storage account keys list -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --query [0].value -o tsv) --account-name ${{ parameters.STORAGEACCOUNTNAME }}
                  echo "#####################################################"
                  echo "SFTP User Home Directory Container ${{ parameters.SFTPUSER }}-dir created successfully!!!"
                  echo "#####################################################"
                  echo "Validating if SFTP Local User Exists!!!"
                  echo "#####################################################"
                  $k = az storage account local-user show --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --query [name] --output tsv
                    if ($k -ne "${{ parameters.SFTPUSER }}") {
                      az storage account local-user create --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --home-directory ${{ parameters.SFTPUSER }}-dir --permission-scope permissions=$(Permissions) service=$(Service) resource-name=${{ parameters.SFTPUSER }}-dir --has-ssh-password $(SSHPasswd)
                      $sftppasswd = az storage account local-user regenerate-password --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --query [sshPassword] -o tsv
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-username --vault-name ${{ parameters.KVNAME }} --value ${{ parameters.SFTPUSER }}
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-passwd --vault-name ${{ parameters.KVNAME }} --value $sftppasswd
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-connection-string --vault-name ${{ parameters.KVNAME }} --value "${{ parameters.STORAGEACCOUNTNAME }}.${{ parameters.SFTPUSER }}@${{ parameters.STORAGEACCOUNTNAME }}.blob.core.windows.net"
                      echo "#####################################################"
                      echo "SFTP Local User ${{ parameters.SFTPUSER }} created successfully and Credentials Stored in ${{ parameters.KVNAME }}!!!"
                      echo "#####################################################"
                    }
                    else {
                      echo "#####################################################"
                      echo "SFTP Local User ${{ parameters.SFTPUSER }} already Exists!!!"
                      echo "#####################################################"
                      exit 1
                    }
                }
                else {
                  echo "#####################################################"
                  echo "SFTP User Home Directory Container ${{ parameters.SFTPUSER }}-dir already Exists!!!"
                  echo "#####################################################"
                  exit 1
                }
            }
          else {
            echo "#####################################################"
            echo "SFTP is already Enabled for Storage Account ${{ parameters.STORAGEACCOUNTNAME }} in the Resource Group ${{ parameters.RGNAME }}!!!"
            echo "#####################################################"
              $l = az storage container exists --account-name ${{ parameters.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --name ${{ parameters.SFTPUSER }}-dir --query "exists" --out tsv 
                if ($l -ne "true") {
                  az storage container create --name ${{ parameters.SFTPUSER }}-dir --account-key $(az storage account keys list -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --query [0].value -o tsv) --account-name ${{ parameters.STORAGEACCOUNTNAME }}
                  echo "#####################################################"
                  echo "SFTP User Home Directory Container ${{ parameters.SFTPUSER }}-dir created successfully!!!"
                  echo "#####################################################"
                  echo "Validating if SFTP Local User Exists!!!"
                  echo "#####################################################"
                  $m = az storage account local-user show --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --query [name] --output tsv
                    if ($m -ne "${{ parameters.SFTPUSER }}") {
                      az storage account local-user create --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --home-directory ${{ parameters.SFTPUSER }}-dir --permission-scope permissions=$(Permissions) service=$(Service) resource-name=${{ parameters.SFTPUSER }}-dir --has-ssh-password $(SSHPasswd)
                      $sftppasswd = az storage account local-user regenerate-password --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --query [sshPassword] -o tsv
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-username --vault-name ${{ parameters.KVNAME }} --value ${{ parameters.SFTPUSER }}
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-passwd --vault-name ${{ parameters.KVNAME }} --value $sftppasswd
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-connection-string --vault-name ${{ parameters.KVNAME }} --value "${{ parameters.STORAGEACCOUNTNAME }}.${{ parameters.SFTPUSER }}@${{ parameters.STORAGEACCOUNTNAME }}.blob.core.windows.net"
                      echo "#####################################################"
                      echo "SFTP Local User ${{ parameters.SFTPUSER }} created successfully and Credentials Stored in ${{ parameters.KVNAME }}!!!"
                      echo "#####################################################"
                    }
                    else {
                      echo "#####################################################"
                      echo "SFTP Local User ${{ parameters.SFTPUSER }} already Exists!!!"
                      echo "#####################################################"
                      exit 1
                    }
                }
                else {
                  echo "#####################################################"
                  echo "SFTP User Home Directory Container ${{ parameters.SFTPUSER }}-dir already Exists!!!"
                  echo "#####################################################"
                  exit 1
                }  
          }
```

Now, let me explain each part of YAML Pipeline for better understanding.

| __PART #1:-__ | 
| --------- |

| __BELOW FOLLOWS PIPELINE RUNTIME VARIABLES CODE SNIPPET:-__ | 
| --------- |

```
######################
#DECLARE PARAMETERS:-
######################
parameters:
- name: SUBSCRIPTIONID
  displayName: Subscription ID Details Follow Below:-
  type: string
  default: 210e66cb-55cf-424e-8daa-6cad804ab604
  values:
  - 210e66cb-55cf-424e-8daa-6cad804ab604

- name: RGNAME
  displayName: Please Provide the Resource Group Name:-
  type: object
  default: 

- name: STORAGEACCOUNTNAME
  displayName: Please Provide the Storage Account Name:-
  type: object
  default:

- name: KVNAME
  displayName: Please Provide the Keyvault Name:-
  type: object
  default: 

- name: SFTP
  displayName: Enable or Disable SFTP:-
  type: string
  default: Enable
  values:
  - Enable 

- name: SFTPUSER
  displayName: Please Provide the SFTP Username:-
  type: object
  default:

```

| __PART #2:-__ | 
| --------- |

| __BELOW FOLLOWS PIPELINE VARIABLES CODE SNIPPET:-__ | 
| --------- |

```
######################
#DECLARE VARIABLES:-
######################
variables:
  ServiceConnection: amcloud-cicd-service-connection
  BuildAgent: windows-latest
  Permissions: rwdlc 
  Service: blob
  SSHPasswd: true

```

| __NOTE:-__ | 
| --------- |
| Please change the values of the variables accordingly. |
| The entire YAML pipeline is build using __Runtime Parameters and Variables__. No Values are Hardcoded. |


| __PART #3:-__ | 
| --------- |

| __This is a 2 Stage Pipeline:-__ | 
| --------- |

| __STAGE #1 - VALIDATE_RG_STORAGE_ACCOUNT_HIERARCHICAL_NAMESPACE_AND_KV:-__ | 
| --------- |
| In this Stage, Pipeline will validate __Resource Group__, __Storage Account (With Hierarchal Namespace)__, and __Key Vault__. If any one of the Azure Resource is __Not Available__, Pipeline will __FAIL__ and the Next Stage will get __SKIPPED__. |

```
- stage: VALIDATE_RG_STORAGE_ACCOUNT_HIERARCHICAL_NAMESPACE_AND_KV 
  jobs:
  - job: VALIDATE_RG_STORAGE_ACCOUNT_HIERARCHICAL_NAMESPACE_AND_KV 
    displayName: VALIDATE RG STORAGE ACCOUNT HIERARCHICAL_NAMESPACE & KV
    steps:
    - task: AzureCLI@2
      displayName: SET AZURE ACCOUNT
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          az --version
          az account set --subscription ${{ parameters.SUBSCRIPTIONID }}
          az account show  
    - task: AzureCLI@2
      displayName: VALIDATE RG STORAGE ACCOUNT HIERARCHICAL_NAMESPACE & RG
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |      
          $i = az group exists -n ${{ parameters.RGNAME }}
            if ($i -eq "true") {
              echo "#####################################################"
              echo "Resource Group ${{ parameters.RGNAME }} exists!!!"
              echo "#####################################################"
              $j = az storage account check-name --name ${{ parameters.STORAGEACCOUNTNAME }} --query "reason" --out tsv
                if ($j -eq "AlreadyExists") {
                  echo "###################################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} exists!!!"
                  echo "###################################################################"
                  $k = az storage account show -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --query [isHnsEnabled] --output tsv
                    if ($k -eq "true") {
                      echo "###################################################################"
                      echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} has Hierarchical Namespace Enabled!!!"
                      echo "###################################################################"
                      $l = az keyvault list --resource-group ${{ parameters.RGNAME }} --query [].name -o tsv		
                        if ($l -eq "${{ parameters.KVNAME }}") {
                          echo "###################################################################"
                          echo "Key Vault ${{ parameters.KVNAME }} exists!!!"
                          echo "###################################################################"
                        }
                        else {
                          echo "###################################################################################################"
                          echo "Key Vault ${{ parameters.KVNAME }} DOES NOT EXISTS in Resource Group ${{ parameters.RGNAME }}!!!"
                          echo "###################################################################################################"
                          exit 1
                        }
                    }  
                    else {
                      echo "#######################################################################################################################"
                      echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} DOES NOT have Hierarchical Namespace Enabled!!!!!!"
                      echo "#######################################################################################################################"
                      exit 1
                    }              
                }
                else {
                  echo "#######################################################################################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} DOES NOT EXISTS in Resource Group ${{ parameters.RGNAME }}!!!"
                  echo "#######################################################################################################################"
                  exit 1
                }
            }
            else {
              echo "#############################################################"
              echo "Resource Group ${{ parameters.RGNAME }} DOES NOT EXISTS!!!"
              echo "#############################################################"
              exit 1
            }
```

| __STAGE #2 - SFTP_ENABLE:-__ | 
| --------- |
| In this Stage, Pipeline has Conditions in Place. |
| __Condition #1: The Previous Stage has to be Successful.__ |
| __Condition #2: The User should Select option "Enable".__ |

```
- stage: SFTP_ENABLE
  condition: |
     and(succeeded(),
       eq('${{ parameters.SFTP }}', 'Enable')
     ) 
```

| __BELOW FOLLOWS THE LOGIC DEFINED TO ENABLE SFTP IN STORAGE ACCOUNT AND STORE CREDENTIALS IN THE MENTIONED KEYVAULT:-__ | 
| --------- |
| Validate if SFTP is enabled in the specified Storage Account. If No, it will enable SFTP and Proceed to Next Validation. If Yes, It will skip and and Proceed to Next Validation. |
| Validate if SFTP Local User Home Directory Container exists. If Yes, Pipeline will FAIL. |
| Validate If SFTP Local User Exists. If Yes, Pipeline will FAIL. |
| If all of the above validation is SUCCESSFUL, SFTP will be Enabled or Skipped in the Storage Account (Depending upon the Status at the time), Local SSH User will be created and Password will be Generated. Finally, Local SSH Username, Password and Connection String will be stored in Key Vault. |

```
jobs:
  - job: SFTP_ENABLE 
    displayName: ENABLE SFTP & STORE CREDENTIALS IN KV
    steps:
    - task: AzureCLI@2
      displayName: ENABLE SFTP & STORE CREDENTIALS IN KV
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          $i = az storage account show -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --query [isSftpEnabled] --output tsv
            if ($i -eq "false") {
              az storage account update -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --enable-sftp=true
              echo "#####################################################"
              echo "SFTP Enabled for Storage Account ${{ parameters.STORAGEACCOUNTNAME }} in the Resource Group ${{ parameters.RGNAME }}!!!"
              echo "#####################################################"
              echo "Validating if SFTP Local User Home Directory Exists!!!"
              echo "#####################################################"
              $j = az storage container exists --account-name ${{ parameters.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --name ${{ parameters.SFTPUSER }}-dir --query "exists" --out tsv 
                if ($j -ne "true") {
                  az storage container create --name ${{ parameters.SFTPUSER }}-dir --account-key $(az storage account keys list -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --query [0].value -o tsv) --account-name ${{ parameters.STORAGEACCOUNTNAME }}
                  echo "#####################################################"
                  echo "SFTP User Home Directory Container ${{ parameters.SFTPUSER }}-dir created successfully!!!"
                  echo "#####################################################"
                  echo "Validating if SFTP Local User Exists!!!"
                  echo "#####################################################"
                  $k = az storage account local-user show --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --query [name] --output tsv
                    if ($k -ne "${{ parameters.SFTPUSER }}") {
                      az storage account local-user create --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --home-directory ${{ parameters.SFTPUSER }}-dir --permission-scope permissions=$(Permissions) service=$(Service) resource-name=${{ parameters.SFTPUSER }}-dir --has-ssh-password $(SSHPasswd)
                      $sftppasswd = az storage account local-user regenerate-password --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --query [sshPassword] -o tsv
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-username --vault-name ${{ parameters.KVNAME }} --value ${{ parameters.SFTPUSER }}
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-passwd --vault-name ${{ parameters.KVNAME }} --value $sftppasswd
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-connection-string --vault-name ${{ parameters.KVNAME }} --value "${{ parameters.STORAGEACCOUNTNAME }}.${{ parameters.SFTPUSER }}@${{ parameters.STORAGEACCOUNTNAME }}.blob.core.windows.net"
                      echo "#####################################################"
                      echo "SFTP Local User ${{ parameters.SFTPUSER }} created successfully and Credentials Stored in ${{ parameters.KVNAME }}!!!"
                      echo "#####################################################"
                    }
                    else {
                      echo "#####################################################"
                      echo "SFTP Local User ${{ parameters.SFTPUSER }} already Exists!!!"
                      echo "#####################################################"
                      exit 1
                    }
                }
                else {
                  echo "#####################################################"
                  echo "SFTP User Home Directory Container ${{ parameters.SFTPUSER }}-dir already Exists!!!"
                  echo "#####################################################"
                  exit 1
                }
            }
          else {
            echo "#####################################################"
            echo "SFTP is already Enabled for Storage Account ${{ parameters.STORAGEACCOUNTNAME }} in the Resource Group ${{ parameters.RGNAME }}!!!"
            echo "#####################################################"
              $l = az storage container exists --account-name ${{ parameters.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --name ${{ parameters.SFTPUSER }}-dir --query "exists" --out tsv 
                if ($l -ne "true") {
                  az storage container create --name ${{ parameters.SFTPUSER }}-dir --account-key $(az storage account keys list -n ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} --query [0].value -o tsv) --account-name ${{ parameters.STORAGEACCOUNTNAME }}
                  echo "#####################################################"
                  echo "SFTP User Home Directory Container ${{ parameters.SFTPUSER }}-dir created successfully!!!"
                  echo "#####################################################"
                  echo "Validating if SFTP Local User Exists!!!"
                  echo "#####################################################"
                  $m = az storage account local-user show --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --query [name] --output tsv
                    if ($m -ne "${{ parameters.SFTPUSER }}") {
                      az storage account local-user create --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --home-directory ${{ parameters.SFTPUSER }}-dir --permission-scope permissions=$(Permissions) service=$(Service) resource-name=${{ parameters.SFTPUSER }}-dir --has-ssh-password $(SSHPasswd)
                      $sftppasswd = az storage account local-user regenerate-password --account-name ${{ parameters.STORAGEACCOUNTNAME }} -g ${{ parameters.RGNAME }} -n ${{ parameters.SFTPUSER }} --query [sshPassword] -o tsv
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-username --vault-name ${{ parameters.KVNAME }} --value ${{ parameters.SFTPUSER }}
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-passwd --vault-name ${{ parameters.KVNAME }} --value $sftppasswd
                      az keyvault secret set --name ${{ parameters.SFTPUSER }}-connection-string --vault-name ${{ parameters.KVNAME }} --value "${{ parameters.STORAGEACCOUNTNAME }}.${{ parameters.SFTPUSER }}@${{ parameters.STORAGEACCOUNTNAME }}.blob.core.windows.net"
                      echo "#####################################################"
                      echo "SFTP Local User ${{ parameters.SFTPUSER }} created successfully and Credentials Stored in ${{ parameters.KVNAME }}!!!"
                      echo "#####################################################"
                    }
                    else {
                      echo "#####################################################"
                      echo "SFTP Local User ${{ parameters.SFTPUSER }} already Exists!!!"
                      echo "#####################################################"
                      exit 1
                    }
                }
                else {
                  echo "#####################################################"
                  echo "SFTP User Home Directory Container ${{ parameters.SFTPUSER }}-dir already Exists!!!"
                  echo "#####################################################"
                  exit 1
                }  
          }  
```

__NOW ITS TIME TO TEST !!!...__

| __TEST CASES:-__ | 
| --------- |

| __TEST CASE #1: VALIDATE RESOURCE GROUP, STORAGE ACCOUNT (WITH HIERARCHICAL NAMESPACE) AND KEY VAULT:-__ | 
| --------- |
| __DESIRED OUTPUT: PIPELINE FAILS WHEN RESOURCE GROUP DOES NOT EXISTS.__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r1pijbupjnul1hcyeqjp.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/srggfbs2eqn89jl0394w.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gcbc6h7xxcacg6i9wr6m.jpg) |
| __DESIRED OUTPUT: PIPELINE FAILS WHEN STORAGE ACCOUNT DOES NOT EXISTS.__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dsmfaqp8fa2sc9gmedpf.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/haxt0o8ozo2p82y8kar0.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2m10za0ixivy77vguk5g.jpg) |
| __DESIRED OUTPUT: PIPELINE FAILS WHEN STORAGE ACCOUNT DOES NOT HAVE HIERARCHICAL NAMESPACE ENABLED.__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6eyliwea8ky0nl1f6m4q.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/24pm0k42mi60tf7cqd5k.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/boomf8enad7mh3i0y7yj.jpg) |
| __DESIRED OUTPUT: PIPELINE FAILS WHEN KEY VAULT DOES NOT EXISTS.__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j56hrditrth94zjc1h9k.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vn3fjc1hxgslwjof7lpo.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cw0pfi67huk8i0otf6lf.jpg) |

| __TEST CASE #2: SFTP NOT ENABLED, LOCAL SSH USER AND HOME DIRECTORY CONTAINER DOES NOT EXISTS:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/knj1r7bu4os3z42eqnci.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bkzbc80ea0nlucozbruf.jpg) |
| __DESIRED OUTPUT: SFTP IS ENABLED. LOCAL USER IS CREATED WITH HOME DIRECTORY CONTAINER. PASSWORD GENERATED. ALL CREDENTIALS STORED IN KEY VAULT.__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d27gat6kaioub0y1ff2d.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ss73gev2sa9bt6egzkat.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p71ximxv1143kg69mdk5.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x12uynt9wpkz41a3ull3.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nl2lce2pze51d5x2oqx2.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0q1ku884x4g6qfs0akqz.jpg) |


| __TEST CASE #3: SFTP ALREADY ENABLED. LOCAL SSH USER DOES NOT EXISTS BUT PREVIOUSLY CREATED LOCAL SSH USER`S HOME DIRECTORY CONTAINER ALREADY EXISTS:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7frepm4u5nntp907j8gx.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0wryakfzieksgtdut5h9.jpg) |
| __DESIRED OUTPUT: PIPELINE FAILS. LOCAL SSH USER CREATED PREVIOUSLY WAS DELETED BUT HOME DIRECTORY STILL EXISTS__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vfpqz1ehjwminxv84tq1.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wlgmbk79e44eh41avqp2.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zeavd2ovc59qc458d52m.jpg) |

| __TEST CASE #4: SFTP ENABLED, LOCAL SSH USER ALREADY EXISTS:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/poup8lodc5cgh4m70hst.jpg) |
| __DESIRED OUTPUT: PIPELINE FAILS SINCE WE ARE TRYING TO CREATE LOCAL SSH USER WITH SAME NAME WHICH ALREADY EXISTS__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3ip60jmsuxm5u8l3gd3j.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/040oetmtkirtvra2tiva.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qwfsu7dv7zbwzuv031vi.jpg) |

| __TEST CASE #5: SFTP ENABLED, CREATE NEW ADDITIONAL LOCAL SSH USER AND HOME DIRECTORY CONTAINER:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x12uynt9wpkz41a3ull3.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nl2lce2pze51d5x2oqx2.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0q1ku884x4g6qfs0akqz.jpg) |
| __DESIRED OUTPUT: ADDITIONAL NEW LOCAL USER IS CREATED WITH HOME DIRECTORY CONTAINER. PASSWORD GENERATED. ALL CREDENTIALS STORED IN KEY VAULT.__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ux2x6qg6ln39pnxcxrco.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vaj22lsv7h0cueyidizq.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tblvhyifpme3rcflsjkj.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3v32tfm7ybmbwartphwq.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z5q236kjxqlvfkigse52.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/er84kht66frcrtzxfnsz.jpg) |

__Hope You Enjoyed the Session!!!__

__Stay Safe | Keep Learning | Spread Knowledge__
