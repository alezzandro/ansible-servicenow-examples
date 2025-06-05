# Ansible Playbook: ServiceNow Incident Creation and Update (for Ansible Automation Platform)

This Ansible playbook automates the creation of a new incident in a ServiceNow instance and subsequently posts an update (work note) to that incident. It is designed to be executed via Ansible Automation Platform (AAP) and interacts with the ServiceNow API.

## Features

* Creates a new incident in ServiceNow.
* Allows providing the incident description at runtime via AAP Job Template Surveys.
* Posts a custom update message (work note) to the newly created incident.
* Allows providing the update message at runtime via AAP Job Template Surveys.
* Includes checks for required variables.

## Prerequisites

Before running this playbook via Ansible Automation Platform, ensure you have the following:

1.  **Ansible Automation Platform (AAP) Environment**:
    * An operational AAP (Controller) instance.
    * Access to create/manage Projects, Credentials, Inventories, and Job Templates.
2.  **Execution Environment (EE)**:
    * The EE used by your Job Template must include the `servicenow.itsm` Ansible collection.
    * If using a custom EE, ensure this collection is built into it. If using a pre-built EE, verify its contents or consider synchronizing the collection from Automation Hub/Galaxy to your AAP Controller.
    * To install the collection for inclusion in an EE (if building one):
        ```bash
        ansible-galaxy collection install servicenow.itsm
        ```
3.  **ServiceNow Instance Access**:
    * A ServiceNow instance URL (e.g., `your_instance.service-now.com`).
    * Credentials (username and password) for a ServiceNow user with permissions to:
        * Create incidents (`incident` table).
        * Update incidents (specifically, add work notes).
        These credentials will be stored securely in AAP's credential store.
4.  **Playbook Repository**: The `servicenow_incident_management.yml` playbook and this `README.md` should be stored in a Git repository accessible by your AAP instance.

## Playbook Variables & AAP Configuration

The playbook utilizes several variables. When configuring this in Ansible Automation Platform, these are typically handled as follows:

* **`snow_instance`**: (Required) The hostname of your ServiceNow instance (e.g., `myinstance.service-now.com`).
    * **AAP Configuration**: Can be set as an "Extra Variable" in the Job Template, as a Survey question, or as part of a custom Credential (see below).
* **`snow_user`**: (Required) The username for connecting to ServiceNow.
    * **AAP Configuration**: This is typically part of a Credential object in AAP.
* **`snow_password`**: (Required) The password for the ServiceNow user.
    * **AAP Configuration**: This is typically part of a Credential object in AAP (stored securely).
* **`incident_description`**: (Required) A string describing the incident to be created.
    * **AAP Configuration**: Best configured as a "Survey" question in the Job Template to prompt the user at launch time.
* **`update_message`**: (Required) A string containing the message for the work note to be added to the incident.
    * **AAP Configuration**: Best configured as a "Survey" question in the Job Template to prompt the user at launch time.

**Variable names used by the playbook (ensure your AAP configuration maps to these):**
* `snow_instance_host` (for the instance, overrides `snow_instance` if set)
* `snow_username` (for the user, overrides `snow_user` if set)
* `snow_user_password` (for the password, overrides `snow_password` if set)
* `incident_description`
* `update_message`

The playbook includes default values for `snow_instance`, `snow_user`, and `snow_password` which are placeholders and **must** be overridden by AAP configurations (Credentials, Extra Variables, or Survey).

## How to Deploy and Run in Ansible Automation Platform

1.  **Create a Project in AAP**:
    * Navigate to `Projects` in your AAP Controller UI.
    * Click `Add` and create a new project.
    * **Source Control Type**: Git
    * **Source Control URL**: URL to your Git repository containing this playbook.
    * Configure Source Control Branch/Ref and Credential if your repository is private.
    * Save and sync the project.

2.  **Create a Custom ServiceNow Credential Type and Credential (Recommended for Grouping All Connection Vars)**:

    This approach allows you to group `snow_instance_host`, `snow_username`, and `snow_user_password` into a single, reusable custom credential.

    * **A. Create a Custom Credential Type**:
        * Navigate to `Credential Types` in your AAP Controller UI.
        * Click `Add`.
        * **Name**: e.g., "ServiceNow Instance & Auth"
        * **Description**: (Optional) "Custom credential for ServiceNow instance, username, and password."
        * **Input Configuration (YAML)**: This defines the fields users will see when creating a credential of this type.
            ```yaml
            fields:
              - id: snow_instance_host
                label: ServiceNow Instance Host
                type: string
                help_text: "e.g., myinstance.service-now.com"
              - id: snow_username
                label: ServiceNow Username
                type: string
              - id: snow_user_password
                label: ServiceNow Password
                type: string
                secret: true
            required:
              - snow_instance_host
              - snow_username
              - snow_user_password
            ```
        * **Injector Configuration (YAML)**: This defines how the fields are injected as extra variables for the playbook.
            ```yaml
            extra_vars:
              snow_instance_host: '{{ snow_instance_host }}'
              snow_username: '{{ snow_username }}'
              snow_user_password: '{{ snow_user_password }}'
            ```
        * Save the Custom Credential Type.

    * **B. Create a Credential Using the Custom Type**:
        * Navigate to `Credentials` in AAP.
        * Click `Add`.
        * **Name**: e.g., "My ServiceNow Dev Instance Auth"
        * **Credential Type**: Select the custom type you just created (e.g., "ServiceNow Instance & Auth").
        * Fill in the fields: `ServiceNow Instance Host`, `ServiceNow Username`, and `ServiceNow Password`.
        * Save the credential.

