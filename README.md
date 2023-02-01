# GitHubAction_Workflow
GitHub Actions workflows

# checks

name: "Terraform - Pre-deployment Checks"

on:
  workflow_call:
    inputs:
    secrets:
      ARM_CLIENT_ID:
        description: 'Service Principal which will have permissions to manage resources in the specified Subscription.'
        required: true
      ARM_CLIENT_SECRET:
        description: 'Password (secret) of the Service Principal.'
        required: true
      ARM_TENANT_ID:
        description: 'Tenand ID where of the Service Principal.'
        required: true

permissions:
  contents: read
  pull-requests: write

jobs:
  terraform-fmt:
    name: Terraform Format
    runs-on: ubuntu-latest
    steps:

      - name: Find Branch
        if: ${{ github.event.issue.pull_request }}
        id: branch
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PULL_URL=${{ github.event.issue.pull_request.url }}
          Accept_Header="Accept: application/vnd.github.v3+json"
          Auth_Header="Authorization: token $GITHUB_TOKEN"
          Branch=$(curl -sS -H "$Auth_Header" -H "$Accept_Header" -L $PULL_URL | jq -r ' .head.ref' )
          echo "Branch=${Branch}" >> $GITHUB_OUTPUT
      - name: Checkout - With Ref
        if: ${{ github.event.issue.pull_request }}
        uses: actions/checkout@v3
        with:
          ref: ${{ steps.branch.outputs.Branch }}

      - name: Checkout
        if: ${{ ! github.event.issue.pull_request }}
        uses: actions/checkout@v3

      - name: Terraform Format
        id: fmt
        uses: github-action-tf-fmt@v1
        with:
          Recursive: true
          Comment: true
          Github_Token: ${{ secrets.GITHUB_TOKEN }}

  terraform-validate:
    name: Terraform Validate
    runs-on: ubuntu-latest
    steps:

      - name: Find Branch
        if: ${{ github.event.issue.pull_request }}
        id: branch
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PULL_URL=${{ github.event.issue.pull_request.url }}
          Accept_Header="Accept: application/vnd.github.v3+json"
          Auth_Header="Authorization: token $GITHUB_TOKEN"
          Branch=$(curl -sS -H "$Auth_Header" -H "$Accept_Header" -L $PULL_URL | jq -r ' .head.ref' )
          echo "Branch=${Branch}" >> $GITHUB_OUTPUT
          
      - name: Checkout - With Ref
        if: ${{ github.event.issue.pull_request }}
        uses: actions/checkout@v3
        with:
          ref: ${{ steps.branch.outputs.Branch }}

      - name: Checkout
        if: ${{ ! github.event.issue.pull_request }}
        uses: actions/checkout@v3

      - name: Terraform Init
        id: init
        run: terraform init

      - name: Terraform Validate
        id: validate
        uses: github-action-tf-validate@v1
        with:
          Comment: true
          Github_Token: ${{ secrets.GITHUB_TOKEN }}

  checkov-on-code:
    name: Checkov on code
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:

      - name: Find Branch
        if: ${{ github.event.issue.pull_request }}
        id: branch
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PULL_URL=${{ github.event.issue.pull_request.url }}
          Accept_Header="Accept: application/vnd.github.v3+json"
          Auth_Header="Authorization: token $GITHUB_TOKEN"
          Branch=$(curl -sS -H "$Auth_Header" -H "$Accept_Header" -L $PULL_URL | jq -r ' .head.ref' )
          echo "Branch=${Branch}" >> $GITHUB_OUTPUT
       
      - name: Checkout - With Ref
        if: ${{ github.event.issue.pull_request }}
        uses: actions/checkout@v3
        with:
          ref: ${{ steps.branch.outputs.Branch }}

      - name: Checkout
        if: ${{ ! github.event.issue.pull_request }}
        uses: actions/checkout@v3

      - name: Download checkov config
        id: download-checkov-config
        run: |
          URL=""
          wget $URL --no-check-certificate -o /dev/null
          RESULT=${?}
          if [[ $RESULT -eq 0 ]]; then
              wget $URL --no-check-certificate -O checkov_terraform_azure.yml -o /dev/null
          else
              echo "Config file does not exist."
              exit 1
          fi
          
      - name: Checkov
        id: checkov
        uses: github-action-checkov@v1
        with:
          external_checks_repos: 
          run_all_external_checks: true
          compact: true
          quiet: true
          framework: terraform
          download_external_modules: true
          config_file: checkov_terraform_azure.yml

  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    needs: [terraform-fmt, terraform-validate, checkov-on-code]
    env:
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
    steps:

      - name: Find Branch
        if: ${{ github.event.issue.pull_request }}
        id: branch
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PULL_URL=${{ github.event.issue.pull_request.url }}
          Accept_Header="Accept: application/vnd.github.v3+json"
          Auth_Header="Authorization: token $GITHUB_TOKEN"
          Branch=$(curl -sS -H "$Auth_Header" -H "$Accept_Header" -L $PULL_URL | jq -r ' .head.ref' )
          echo "Branch=${Branch}" >> $GITHUB_OUTPUT
          
      - name: Checkout - With Ref
        if: ${{ github.event.issue.pull_request }}
        uses: actions/checkout@v3
        with:
          ref: ${{ steps.branch.outputs.Branch }}

      - name: Checkout
        if: ${{ ! github.event.issue.pull_request }}
        uses: actions/checkout@v3

      - name: Terraform Init
        id: init
        run: terraform init

      - name: Terraform Plan
        id: plan
        uses: github-action-tf-plan@v1
        with:
          GitHub_Token: ${{ secrets.GITHUB_TOKEN }}
          Out: terraform.tfplan

  checkov-on-plan:
    name: Checkov on plan
    runs-on: ubuntu-latest
    needs: [terraform-plan]
    env:
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
    steps:

      - name: Find Branch
        if: ${{ github.event.issue.pull_request }}
        id: branch
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PULL_URL=${{ github.event.issue.pull_request.url }}
          Accept_Header="Accept: application/vnd.github.v3+json"
          Auth_Header="Authorization: token $GITHUB_TOKEN"
          Branch=$(curl -sS -H "$Auth_Header" -H "$Accept_Header" -L $PULL_URL | jq -r ' .head.ref' )
          echo "Branch=${Branch}" >> $GITHUB_OUTPUT
          
      - name: Checkout - With Ref
        if: ${{ github.event.issue.pull_request }}
        uses: actions/checkout@v3
        with:
          ref: ${{ steps.branch.outputs.Branch }}

      - name: Checkout
        if: ${{ ! github.event.issue.pull_request }}
        uses: actions/checkout@v3

      - name: Terraform Init
        id: init
        run: terraform init

      - name: Terraform Plan
        id: plan
        uses: github-action-tf-plan@v1
        with:
          GitHub_Token: ${{ secrets.GITHUB_TOKEN }}
          Out: terraform.tfplan
      
      - name: Convert to JSON
        id: json
        run: |
          terraform show -json terraform.tfplan  > terraform.json 
          
      - name: Download checkov config
        id: download-checkov-config
        run: |
          URL=""
          wget $URL --no-check-certificate -o /dev/null
          RESULT=${?}
          if [[ $RESULT -eq 0 ]]; then
              wget $URL --no-check-certificate -O checkov_terraform_azure.yml -o /dev/null
          else
              echo "Config file does not exist."
              exit 1
          fi
      - name: Checkov
        id: checkov
        uses: github-action-checkov@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          file: terraform.json
          external_checks_repos: 
          run_all_external_checks: true
          compact: true
          quiet: true
          framework: terraform_plan
          download_external_modules: true
          config_file: checkov_terraform_azure.yml

  pull_request-comment:
    name: Pull Request Commment
    runs-on: ubuntu-latest
    needs: [checkov-on-plan]
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:

      - name: Add Pull Request Comment
        run: |
          ACCEPT_HEADER="Accept: application/vnd.github.v3+json"
          AUTH_HEADER="Authorization: token $GITHUB_TOKEN"
          CONTENT_HEADER="Content-Type: application/json"
          CONTENT_HEADER="Content-Type: application/json"
          if [[ "$GITHUB_EVENT_NAME" == "issue_comment" ]]; then
              PR_COMMENTS_URL=$(jq -r ".issue.comments_url" "$GITHUB_EVENT_PATH")
          else
              PR_COMMENTS_URL=$(jq -r ".pull_request.comments_url" "$GITHUB_EVENT_PATH")
          fi
          PR_COMMENT_ID=$(curl -sS -H "$AUTH_HEADER" -H "$ACCEPT_HEADER" -L "$PR_COMMENTS_URL" | jq '.[] | select(.body|test ("## Instructions")) | .id')
          if [ "$PR_COMMENT_ID" ]; then
              echo "Found existing pull request comment with instructions."
          else
            PR_COMMENT='## Instructions
            Before starting:
            - Request a pull request review.
            - When you got a pull request review, write `terraform apply` in a comment.
            This will start a workflow that will deploy the required changes to the infrastructure. 
            The result will be displayed in a comment once the task is completed.
            If the change is successful:
            - Merge your pull request.
            - Delete your branch.
            
            If the change is **NOT** successful:
            - Fix the code.
            - Push the code into the current branch. This will launch the workflow again.'
            PR_PAYLOAD=$(echo '{}' | jq --arg body "$PR_COMMENT" '.body = $body')
            curl -sS -X POST -H "$AUTH_HEADER" -H "$ACCEPT_HEADER" -H "$CONTENT_HEADER" -d "$PR_PAYLOAD" -L "$PR_COMMENTS_URL" > /dev/null
          fi

