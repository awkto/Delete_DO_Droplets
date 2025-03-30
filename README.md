# DigitalOcean Droplet Deletion Workflow

This GitHub Actions workflow automates the deletion of DigitalOcean droplets based on a specified tag. It includes both a dry-run mode for testing and a production mode for actual deletion.

## Features

* **Scheduled Deletion:** Runs daily at 9:00 AM UTC (7:00 PM Brisbane time) to delete droplets with a specified tag.
* **Manual Trigger:** Allows manual triggering of the workflow via `workflow_dispatch`.
* **Dry Run:** Provides a dry-run mode that lists the droplets to be deleted without actually deleting them, useful for testing.
* **Production Deletion:** Performs the actual deletion of droplets with the specified tag.
* **Secure Secret Management:** Uses GitHub Actions secrets to store the DigitalOcean API token, enhancing security.
* **Clear Logging:** Provides detailed logging of the deletion process.

## Usage

### Prerequisites

* A DigitalOcean account with an API token.
* Droplets with a specific tag that you want to delete.
* A GitHub repository.

### Setup

1.  **Create a DigitalOcean API Token:**
    * Generate a DigitalOcean API token with write access.
    * [DigitalOcean API Documentation](https://docs.digitalocean.com/reference/api/create-personal-access-token/)

2.  **Add GitHub Actions Secret:**
    * Go to your GitHub repository's "Settings" tab.
    * Click "Secrets and variables" > "Actions".
    * Click "New repository secret".
    * Name the secret `DIGITALOCEAN_ACCESS_TOKEN` and paste your DigitalOcean API token as the value.
    * Click "Add secret".

3.  **Create the Workflow File:**
    * Create a new file named `.github/workflows/delete-droplets.yml` in your repository.
    * Copy and paste the following workflow code into the file:

    ```yaml
    name: Delete DigitalOcean Droplets by Tag (Scheduled)

    on:
      schedule:
        - cron: '0 9 * * *' # Runs every day at 9 AM UTC (7 PM Brisbane time)
      workflow_dispatch:
        inputs:
          tag_name:
            description: 'The tag name to filter droplets by'
            required: true
            type: string

    jobs:
      dry_run_delete_droplets:
        runs-on: ubuntu-latest

        steps:
          - name: Checkout code (if needed)
            uses: actions/checkout@v4

          - name: Set up doctl
            uses: digitalocean/action-doctl@v2
            with:
              token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

          - name: Get droplet IDs and names with the specified tag (Dry Run)
            id: get_droplets_dry_run
            run: |
              DROPS=$(doctl compute droplet list --tag-name "${{ github.event.inputs.tag_name }}" --format json)
              if [[ -n "$DROPS" ]]; then
                echo "::group::Droplets to be deleted (Dry Run):"
                for droplet in $(echo "$DROPS"); do
                  id=$(echo "$droplet" | jq -r '.id')
                  name=$(echo "$droplet" | jq -r '.name')
                  echo "Droplet ID: $id, Name: $name"
                done
                echo "::endgroup::"
              else
                echo "No droplets found with tag: ${{ github.event.inputs.tag_name }}"
              fi

      delete_droplets:
        runs-on: ubuntu-latest
        needs: dry_run_delete_droplets

        steps:
          - name: Checkout code (if needed)
            uses: actions/checkout@v4

          - name: Set up doctl
            uses: digitalocean/action-doctl@v2
            with:
              token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

          - name: Get droplet IDs with the specified tag
            id: get_droplets
            run: |
              DROPS=$(doctl compute droplet list --tag-name "${{ github.event.inputs.tag_name }}" --format json)
              if [[ -n "$DROPS" ]]; then
                echo "::set-output name=droplet_ids::$(echo "$DROPS" | jq -r '.[].id' | tr '\n' ',')"
              else
                echo "No droplets found with tag: ${{ github.event.inputs.tag_name }}"
                exit 0
              fi

          - name: Delete droplets
            if: steps.get_droplets.outputs.droplet_ids != ''
            run: |
              DROPLET_IDS=$(echo "${{ steps.get_droplets.outputs.droplet_ids }}" | sed 's/,$//')
              echo "Deleting droplets with IDs: $DROPLET_IDS"
              for droplet_id in $(echo "$DROPLET_IDS" | tr ',' ' '); do
                doctl compute droplet delete --force $droplet_id
              done

          - name: Print completion message
            if: steps.get_droplets.outputs.droplet_ids != ''
            run: echo "Droplets with tag '${{ github.event.inputs.tag_name }}' have been deleted."
    ```

### Running the Workflow

* **Scheduled Runs:** The workflow will automatically run every day at 9:00 AM UTC.
* **Manual Runs:**
    * Go to your repository's "Actions" tab.
    * Find the "Delete DigitalOcean Droplets by Tag (Scheduled)" workflow.
    * Click "Run workflow".
    * Enter the tag name you want to use for filtering and click "Run workflow".
* **Testing:** To test, simply run the workflow. The dry_run job will output the droplets that *would* have been deleted. To test the actual deletion, ensure both jobs are enabled.

### Commenting Out Jobs

* To disable a job, place a `#` at the start of each line for that job.
* Example: Commenting out the `delete_droplets` job:

    ```yaml
    # jobs:
    #   delete_droplets:
    #     # ...
    ```

### Important Notes

* Ensure that the DigitalOcean API token has the necessary permissions to delete droplets.
* Test the workflow thoroughly in dry-run mode before enabling the actual deletion.
* Be mindful of DigitalOcean's API rate limits.
* Add error handling and logging as needed for your specific use case.