3.  **Create a Job Template in AAP**:
    * Navigate to `Templates` in AAP.
    * Click `Add` and choose `Add job template`.
    * **Name**: e.g., "Create and Update ServiceNow Incident"
    * **Job Type**: Run
    * **Inventory**: Select an appropriate inventory (the playbook runs `localhost`, so a simple inventory or "Prompt on launch" for one might be fine).
    * **Project**: Select the Project created in Step 1.
    * **Playbook**: Choose `servicenow_incident_management.yml` from the dropdown.
    * **Credentials**: Attach the ServiceNow credential created in Step 2 (if using standard) or Step 3B (if using custom).
    * **Variables / Extra Variables**:
        * If you are using the standard ServiceNow credential type from Step 2, you might still need to provide `snow_instance_host` here if it's not part of that credential or a survey:
            ```yaml
            # snow_instance_host: "your_instance.service-now.com" # Only if not in credential/survey
            ```
        * If using the custom credential from Step 3B, `snow_instance_host`, `snow_username`, and `snow_user_password` are already handled by the credential.
    * **Survey**: This is the recommended way to provide dynamic inputs for the incident details.
        * Click `Add Survey`.
        * **Prompt for `incident_description`**:
            * Question Name: `Incident Description`
            * Variable Name: `incident_description`
            * Answer Type: Text Area
            * Required: Yes
        * **Prompt for `update_message`**:
            * Question Name: `Update Message / Work Note`
            * Variable Name: `update_message`
            * Answer Type: Text Area
            * Required: Yes
    * Save the Job Template.

4.  **Launch the Job Template**:
    * From the `Templates` view, click the rocket icon next to your Job Template.
    * If you configured a Survey, AAP will prompt you to enter the `Incident Description` and `Update Message`.
    * Click `Launch`. The playbook will execute within AAP, creating and updating the incident in ServiceNow.

## Security Note: Managing Credentials in AAP

Ansible Automation Platform provides a secure way to store credentials. **Always use AAP's built-in credential management** for sensitive data like ServiceNow passwords. Do not hardcode them in playbooks or pass them insecurely. The playbook is written to expect `snow_instance_host`, `snow_username`, and `snow_user_password` to be supplied, which AAP's credential system (standard or custom) will inject when the job runs.

## Playbook Structure (Brief Overview)

The playbook (`servicenow_incident_management.yml`) includes:

1.  **Host and Connection Setup**: Targets `localhost`.
2.  **Variable Definitions (`vars`)**: Default placeholders, intended to be overridden by AAP configuration (Credentials, Survey, or Extra Vars).
3.  **Tasks**:
    * **Assertion Task**: Checks if essential variables (like descriptions and ServiceNow connection parameters) are properly set.
    * **Create Incident Task**: Uses `servicenow.itsm.incident` to create the incident.
    * **Update Incident Task**: Uses `servicenow.itsm.incident` to add a work note.
    * Debug tasks for output.

## Troubleshooting in AAP

* **Job Output**: Check the job output in AAP for detailed error messages.
* **Credential Issues**: Ensure the ServiceNow credential in AAP is correct, has the right permissions in ServiceNow, and is correctly mapped/applied to the Job Template. Verify the custom credential type's input and injector configurations if used.
* **Execution Environment**: Verify the `servicenow.itsm` collection is available in the EE. Logs might indicate "module not found."
* **Connectivity**: Ensure your AAP execution nodes can reach the ServiceNow instance URL (`snow_instance_host`) over HTTPS (port 443). Check firewall rules and proxy settings if applicable for your execution nodes.
* **Survey Variables**: Double-check that the "Variable Name" in your Survey matches what the playbook expects (e.g., `incident_description`, `update_message`).

## Contributing

Feel free to fork this repository, make improvements, and submit pull requests. For major changes, please open an issue first to discuss what you would like to change.
