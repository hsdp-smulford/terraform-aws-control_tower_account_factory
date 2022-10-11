## Disclamer
We do not have access to a lab environment to test these updates. I've run tf validate, but additional errors may surface when appliying in a live environment.

## What changed ??
```bash
>
> git diff
diff --git a/main.tf b/main.tf
index 9d4b950..af92024 100644
--- a/main.tf
+++ b/main.tf
@@ -74,15 +74,19 @@ module "aft_code_repositories" {
   aft_key_arn                                     = module.aft_account_request_framework.aft_kms_key_arn
   account_request_repo_branch                     = var.account_request_repo_branch
   account_request_repo_name                       = var.account_request_repo_name
+  account_request_repo_path                       = var.account_request_repo_path
   account_customizations_repo_name                = var.account_customizations_repo_name
+  account_customizations_repo_branch              = var.account_customizations_repo_branch
+  account_customizations_repo_path                = var.account_customizations_repo_path
   global_customizations_repo_name                 = var.global_customizations_repo_name
+  global_customizations_repo_branch               = var.global_customizations_repo_branch
+  global_customizations_repo_path                 = var.global_customizations_repo_path
+  account_provisioning_customizations_repo_name   = var.account_provisioning_customizations_repo_name
+  account_provisioning_customizations_repo_branch = var.account_provisioning_customizations_repo_branch
+  account_provisioning_customizations_repo_path   = var.account_provisioning_customizations_repo_path
   github_enterprise_url                           = var.github_enterprise_url
   vcs_provider                                    = var.vcs_provider
   terraform_distribution                          = var.terraform_distribution
-  account_provisioning_customizations_repo_name   = var.account_provisioning_customizations_repo_name
-  account_provisioning_customizations_repo_branch = var.account_provisioning_customizations_repo_branch
-  account_customizations_repo_branch              = var.account_customizations_repo_branch
-  global_customizations_repo_branch               = var.global_customizations_repo_branch
   log_group_retention                             = var.cloudwatch_log_group_retention
   global_codebuild_timeout                        = var.global_codebuild_timeout
 }
@@ -224,6 +228,7 @@ module "aft_ssm_parameters" {
   terraform_api_endpoint                                      = var.terraform_api_endpoint
   account_request_repo_branch                                 = var.account_request_repo_branch
   account_request_repo_name                                   = var.account_request_repo_name
+  account_request_repo_path                                   = var.account_request_repo_path
   vcs_provider                                                = var.vcs_provider
   aft_config_backend_primary_region                           = var.ct_home_region
   aft_config_backend_secondary_region                         = var.tf_backend_secondary_region
@@ -237,10 +242,13 @@ module "aft_ssm_parameters" {
   aft_feature_delete_default_vpcs_enabled                     = var.aft_feature_delete_default_vpcs_enabled
   account_customizations_repo_name                            = var.account_customizations_repo_name
   account_customizations_repo_branch                          = var.account_customizations_repo_branch
+  account_customizations_repo_path                            = var.account_customizations_repo_path
   global_customizations_repo_name                             = var.global_customizations_repo_name
   global_customizations_repo_branch                           = var.global_customizations_repo_branch
+  global_customizations_repo_path                             = var.global_customizations_repo_path
   account_provisioning_customizations_repo_name               = var.account_provisioning_customizations_repo_name
   account_provisioning_customizations_repo_branch             = var.account_provisioning_customizations_repo_branch
+  account_provisioning_customizations_repo_path               = var.account_provisioning_customizations_repo_path
   maximum_concurrent_customizations                           = var.maximum_concurrent_customizations
   github_enterprise_url                                       = var.github_enterprise_url
   aft_metrics_reporting                                       = var.aft_metrics_reporting
diff --git a/modules/aft-code-repositories/buildspecs/ct-aft-account-provisioning-customizations.yml b/modules/aft-code-repositories/buildspecs/ct-aft-account-provisioning-customizations.yml
index 412fe9e..e9526a4 100644
--- a/modules/aft-code-repositories/buildspecs/ct-aft-account-provisioning-customizations.yml
+++ b/modules/aft-code-repositories/buildspecs/ct-aft-account-provisioning-customizations.yml
@@ -17,6 +17,7 @@ phases:
       - AFT_EXEC_ROLE_ARN=arn:$AWS_PARTITION:iam::$AFT_MGMT_ACCOUNT:role/AWSAFTExecution
       - AFT_ADMIN_ROLE_NAME=$(aws ssm get-parameter --name /aft/resources/iam/aft-administrator-role-name | jq --raw-output ".Parameter.Value")
       - AFT_ADMIN_ROLE_ARN=arn:$AWS_PARTITION:iam::$AFT_MGMT_ACCOUNT:role/$AFT_ADMIN_ROLE_NAME
+      - AFT_REPO_PATH=$(aws ssm get-parameter --name account_provisioning_customizations_repo_path | jq --raw-output ".Parameter.Value")
       - ROLE_SESSION_NAME=$(aws ssm get-parameter --name /aft/resources/iam/aft-session-name | jq --raw-output ".Parameter.Value")
       - |
         ssh_key_parameter=$(aws ssm get-parameter --name /aft/config/aft-ssh-key --with-decryption 2> /dev/null || echo "None")
@@ -50,7 +51,7 @@ phases:
           curl -o terraform_${TF_VERSION}_linux_amd64.zip https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip
           unzip -o terraform_${TF_VERSION}_linux_amd64.zip && mv terraform /usr/bin
           terraform --version
-          cd $DEFAULT_PATH/terraform
+          cd $DEFAULT_PATH/$AFT_REPO_PATH
           for f in *.jinja; do jinja2 $f -D timestamp="$TIMESTAMP" -D tf_distribution_type=$TF_DISTRIBUTION -D region=$TF_BACKEND_REGION -D provider_region=$CT_MGMT_REGION -D bucket=$TF_S3_BUCKET -D key=$TF_S3_KEY -D dynamodb_table=$TF_DDB_TABLE -D kms_key_id=$TF_KMS_KEY_ID -D aft_admin_role_arn=$AFT_EXEC_ROLE_ARN >> ./$(basename $f .jinja).tf; done
           for f in *.tf; do echo "\n \n"; echo $f; cat $f; done
           JSON=$(aws sts assume-role --role-arn ${AFT_ADMIN_ROLE_ARN} --role-session-name ${ROLE_SESSION_NAME})
diff --git a/modules/aft-code-repositories/buildspecs/ct-aft-account-request.yml b/modules/aft-code-repositories/buildspecs/ct-aft-account-request.yml
index 998a03e..e931d05 100644
--- a/modules/aft-code-repositories/buildspecs/ct-aft-account-request.yml
+++ b/modules/aft-code-repositories/buildspecs/ct-aft-account-request.yml
@@ -17,6 +17,7 @@ phases:
       - AFT_EXEC_ROLE_ARN=arn:$AWS_PARTITION:iam::$AFT_MGMT_ACCOUNT:role/AWSAFTExecution
       - AFT_ADMIN_ROLE_NAME=$(aws ssm get-parameter --name /aft/resources/iam/aft-administrator-role-name | jq --raw-output ".Parameter.Value")
       - AFT_ADMIN_ROLE_ARN=arn:$AWS_PARTITION:iam::$AFT_MGMT_ACCOUNT:role/$AFT_ADMIN_ROLE_NAME
+      - AFT_REPO_PATH=$(aws ssm get-parameter --name /aft/config/account-request/repo-path | jq --raw-output ".Parameter.Value")
       - ROLE_SESSION_NAME=$(aws ssm get-parameter --name /aft/resources/iam/aft-session-name | jq --raw-output ".Parameter.Value")
       - |
         ssh_key_parameter=$(aws ssm get-parameter --name /aft/config/aft-ssh-key --with-decryption 2> /dev/null || echo "None")
@@ -50,7 +51,7 @@ phases:
           curl -o terraform_${TF_VERSION}_linux_amd64.zip https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip
           unzip -o terraform_${TF_VERSION}_linux_amd64.zip && mv terraform /usr/bin
           terraform --version
-          cd $DEFAULT_PATH/terraform
+          cd $DEFAULT_PATH/$AFT_REPO_PATH
           for f in *.jinja; do jinja2 $f -D timestamp="$TIMESTAMP" -D tf_distribution_type=$TF_DISTRIBUTION -D provider_region=$CT_MGMT_REGION -D region=$TF_BACKEND_REGION -D bucket=$TF_S3_BUCKET -D key=$TF_S3_KEY -D dynamodb_table=$TF_DDB_TABLE -D kms_key_id=$TF_KMS_KEY_ID -D aft_admin_role_arn=$AFT_EXEC_ROLE_ARN >> ./$(basename $f .jinja).tf; done
           for f in *.tf; do echo "\n \n"; echo $f; cat $f; done
           JSON=$(aws sts assume-role --role-arn ${AFT_ADMIN_ROLE_ARN} --role-session-name ${ROLE_SESSION_NAME})
diff --git a/modules/aft-code-repositories/variables.tf b/modules/aft-code-repositories/variables.tf
index 589616f..cb37cb4 100644
--- a/modules/aft-code-repositories/variables.tf
+++ b/modules/aft-code-repositories/variables.tf
@@ -65,6 +65,10 @@ variable "account_request_repo_branch" {
   type = string
 }

+variable "account_request_repo_path" {
+  type = string
+}
+
 variable "global_customizations_repo_name" {
   type = string
 }
@@ -73,6 +77,10 @@ variable "global_customizations_repo_branch" {
   type = string
 }

+variable "global_customizations_repo_path" {
+  type = string
+}
+
 variable "account_customizations_repo_name" {
   type = string
 }
@@ -81,6 +89,10 @@ variable "account_customizations_repo_branch" {
   type = string
 }

+variable "account_customizations_repo_path" {
+  type = string
+}
+
 variable "account_provisioning_customizations_repo_name" {
   type = string
 }
@@ -89,6 +101,10 @@ variable "account_provisioning_customizations_repo_branch" {
   type = string
 }

+variable "account_provisioning_customizations_repo_path" {
+  type = string
+}
+
 variable "global_codebuild_timeout" {
   type = number
 }
diff --git a/modules/aft-customizations/buildspecs/aft-account-customizations-terraform.yml b/modules/aft-customizations/buildspecs/aft-account-customizations-terraform.yml
index 45ef818..5709c93 100644
--- a/modules/aft-customizations/buildspecs/aft-account-customizations-terraform.yml
+++ b/modules/aft-customizations/buildspecs/aft-account-customizations-terraform.yml
@@ -19,6 +19,7 @@ phases:
       - VENDED_EXEC_ROLE_ARN=arn:$AWS_PARTITION:iam::$VENDED_ACCOUNT_ID:role/AWSAFTExecution
       - AFT_ADMIN_ROLE_NAME=$(aws ssm get-parameter --name /aft/resources/iam/aft-administrator-role-name | jq --raw-output ".Parameter.Value")
       - AFT_ADMIN_ROLE_ARN=arn:$AWS_PARTITION:iam::$AFT_MGMT_ACCOUNT:role/$AFT_ADMIN_ROLE_NAME
+      - AFT_REPO_PATH=$(aws ssm get-parameter --name account_customizations_repo_path | jq --raw-output ".Parameter.Value")
       - ROLE_SESSION_NAME=$(aws ssm get-parameter --name /aft/resources/iam/aft-session-name | jq --raw-output ".Parameter.Value")
       - |
         CUSTOMIZATION=$(aws dynamodb get-item --table-name aft-request-metadata --key "{\"id\": {\"S\": \"$VENDED_ACCOUNT_ID\"}}" --attributes-to-get "account_customizations_name" | jq --raw-output ".Item.account_customizations_name.S")
@@ -58,16 +59,16 @@ phases:

           # Install AFT Python Dependencies
           python3 -m venv $DEFAULT_PATH/aft-venv
-          $DEFAULT_PATH/aft-venv/bin/pip install pip==22.1.2
-          $DEFAULT_PATH/aft-venv/bin/pip install jinja2-cli==0.7.0 Jinja2==3.0.1 MarkupSafe==2.0.1 boto3==1.18.56 requests==2.26.0
+          $DEFAULT_PATH/$AFT_REPO_PATH/aft-venv/bin/pip install pip==22.1.2
+          $DEFAULT_PATH/$AFT_REPO_PATH/aft-venv/bin/pip install jinja2-cli==0.7.0 Jinja2==3.0.1 MarkupSafe==2.0.1 boto3==1.18.56 requests==2.26.0

           # Install API Helper Python Dependencies
-          python3 -m venv $DEFAULT_PATH/api-helpers-venv
-          $DEFAULT_PATH/api-helpers-venv/bin/pip install -r $DEFAULT_PATH/$CUSTOMIZATION/api_helpers/python/requirements.txt
+          python3 -m venv $DEFAULT_PATH/$AFT_REPO_PATH/api-helpers-venv
+          $DEFAULT_PATH/$AFT_REPO_PATH/api-helpers-venv/bin/pip install -r $DEFAULT_PATH/$CUSTOMIZATION/api_helpers/python/requirements.txt

           # Mark helper scripts as executable
-          chmod +x $DEFAULT_PATH/$CUSTOMIZATION/api_helpers/pre-api-helpers.sh
-          chmod +x $DEFAULT_PATH/$CUSTOMIZATION/api_helpers/post-api-helpers.sh
+          chmod +x $DEFAULT_PATH/$AFT_REPO_PATH/$CUSTOMIZATION/api_helpers/pre-api-helpers.sh
+          chmod +x $DEFAULT_PATH/$AFT_REPO_PATH/$CUSTOMIZATION/api_helpers/post-api-helpers.sh

           # Generate session profiles
           chmod +x $DEFAULT_PATH/aws-aft-core-framework/sources/scripts/creds.sh
@@ -82,7 +83,7 @@ phases:
         if [[ ! -z "$CUSTOMIZATION" ]]; then
           source $DEFAULT_PATH/api-helpers-venv/bin/activate
           export AWS_PROFILE=aft-target
-          $DEFAULT_PATH/$CUSTOMIZATION/api_helpers/pre-api-helpers.sh
+          $DEFAULT_PATH/$AFT_REPO_PATH/$CUSTOMIZATION/api_helpers/pre-api-helpers.sh
           unset AWS_PROFILE
         fi

@@ -92,7 +93,7 @@ phases:
       # Apply Customizations
       - |
         if [[ ! -z "$CUSTOMIZATION" ]]; then
-          source $DEFAULT_PATH/aft-venv/bin/activate
+          source $DEFAULT_PATH/$AFT_REPO_PATH/aft-venv/bin/activate
           if [ $TF_DISTRIBUTION = "oss" ]; then
             TF_BACKEND_REGION=$(aws ssm get-parameter --name "/aft/config/oss-backend/primary-region" --query "Parameter.Value" --output text)
             TF_KMS_KEY_ID=$(aws ssm get-parameter --name "/aft/config/oss-backend/kms-key-id" --query "Parameter.Value" --output text)
@@ -108,7 +109,7 @@ phases:
             mv terraform /opt/aft/bin
             /opt/aft/bin/terraform --version

-            cd $DEFAULT_PATH/$CUSTOMIZATION/terraform
+            cd $DEFAULT_PATH/$AFT_REPO_PATH/$CUSTOMIZATION/terraform
             for f in *.jinja; do jinja2 $f -D timestamp="$TIMESTAMP" -D tf_distribution_type=$TF_DISTRIBUTION -D provider_region=$CT_MGMT_REGION -D region=$TF_BACKEND_REGION -D aft_admin_role_arn=$AFT_EXEC_ROLE_ARN -D target_admin_role_arn=$VENDED_EXEC_ROLE_ARN -D bucket=$TF_S3_BUCKET -D key=$TF_S3_KEY -D dynamodb_table=$TF_DDB_TABLE -D kms_key_id=$TF_KMS_KEY_ID >> ./$(basename $f .jinja).tf; done
             for f in *.tf; do echo "\n \n"; echo $f; cat $f; done

@@ -149,7 +150,7 @@ phases:
         fi
       - |
         if [[ ! -z "$CUSTOMIZATION" ]]; then
-          source $DEFAULT_PATH/api-helpers-venv/bin/activate
+          source $DEFAULT_PATH/$AFT_REPO_PATH/api-helpers-venv/bin/activate
           export AWS_PROFILE=aft-target
-          $DEFAULT_PATH/$CUSTOMIZATION/api_helpers/post-api-helpers.sh
+          $DEFAULT_PATH/$AFT_REPO_PATH/$CUSTOMIZATION/api_helpers/post-api-helpers.sh
         fi
diff --git a/modules/aft-customizations/buildspecs/aft-global-customizations-terraform.yml b/modules/aft-customizations/buildspecs/aft-global-customizations-terraform.yml
index c2d2d4c..285e656 100644
--- a/modules/aft-customizations/buildspecs/aft-global-customizations-terraform.yml
+++ b/modules/aft-customizations/buildspecs/aft-global-customizations-terraform.yml
@@ -18,6 +18,7 @@ phases:
       - VENDED_EXEC_ROLE_ARN=arn:$AWS_PARTITION:iam::$VENDED_ACCOUNT_ID:role/AWSAFTExecution
       - AFT_ADMIN_ROLE_NAME=$(aws ssm get-parameter --name /aft/resources/iam/aft-administrator-role-name | jq --raw-output ".Parameter.Value")
       - AFT_ADMIN_ROLE_ARN=arn:$AWS_PARTITION:iam::$AFT_MGMT_ACCOUNT:role/$AFT_ADMIN_ROLE_NAME
+      - AFT_REPO_PATH=$(aws ssm get-parameter --name global_customizations_repo_path | jq --raw-output ".Parameter.Value")
       - ROLE_SESSION_NAME=$(aws ssm get-parameter --name /aft/resources/iam/aft-session-name | jq --raw-output ".Parameter.Value")

       # Configure Development SSH Key
@@ -48,31 +49,31 @@ phases:
       - $DEFAULT_PATH/aws-aft-core-framework/sources/scripts/creds.sh

       # Install AFT Python Dependencies
-      - python3 -m venv $DEFAULT_PATH/aft-venv
+      - python3 -m venv $DEFAULT_PATH/$AFT_REPO_PATH/aft-venv
       - $DEFAULT_PATH/aft-venv/bin/pip install pip==22.1.2
       - $DEFAULT_PATH/aft-venv/bin/pip install jinja2-cli==0.7.0 Jinja2==3.0.1 MarkupSafe==2.0.1 boto3==1.18.56 requests==2.26.0

       # Install API Helper Python Dependencies
-      - python3 -m venv $DEFAULT_PATH/api-helpers-venv
-      - $DEFAULT_PATH/api-helpers-venv/bin/pip install -r $DEFAULT_PATH/api_helpers/python/requirements.txt
+      - python3 -m venv $DEFAULT_PATH/$AFT_REPO_PATH/api-helpers-venv
+      - $DEFAULT_PATH/$AFT_REPO_PATH/api-helpers-venv/bin/pip install -r $DEFAULT_PATH/api_helpers/python/requirements.txt

       # Mark helper scripts as executable
-      - chmod +x $DEFAULT_PATH/api_helpers/pre-api-helpers.sh
-      - chmod +x $DEFAULT_PATH/api_helpers/post-api-helpers.sh
+      - chmod +x $DEFAULT_PATH/$AFT_REPO_PATH/api_helpers/pre-api-helpers.sh
+      - chmod +x $DEFAULT_PATH/$AFT_REPO_PATH/api_helpers/post-api-helpers.sh

   pre_build:
     on-failure: ABORT
     commands:
-      - source $DEFAULT_PATH/api-helpers-venv/bin/activate
+      - source $DEFAULT_PATH/$AFT_REPO_PATH/api-helpers-venv/bin/activate
       - export AWS_PROFILE=aft-target
-      - $DEFAULT_PATH/api_helpers/pre-api-helpers.sh
+      - $DEFAULT_PATH/$AFT_REPO_PATH/api_helpers/pre-api-helpers.sh
       - unset AWS_PROFILE

   build:
     on-failure: CONTINUE
     commands:
       # Apply customizations
-      - source $DEFAULT_PATH/aft-venv/bin/activate
+      - source $DEFAULT_PATH/$AFT_REPO_PATH/aft-venv/bin/activate
       - |
         if [ $TF_DISTRIBUTION = "oss" ]; then
           TF_BACKEND_REGION=$(aws ssm get-parameter --name "/aft/config/oss-backend/primary-region" --query "Parameter.Value" --output text)
@@ -90,11 +91,11 @@ phases:
           /opt/aft/bin/terraform --version

           # Move back to customization module
-          cd $DEFAULT_PATH/terraform
+          cd $DEFAULT_PATH/$AFT_REPO_PATH
           for f in *.jinja; do jinja2 $f -D timestamp="$TIMESTAMP" -D tf_distribution_type=$TF_DISTRIBUTION -D provider_region=$CT_MGMT_REGION -D region=$TF_BACKEND_REGION -D aft_admin_role_arn=$AFT_EXEC_ROLE_ARN -D target_admin_role_arn=$VENDED_EXEC_ROLE_ARN -D bucket=$TF_S3_BUCKET -D key=$TF_S3_KEY -D dynamodb_table=$TF_DDB_TABLE -D kms_key_id=$TF_KMS_KEY_ID >> ./$(basename $f .jinja).tf; done
           for f in *.tf; do echo "\n \n"; echo $f; cat $f; done

-          cd $DEFAULT_PATH/terraform
+          cd $DEFAULT_PATH/$AFT_REPO_PATH
           export AWS_PROFILE=aft-management-admin
           /opt/aft/bin/terraform init
           /opt/aft/bin/terraform apply --auto-approve
@@ -104,7 +105,7 @@ phases:
           TF_ENDPOINT=$(aws ssm get-parameter --name "/aft/config/terraform/api-endpoint" --query "Parameter.Value" --output text)
           TF_WORKSPACE_NAME=$VENDED_ACCOUNT_ID-aft-global-customizations
           TF_CONFIG_PATH="./temp_configuration_file.tar.gz"
-          cd $DEFAULT_PATH/terraform
+          cd $DEFAULT_PATH/$AFT_REPO_PATH
           for f in *.jinja; do jinja2 $f -D timestamp="$TIMESTAMP" -D provider_region=$CT_MGMT_REGION -D tf_distribution_type=$TF_DISTRIBUTION -D aft_admin_role_arn=$AFT_EXEC_ROLE_ARN -D target_admin_role_arn=$VENDED_EXEC_ROLE_ARN -D terraform_org_name=$TF_ORG_NAME -D terraform_workspace_name=$TF_WORKSPACE_NAME  >> ./$(basename $f .jinja).tf; done
           for f in *.tf; do echo "\n \n"; echo $f; cat $f; done
           cd $DEFAULT_PATH
@@ -123,6 +124,6 @@ phases:
         if [[ $CODEBUILD_BUILD_SUCCEEDING == 0 ]]; then
           exit 1
         fi
-      - source $DEFAULT_PATH/api-helpers-venv/bin/activate
+      - source $DEFAULT_PATH/$AFT_REPO_PATH/api-helpers-venv/bin/activate
       - export AWS_PROFILE=aft-target
-      - $DEFAULT_PATH/api_helpers/post-api-helpers.sh
+      - $DEFAULT_PATH/$AFT_REPO_PATH/api_helpers/post-api-helpers.sh
diff --git a/modules/aft-ssm-parameters/ssm.tf b/modules/aft-ssm-parameters/ssm.tf
index 2800f21..5335e38 100644
--- a/modules/aft-ssm-parameters/ssm.tf
+++ b/modules/aft-ssm-parameters/ssm.tf
@@ -303,6 +303,12 @@ resource "aws_ssm_parameter" "account_request_repo_branch" {
   value = var.account_request_repo_branch
 }

+resource "aws_ssm_parameter" "account_request_repo_path" {
+  name  = "/aft/config/account-request/repo-path"
+  type  = "String"
+  value = var.account_request_repo_path
+}
+
 resource "aws_ssm_parameter" "global_customizations_repo_name" {
   name  = "/aft/config/global-customizations/repo-name"
   type  = "String"
@@ -315,6 +321,12 @@ resource "aws_ssm_parameter" "global_customizations_repo_branch" {
   value = var.global_customizations_repo_branch
 }

+resource "aws_ssm_parameter" "global_customizations_repo_path" {
+  name  = "/aft/config/global-customizations/repo-path"
+  type  = "String"
+  value = var.global_customizations_repo_path
+}
+
 resource "aws_ssm_parameter" "account_customizations_repo_name" {
   name  = "/aft/config/account-customizations/repo-name"
   type  = "String"
@@ -327,6 +339,12 @@ resource "aws_ssm_parameter" "account_customizations_repo_branch" {
   value = var.account_customizations_repo_branch
 }

+resource "aws_ssm_parameter" "account_customizations_repo_path" {
+  name  = "/aft/config/account-customizations/repo-path"
+  type  = "String"
+  value = var.account_customizations_repo_path
+}
+
 resource "aws_ssm_parameter" "account_provisioning_customizations_repo_name" {
   name  = "/aft/config/account-provisioning-customizations/repo-name"
   type  = "String"
@@ -339,6 +357,12 @@ resource "aws_ssm_parameter" "account_provisioning_customizations_repo_branch" {
   value = var.account_provisioning_customizations_repo_branch
 }

+resource "aws_ssm_parameter" "account_provisioning_customizations_repo_path" {
+  name  = "/aft/config/account-provisioning-customizations/repo-path"
+  type  = "String"
+  value = var.account_provisioning_customizations_repo_path
+}
+
 resource "aws_ssm_parameter" "codestar_connection_arn" {
   name  = "/aft/config/vcs/codestar-connection-arn"
   type  = "String"
diff --git a/modules/aft-ssm-parameters/variables.tf b/modules/aft-ssm-parameters/variables.tf
index 3530b29..72b46e7 100644
--- a/modules/aft-ssm-parameters/variables.tf
+++ b/modules/aft-ssm-parameters/variables.tf
@@ -142,6 +142,10 @@ variable "account_request_repo_branch" {
   type = string
 }

+variable "account_request_repo_path" {
+  type = string
+}
+
 variable "account_provisioning_customizations_repo_name" {
   type = string
 }
@@ -150,6 +154,10 @@ variable "account_provisioning_customizations_repo_branch" {
   type = string
 }

+variable "account_provisioning_customizations_repo_path" {
+  type = string
+}
+
 variable "terraform_org_name" {
   type = string
 }
@@ -218,6 +226,10 @@ variable "global_customizations_repo_branch" {
   type = string
 }

+variable "global_customizations_repo_path" {
+  type = string
+}
+
 variable "account_customizations_repo_name" {
   type = string
 }
@@ -226,6 +238,10 @@ variable "account_customizations_repo_branch" {
   type = string
 }

+variable "account_customizations_repo_path" {
+  type = string
+}
+
 variable "codestar_connection_arn" {
   type = string
 }
diff --git a/variables.tf b/variables.tf
index be95d13..34f81bb 100644
--- a/variables.tf
+++ b/variables.tf
@@ -184,6 +184,16 @@ variable "account_request_repo_branch" {
   }
 }

+variable "account_request_repo_path" {
+  description = "The root path to find the terraform and scripts for account customizations"
+  type        = string
+  default     = "terraform"
+  validation {
+    condition     = length(var.account_request_repo_path) > 0
+    error_message = "Variable var: account_request_repo_path cannot be empty."
+  }
+}
+
 variable "global_customizations_repo_name" {
   description = "Repository name for the global customization files. For non-CodeCommit repos, name should be in the format of Org/Repo"
   type        = string
@@ -204,6 +214,16 @@ variable "global_customizations_repo_branch" {
   }
 }

+variable "global_customizations_repo_path" {
+  description = "The root path to find the terraform and scripts for account customizations"
+  type        = string
+  default     = "terraform"
+  validation {
+    condition     = length(var.global_customizations_repo_path) > 0
+    error_message = "Variable var: global_customizations_repo_path cannot be empty."
+  }
+}
+
 variable "account_customizations_repo_name" {
   description = "Repository name for the account customizations files. For non-CodeCommit repos, name should be in the format of Org/Repo"
   type        = string
@@ -224,6 +244,16 @@ variable "account_customizations_repo_branch" {
   }
 }

+variable "account_customizations_repo_path" {
+  description = "The root path to find the terraform and scripts for account customizations"
+  type        = string
+  default     = "terraform"
+  validation {
+    condition     = length(var.account_customizations_repo_path) > 0
+    error_message = "Variable var: account_customizations_repo_path cannot be empty."
+  }
+}
+
 variable "account_provisioning_customizations_repo_name" {
   description = "Repository name for the account provisioning customizations files. For non-CodeCommit repos, name should be in the format of Org/Repo"
   type        = string
@@ -244,6 +274,16 @@ variable "account_provisioning_customizations_repo_branch" {
   }
 }

+variable "account_provisioning_customizations_repo_path" {
+  description = "The root path to find the terraform and scripts for account customizations"
+  type        = string
+  default     = "terraform"
+  validation {
+    condition     = length(var.account_provisioning_customizations_repo_path) > 0
+    error_message = "Variable var: account_provisioning_customizations_repo_path cannot be empty."
+  }
+}
+
 #########################################
 # AFT Terraform Distribution Variables
 #########################################
>
```
