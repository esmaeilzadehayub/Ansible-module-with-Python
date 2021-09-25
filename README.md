# Write a Ansible module with Python

Ansible has a number of modules available here. But I wished to write my own. Ansible has integration with Python to enable this. I will start with a hello world module and work on from there
**When to develop an Ansible module?**
When you’re interacting with an API that has no module available. (you could use the URL module but this becomes complex)
When there is no similar Pull Request? (check the GitHub Pull Requests someone might have already started on a similar module)
When you need to simplify a complex role.

 1. In your Ansible project create the folding directory structure.

```
library
 - bugsnag.py
playbook.yml
```
2. Create a playbook file called “playbook.yml”
We are calling the “bugsnag” module which we will be developing soon.
```
---
- hosts: localhost
  tasks:
    - name: Trigger a new release
      bugsnag:
        api_key: ''
        builder_name: 'soroush'
        release_stage: 'prod/infinitypp'
        app_version: '1321358'
```
3. Developing the module.
The “bugsnag.py” file will be holding our module source code.
We will be importing two Ansible class
**AnsibleModule  (instantiate Ansible Module) –  lib/ansible/module_utils/basic.py**
**URLs (to trigger API request will be using Ansible classes )-  lib/ansible/module_utils/urls.py**
AnsibleModule  – {lib/ansible/module_utils/basic.py}
Will be using this class to instantiate our module. We will declare parameters that our modules accept based upon Bugsnag API.

URLs – {lib/ansible/module_utils/urls.py}
We will be using the fetch_url method. It is best to reuse methods that already exist in Ansible. (instead of using the requests library you could use the fetch_url method.

AnsibleModule constructor

```
module = AnsibleModule(
      argument_spec=dict(
          api_key=dict(required=True, no_log=True),
          app_version=dict(required=True),
          release_stage=dict(required=True),
          builder_name=dict(required=False),
          source_control_repository=dict(required=False),
          source_control_revision=dict(required=False)
      ),
      required_together=['source_control_repository', 'source_control_revision'],
      supports_check_mode=True
  )
  ```
  Key points:

1. If you are going pass important parameters such as API keys. set no_log=True . Ansible will not log the value.
2. sometimes, parameters depend on each other. Declare it in required_together.
3. If your module supports the dry run. Set the supports_check_mode to True

``` python
#!/usr/bin/python
from ansible.module_utils.basic import AnsibleModule
from ansible.module_utils.urls import fetch_url
def main():
    # required_together this means that these two parameters needs to be
    # specified together. If source_control_repository is specified
    # then source_control_revision has to be specified as-well
    required_together = [['source_control_repository', 'source_control_revision']]
    # we declare the parameters of our Module here
    # bugsnag needs api_key, api_version and release_stage always
    # as a result will set the required value to True. We won't be logging the API_KEY
    # for security reason. the rest are optional
    arguments = dict(
            api_key=dict(required=True, no_log=True),
            app_version=dict(required=True),
            release_stage=dict(required=True),
            builder_name=dict(required=False),
            source_control_repository=dict(required=False),
            source_control_revision=dict(required=False)
        )
    # we pass the parameters to AnsibleModule class
    # we declare that this module supports check_mode as-well
    module = AnsibleModule(
        argument_spec=arguments,
        required_together=required_together,
        supports_check_mode=True
    )
    # Bugsnag requires raw-json data
    # this is why we will create a dict called "params"
    params = {
        'apiKey': module.params['api_key'],
        'appVersion': module.params['app_version']
    }
    headers = {'Content-Type': 'application/json'}
    if module.params['builder_name']:
        params['builderName'] = module.params['builder_name']
    if module.params['release_stage']:
        params['releaseStage'] = module.params['release_stage']
    if module.params['source_control_repository'] and module.params['source_control_revision']:
        params['sourceControl'] = {}
    if module.params['source_control_repository']:
        params['sourceControl']['repository'] = module.params['source_control_repository']
    if module.params['source_control_revision']:
        params['sourceControl']['revision'] = module.params['source_control_revision']
    # Bugsnag API URL
    url = 'https://build.bugsnag.com/'
    if module.check_mode:
        module.exit_json(changed=True)
    # We will be using Ansible fetch_url method to call API requests. We don't want to
    # use Python requests as its best to use the utils provided by Ansible.
    response, info = fetch_url(module, url, module.jsonify(params), headers=headers)
    # we receive status code 200 we will return it TRUE
    if info['status'] == 200:
        module.exit_json(changed=True)
    else:
        module.fail_json(msg="unable to call API")
if __name__ == '__main__':
    main()
```
Returning output back. If our module runs successfully we need to use the module.exit_json else module.fail_json.