# deployment

name: Terraform Apply

on:
  workflow_call:
    inputs:
    secrets:
      ARM_CLIENT_ID:
        description: 'Service Principal which will have permissions to manage resources in the specified Subscription.'
        required: true
      ARM_CLIENT_SECRET:
        description: 'Password (secret) of the Service Principal.'
        required: true
      ARM_TENANT_ID:
        description: 'Tenand ID where of the Service Principal.'
        required: true

permissions:
  contents: read
  pull-requests: write

jobs:
  terraform-apply:
    name: Terraform Apply - Comment
    runs-on: ubuntu-latest
    env:
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
    steps:

      - name: Validate Approved
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          ACCEPT_HEADER="Accept: application/vnd.github.v3+json"
          AUTH_HEADER="Authorization: token $GITHUB_TOKEN"
          PR_URL=$(jq -r ".issue.pull_request.url" "$GITHUB_EVENT_PATH")
          PR_REVIEWS_URL="$PR_URL/reviews"
          PR_REVIEWS_ID=$(curl -sS -H "$AUTH_HEADER" -H "$ACCEPT_HEADER" -L "$PR_REVIEWS_URL" | jq '.[] | select(.state|test ("APPROVED")) | .id')
          if [ "$PR_REVIEWS_ID" ]; then
            echo "INFO     | Pull Request APPROVED."
          else
            echo "INFO     | Pull Request NOT Approved"
          fi
          
      - name: Find Branch
        if: ${{ github.event.issue.pull_request }}
        id: branch
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PULL_URL=${{ github.event.issue.pull_request.url }}
          Accept_Header="Accept: application/vnd.github.v3+json"
          Auth_Header="Authorization: token $GITHUB_TOKEN"
          Branch=$(curl -sS -H "$Auth_Header" -H "$Accept_Header" -L $PULL_URL | jq -r ' .head.ref' )
          echo "Branch=${Branch}" >> $GITHUB_OUTPUT
          
      - name: Checkout - With Ref
        if: ${{ github.event.issue.pull_request }}
        uses: actions/checkout@v3
        with:
          ref: ${{ steps.branch.outputs.Branch }}

      - name: Checkout
        if: ${{ ! github.event.issue.pull_request }}
        uses: actions/checkout@v3

      - name: Terraform Init
        id: init
        run: terraform init
        
      - name: Terraform Plan
        id: plan
        uses: github-action-tf-plan@v1
        with:
          GitHub_Token: ${{ secrets.GITHUB_TOKEN }}
          Out: terraform.tfplan

      - name: Terraform Apply
        id: apply
        uses: github-action-tf-apply@v1
        with:
          Github_Token: ${{ secrets.GITHUB_TOKEN }}
          Plan: terraform.tfplan
