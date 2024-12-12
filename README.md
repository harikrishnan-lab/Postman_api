# Postman_api
Connect to Postman without any repo in local machine.
Overview
The process involves:

Fetching Postman collections from the Postman API.
Using GitHub Actions to automate the synchronization.
Committing and pushing changes directly to the GitHub repository.
No Local repo is needed for this
1. Tools Required
GitHub Account: A GitHub account with a repository created to host the Postman collections.
Postman Account: A Postman workspace with a collection you want to sync.
Postman API Key: To access the Postman collection programmatically.
GitHub Actions: To automate the workflow for synchronization.
JSON Processor (jq): Used to format the Postman collection JSON.
2. Pre-requisites
Postman API Key:

Log in to your Postman account.
Navigate to Settings > API Keys > Generate API Key.
Copy the generated key for use in the workflow.
GitHub Repository:

Create a new repository in GitHub.
Note the repository URL (e.g., https://github.com/username/repo-name).
Store the API Key as a Secret in GitHub:

Go to your GitHub repository.
Navigate to Settings > Secrets and Variables > Actions > New Repository Secret.
Name the secret as POSTMAN_API_KEY and paste the API key value.
3. YAML Workflow Setup
Below is the YAML code to automate the sync. This workflow will:

Fetch the Postman collection from the Postman API.
Check if there are changes.
Commit and push the changes to the GitHub repository.
YAML Workflow Code
yaml
Copy code
name: Sync Postman Collection to GitHub

on:
  schedule:
    - cron: '0 * * * *'  # Runs every hour (you can customize)
  workflow_dispatch:  # Enables manual triggers

jobs:
  sync-collection:
    runs-on: ubuntu-latest

    steps:
      # Step 1: Checkout the repository
      - name: Checkout repository
        uses: actions/checkout@v3

      # Step 2: Create collections folder (if not present)
      - name: Create collections folder
        run: mkdir -p collections

      # Step 3: Fetch Postman Collection using Postman API
      - name: Fetch and Format Postman Collection
        run: |
          curl -X GET -H "X-Api-Key: ${{ secrets.POSTMAN_API_KEY }}" \
            https://api.getpostman.com/collections/<collection-id> \
            | jq '.' > collections/BasicCollection.json  # Format JSON

      # Step 4: Check for Changes in the Collection
      - name: Check for Changes
        id: check_changes
        run: |
          git config --global user.name "github-actions"
          git config --global user.email "github-actions@github.com"
          git add collections/BasicCollection.json
          git diff --exit-code || echo "changes-detected=true" >> $GITHUB_ENV

      # Step 5: Commit and Push Changes
      - name: Commit and Push Updated Collection
        if: env.changes-detected == 'true'
        run: |
          git commit -m "Updated Postman collection"
          git push origin main
4. Step-by-Step Instructions
Step 1: Create the GitHub Repository
Log in to your GitHub account.
Click New Repository.
Name the repository (e.g., postman-sync).
Choose the visibility (private or public).
Click Create Repository.
Step 2: Add API Key to GitHub Secrets
Go to your repository's Settings > Secrets and Variables > Actions.
Click New Repository Secret.
Add the following:
Name: POSTMAN_API_KEY
Value: Paste your Postman API key.
Step 3: Add the YAML Workflow
In your GitHub repository:
Go to the Actions tab.
Click New Workflow > Set up a workflow yourself.
Paste the YAML code from above into the editor.
Save the file as .github/workflows/sync_postman.yml.
Step 4: Customize the Collection ID
Replace <collection-id> in the YAML file with the ID of your Postman collection.
Find the collection ID by navigating to your collection in Postman:
Click Share Collection > Get Public Link > The ID is part of the URL (e.g., 37340056-abdf1743-bb67-4acd-a85a-c7deac122cf3).
Step 5: Run the Workflow
Manually Trigger:
Go to the Actions tab in your repository.
Select the Sync Postman Collection to GitHub workflow.
Click Run Workflow.
Scheduled Run:
The workflow will run automatically based on the schedule defined in the cron expression (e.g., every hour).
5. Validate the Setup
Navigate to the Actions tab in your repository.
Check the logs of the workflow run to confirm:
The Postman collection is fetched and saved in the collections/ directory.
Changes (if any) are committed and pushed to the repository.
6. Key Notes
Security:
The Postman API key is securely stored in GitHub Secrets and not exposed in the code.
Automatic Detection:
The workflow checks for changes in the Postman collection JSON file before committing.
Manual Trigger:
Use the workflow_dispatch event to manually run the workflow whenever needed.
7. Troubleshooting
Error: No Changes Detected
Ensure the collection ID and API key are correct.
Verify changes in Postman before triggering the workflow.
Error: Authentication Failed
Double-check the API key stored in GitHub Secrets.
Ensure the repository permissions for the GitHub Actions workflow allow writing to the branch.
========================================================================================================================================
Below is the updated workflow to create newman html report and it can be downloaded in result
name: Sync Postman Collection to GitHub

on:
  workflow_dispatch:  # Enables manual trigger
  push:
    branches:
      - main  # This can be changed to any branch you use
#  schedule:
#    - cron: '0 * * * *'  # Runs every hour (change based on your needs)

jobs:
  sync-collection:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Create collections folder (if it doesn’t exist)
        run: mkdir -p collections

      - name: Fetch and Format Postman Collection from Postman API
        run: |
          curl -X GET -H "X-Api-Key: ${{ secrets.POSTMAN_API_KEY }}" \
            https://api.getpostman.com/collections/37340056-abdf1743-bb67-4acd-a85a-c7deac122cf3 \
            | jq '.' > collections/BasicCollection.json  # Pretty-print JSON

      - name: Install Newman
        run: npm install -g newman newman-reporter-html

      - name: Run Postman Collection with HTML Report
        run: newman run collections/BasicCollection.json -r html --reporter-html-export newman/report.html

      - name: Upload Newman Report as Artifact
        uses: actions/upload-artifact@v3
        with:
          name: newman-report
          path: newman/report.html
      

      - name: Check for Changes
        id: check_changes
        run: |
          if [ -n "$(git status --porcelain)" ]; then
            echo "Changes detected."
            echo "has_changes=true" >> $GITHUB_ENV
          else
            echo "No changes detected."
            echo "has_changes=false" >> $GITHUB_ENV
          fi

      - name: Commit and Push Updated Collection
        if: env.has_changes == 'true'
        run: |
          git config --global user.name "github-actions"
          git config --global user.email "github-actions@github.com"
          git add collections/BasicCollection.json
          git commit -m "Sync updated Postman collection"
          git push origin main

      - name: Clean up untracked files in newman directory
        run: |
          git clean -fdX
