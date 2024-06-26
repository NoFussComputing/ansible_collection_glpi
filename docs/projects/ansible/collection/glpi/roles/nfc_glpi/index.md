---
title: GLPI
description: No Fuss Computings Ansible Role GLPI
date: 2023-12-11
template: project.html
about: https://gitlab.com/nofusscomputing/projects/ansible/collection/glpi
---

This playbook is designed to run Automations against GLPI. The intented usage is for a workflow wherein you create a ticket (optionally with template), create a ticket task with the notes/details of the automation and then close the ticket after recording time and marking the ticket task as closed.  
If this playbook is used with AWX / Ansible Automation Platform, artifacts are set with ticket and task creation used at the start of the workflow and the closing of the ticket at the end. As artifacts are set the workflow will pass these to each node of the flow.


## Playbook AWX / Ansible Automation Platform Template Import

This playbook includes the AWX feature where it imports the playbook as job templates in to AWX / Ansible Automation Platform. The following job templates that will be created:

- **ITIL/GLPI/Plugin/FormCreator/Delete/Form_Answer** Delete the specified form answer from the GLPI Plugin formcreator.

- **ITIL/GLPI/Plugin/FormCreator/to_JSON_Object** Fetch a Form Creator Plugin formcreator.

- **ITIL/GLPI/Ticket/Assign/Error** Assigns the ticket to the specified user on error.

- **ITIL/GLPI/Ticket/Close** Close a ticket in GLPI

- **ITIL/GLPI/Ticket/Create** Create a ticket in GLPI

- **ITIL/GLPI/Ticket/Task/Create** Create a ticket task

- **ITIL/GLPI/Ticket/Template/From_ITIL_Category** Provide an ITIL Category and this play will fetch and generate the ticket template.

On import to AWX / Ansible Automation Platform a credential type will also be created, `playbook/glpi/api` that can be used to supply the required secrets and glpi FQDN.


## Plugin - FormCreator Form to JSON Object

- Job tag is `plugin_form_fetch_json`

This task fetches a single form only that has the lowest ID. The intent is that after you have processed the form it is deleted.

This play utilizes the following variables.

``` yaml
nfc_pb_glpi_form_to_ticket: false        # Optional, boolean. Used to Create the variables required to create a ticket
                                         # with the form added to a ticket task in JSON format
nfc_pb_glpi_ticket_task_category: 0      # Mandatory*, integer. Only Mandatory if `nfc_pb_glpi_form_to_ticket=false. otherwise Optional`
                                         # This value is the ITIL Task Category from GLPI.
nfc_pb_glpi_no_log_sensitive_data: true  # Optional, boolean. used to turn `no_log` on/off for logging sensitive data
                                         # NOTE: Sensitive data will be logged. i.e. user and app token.
```

Within the playbook, this task runs first. This is by design so that a ticket forms can be gathered and in the same play have the ticket created with the form data saved within a ticket task.

!!! tip
    If you wish to create the ticket and fetch the form ensure job tags `plugin_form_fetch_json` `ticket_template_from_itil_category` `ticket_create` `ticket_task_create` `plugin_form_delete_answer` are set and no other variables outside of this task.
    
    Specifying the variables for the below tasks for creating a ticket and ticket task, the details from the form will not be used.

Prior to the play completing the following artifact/stats are set:

``` json
{
  "nfc_pb_glpi": {
    "ticket":{
      "approval": {},         // dict. generated if the form has approval configured.
    }
  },
  // The variables below this line are only created if variable 'nfc_pb_glpi_form_to_ticket=true'
  "nfc_pb_glpi_itil_category": 0,
  "nfc_pb_glpi_ticket_create": {},
  "nfc_pb_glpi_ticket_task_create": {},
  "nfc_pb_glpi_ticket_type": "request"
}

```


## Ticket Template From ITIL Category

- Job tag is `ticket_template_from_itil_category`

This play utilizes the following variables.

``` yaml
nfc_pb_glpi_host: glpi.hostname.tld      # Mandatory, string. FQDN that forms part of the url. Don't specify http|https.
nfc_pb_glpi_app_token: ''                # Mandatory, string. application token as generated from GLPI.
nfc_pb_glpi_user_token: ''               # Mandatory, string. user token as generated from GLPI.
nfc_pb_glpi_itil_category: 4             # Mandatory, integer. The ITIL Category to use from GLPI.
nfc_pb_glpi_ticket_type: request         # Mandatory, string. Choice incident|request
nfc_pb_glpi_no_log_sensitive_data: true  # Optional, boolean. used to turn `no_log` on/off for logging sensitive data
                                         # NOTE: Sensitive data will be logged. i.e. user and app token.
```

Prior to the play completing the following artifact/stats are set:

``` json
{
  "nfc_pb_glpi": {
    "ticket":{
      "template": {},         // dict. generated ticket template in the same format as would be used to send to the GLPI API 
      "mandatory_fields": []  // list. field names that are set as required from the ticket template.
    }
  }
}
```


## Create Ticket

- Job tag is `ticket_create`

!!! tip
    Aditionally you can specify tag `ticket_template_from_itil_category` to collect the ticket template from the ITIL category which will be used to create the ticket.

This play utilizes the following variables.

