# Ansible-quay-operator

manage quay with ansible operator in openshift

## Prerequisites

operator-sdk cli : https://sdk.operatorframework.io/docs/installation/
An accessible image registry for operator images (ex. hub.docker.com or quay.io)

## Create a new project 

Create a new project with SDK cli specifying the ansible plugin.

```
mkdir quay-operator
cd quay-operator
operator-sdk init --plugins=ansible --domain example.com
```

``` 
operator-sdk create api --group cache --version v1alpha1 --kind registry-manager-operator --generate-role 
```

## Create the ansible tasks to manage quay creation and deletion of repository in roles/quay/tasks/main.yml

```
--
# tasks file for QuayRegister

- name: check if image exist
  uri:
    url: "https://quay.io/api/v1/repository/{{ namespace }}/{{ repository }}/{{ image }}"
    method: GET
    headers:
      Authorization: "Bearer {{ token }}"
    body_format: json
    status_code:
      - 200
      - 404
  register: repostate

- name: create-repo
  uri:
    url: https://quay.io/api/v1/repository
    method: POST
    headers:
      Authorization: "Bearer {{ token }}"
    body_format: json
    body:
      repo_kind: "{{ image }}"
      namespace: "{{ namespace }}"
      visibility: "{{ visibility }}"
      repository: "{{ repository }}"
      description: blablabla
    status_code:
      - 201
  when:
    - repostate.status == 404


- name: delete-repo
  uri:
    url: "https://quay.io/api/v1/repository/{{ namespace }}/{{ repository }}"
    method: DELETE
    headers:
      Authorization: "Bearer {{ token }}"
    body_format: json
    status_code:
      - 204
  when:
    - repostate.status == 200
    - state == "absent"

```

Then update the quayregister sample in config/samples/cache_v1alpha1_memcached.yaml

```
apiVersion: cache.example2.com/v1alpha1
kind: QuayRegister
metadata:
  name: quayregister-sample
spec:
  token: <your-quay-token>
  namespace : <your-quay-namespace>
  visibility: public
  repository: <your-quay-repositories>
  image: image
```

Finally update the watches.yaml to delete repositories when you delete your ressources

```
---
# Use the 'create api' subcommand to add watches to this file.
- version: v1alpha1
  group: cache.example2.com
  kind: QuayRegister
  role: quayregister
  finalizer:
    name: cache.example2.com/finalizer
    vars:
      state: absent                     
```

## Run the operator

Define your image registry

``` export IMG=quay.io/<your-quay-namespace>/controller:latest ```

Build and push your image

``` make docker-build docker-push ```

Run as a deployment inside the cluster 

``` make deploy ```

You can verify the quay-register operator is up and running with the following command :

``` oc get deployment -n quay-operator-system ```

## Create a quayregister CR

update the following file by changing the repository spec with your repo name config/samples/cache_v1alpha1_memcached.yaml. Then create the CR with:

``` oc apply -f config/samples/cache_v1alpha1_memcached.yaml ```


## Delete repository
To delete your repository, delete the CR:

``` oc delete -f config/samples/cache_v1alpha1_memcached.yaml ```