``` yaml
nfc_pb_glpi_host: glpi.hostname.tld      # Mandatory, string. FQDN that forms part of the url. Don't specify http|https.
nfc_pb_awx_hostname: host.domain.tld     # Optional, String. FQDN of the awx host. used to build job url for ticket description.
nfc_pb_glpi_app_token: ''                # Mandatory, string. application token as generated from GLPI.
nfc_pb_glpi_user_token: ''               # Mandatory, string. user token as generated from GLPI.
nfc_pb_glpi_ticket_create:               # Mandatory, dict. The ticket body in API Format.
  name: ""                               # Mandatory, string. Title of the ticket. If using ticket template will be appended to existing.
  content: ""                            # Mandatory, string. The ticket description. If using ticket template will be appended to existing.
  entities_id: 0                         # Mandatory, integer. entities ID for ticket to be created in.
  type: 1                                # Mandatory, choice. 1=incident|2=Request
  itilcategories_id: 0                   # Mandatory*, integer. the ticket category. ONLY mandatory for create
nfc_pb_glpi_no_log_sensitive_data: true  # Optional, boolean. used to turn `no_log` on/off for logging sensitive data
                                         # NOTE: Sensitive data will be logged. i.e. user and app token.
nfc_pb_glpi_assign_ticket: true          # Optional, boolean. Default=true. Assign the ticket to the API user.
nfc_pb_glpi_ticket_assign_error_user: 0  # Optional, integer. Default= Not specified. Assign the ticket to the specified user on error.
```

Prior to the play completing the following artifact/stats are set:

``` json
{
  "nfc_pb_glpi": {
    "ticket":{
      "assembled": {},   // dict. Assembled ticket in API format
      "create": {},      // dict. contents of 'nfc_pb_glpi_ticket_create' variable
      "items_id": []     // list. List of items to be assosiated to a tickeck on creation in API format
    }
  }
}
```

If there is an error during the play and artifact variable `nfc_automation.error` is updated to integer `1`, when this playbook next runs, on the proviso that there is an existing ticket, the ticket will be assigned to the user in variable `nfc_pb_glpi_ticket_assign_error_user`.

!!! tip
    Using the ticket assignment on error as a node within an AWX / Ansible Automation Platform workflow can be done by simply specifying `job_tags: always` and variable `nfc_pb_glpi_ticket_assign_error_user: {glpi_user_id}`. This node could then be used as a path from any node on error to notify the user.


## Create Ticket Task

- Job tag is `ticket_task_create`

!!! tip
    Aditionally you can specify tags `ticket_template_from_itil_category` and `ticket_create`to collect the ticket template from the ITIL category which will be used to create the ticket, with the ticket task being created as the last step.

This play utilizes the following variables.

``` yaml
nfc_pb_glpi_host: glpi.hostname.tld      # Mandatory, string. FQDN that forms part of the url. Don't specify http|https.
nfc_pb_glpi_app_token: ''                # Mandatory, string. application token as generated from GLPI.
nfc_pb_glpi_user_token: ''               # Mandatory, string. user token as generated from GLPI.
nfc_pb_glpi_ticket_task_create:          # Mandatory, dict. The ticket task body in API Format.
  id: 1                                  # Optional, integer. if specified will update the task. NOTE: the tickets_id must be specified
  taskcategories_id: 1                   # Mandatory, integer. task ITIL Category (ONLY Mandatory for create)
  tickets_id: 64                         # Optional, integer. Only required if not creating a ticket first. Mandatory if 'id' specified
  content: ""                           # Mandatory, string. The content of the ticket task.
  state: 1                               # Optional, choice. 0=Information|1=todo|2=Done.
  actiontime: 0                          # Optional, integer. time in seconds for task duration
  is_private: 0                          # Optional, choice 0=public|1=private
  users_id_tech: 0                       # Optional, integer. the user id of the person to assign the task.
nfc_pb_glpi_no_log_sensitive_data: true  # Optional, boolean. used to turn `no_log` on/off for logging sensitive data
                                         # NOTE: Sensitive data will be logged. i.e. user and app token.
```

Prior to the play completing the following artifact/stats are set:

``` json
{
  "nfc_pb_glpi": {
    "ticket":{
      "task": {}    // dict. ticket task in API format
    }
  }
}
```


## Plugin - Delete FormCreator Form

- Job tag is `plugin_form_delete_answer`

This Deletes a form answer from the GLPI Plugin Form Creator. Within the playbook, this task is placed to run before ticket close/solve and can safely be ran with the other tasks as listed above.

This play utilizes the following variables.

``` yaml
nfc_pb_glpi_plugin_formcreator_delete_id: 0  # Mandatory*, integer or list of integer. ID of the form answers to delete. Only required if task ran alone.
                                             # Note: This var takes precedence over artifact 'nfc_pb_glpi.plugins.form_creator.answer_id'
nfc_pb_glpi_no_log_sensitive_data: true      # Optional, boolean. used to turn `no_log` on/off for logging sensitive data
                                             # NOTE: Sensitive data will be logged. i.e. user and app token.
```

!!! tip
    If you wish to delete multiple forms you can by setting variable `nfc_pb_glpi_plugin_formcreator_delete_id` as a JSON list. i.e.
    
    ``` yaml
    nfc_pb_glpi_plugin_formcreator_delete_id: [1,2,3]
    ```


## Closing the Ticket

- Job tag is `ticket_close`

This task is completely automagic. The artifacts from the ticket and task creation runs are used for the wizardry to close the ticket and ticket task if it exists. The Automation time is added to the ticket task as well.
