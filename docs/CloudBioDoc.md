### Updates
| Dates | Notes |
| -------- | -------- |
| 2023/08/01 | First draft finished|

# Demo文档
## Table of Contents
- [TechStack](#TechStack)
  - [React](#React)
  - [Golang](#Golang)
  - [Kubernetes](#Kubernetes)
  - [Database](#Database)
  - [MessageQueue](#MessageQueue)
- [GeneralWorkflow](#GeneralWorkflow)
  - [FileUpload](#FileUpload)
  - [WorkflowInit](#WorkflowInit)
  - [ConfigYaml](#ConfigYaml)
- [Monitoring](#Monitoring)
  - [AlertManager](#AlertManager)
  - [Exporter](#Exporter)

## TechStack
Record the tools used for this project, e.g. version, reason
## GeneralWorkflow
The backend will consist of four services: Middleware manager (mwm), File transfer manager (ftm), Workflow manager (wfm), and Tool worker manager (twm). These services will be represented as pods with three containers, and there will be replication for each pod.
### FileUpload
```
Handel file transfer
```
  1. FileTransferManager (ftm) server
      - Users can upload files to the server. 
      - The frontend and Beego backend handle the file download to the local system. 
      - After performing necessary checks and validations, the frontend sends an upload request to the ftm's endpoint. 
      - The file is then uploaded to an s3 bucket.
### Workflow
```
Init workflow based on yaml file, the yaml file is customizable (formed based on frontend submit form), it defines the process logic of workflow pipeline and contins the required parameters of each tool within the pipeline.
```
1. Frontend Validation and File Upload
    - Frontend validates the submitted form, including whether the file upload was `successful`, `ongoing`, or `failed`, as well as other required task-related fields.
    - If a file was uploaded, a file upload request is sent to `mwm`, which downloads the file to the local system (processing backend node).
    - Once the download is complete, a request is sent to `ftm`, which takes over to verify the file and upload it to the `s3 bucket`.
    - The status of the file upload is recorded in `etcd`.
2. Task Information Generation and RabbitMQ Queue
    - `Mwm` obtains the task information and attempts to acquire a lock in `etcd` (ignores if another backend is already processing the task).
    - It then generates the main workflow table in the `MySQL` database and adds the task to the `RabbitMQ (workflow) queue`.
    - `Mwm` opens a `grpc server` to listen to messages from `wfm` regarding task updates, placing them in a channel.
    - As soon as any message is received in the channel, it checks the task's status and plans accordingly.
3. Workflow Manager and Kubernetes Job
    - `Wfm` listens to the `RabbitMQ (workflow) queue`.
    - Since there are two wfms, the Message Acknowledgment mechanism ensures each message is consumed only once.
    - Upon receiving a message, `wfm` retrieves detailed task information from the main table and updates the status field to `processing`.
    - It then creates a Kubernetes job (named work manager) based on the provided information.
    - Note that this job will have a single main process (one container) responsible for managing the tools required for the current task.
    - Work manager is also responsible for listening to tool run status through its grpc server.
4. Tool Management and Configuration
    - After starting, `work manager` organizes the tool execution sequence and parameters based on the task's configuration file.
    - The configuration file includes information on the selected workflow or a custom-defined one, including the specific tools, their execution order, dependencies, and parameters (in YAML format).
    - `Work manager` updates the tool information and status in the tool's secondary table based on the workflow main table.
    - It then sends tool execution messages to the RabbitMQ (tool) queue.
    - `Work manager` must maintain an active grpc server to listen to tool run statuses.
5. Tool Execution and Monitoring
    - `Tool worker manager (twm)` listens to the `RabbitMQ (tool) queue` using the Message Acknowledgment mechanism to ensure each message is consumed only once.
    - When `twm` receives a task, it creates an individual Kubernetes job (pod) for each task.
    - Each pod will run a tool job container, along with a sidecar to monitor the tool job's execution status.
    - The sidecar communicates the status to work manager via grpc.
    - Currently, if no tool start message is received within 2 minutes of task delivery, it is considered a delivery failure.
    - If a tool start message is received, `twm` expects a tool keepalive message from the sidecar every 30 seconds.
    - If no keepalive message is received, the tool run is considered a failure.
6. Task Completion and Reporting
    - Based on the received tool statuses, `twm` determines whether to rerun the tool or send a task failure message to `wfm`.
    - Reruns may occur due to recoverable issues such as `memory limit` or `no space left on device`, which can be resolved by modifying resources for successful tool execution.
### ConfigYaml
```
Example need further adjustment 
```
  1. Yaml for each tool
```yaml
  name: tool_name
  cpu: ?
  mem: ?G
  description: |
    the tool general description
  output: |
    the desired output for the tool
  env:
    - Any environmental variable need to be passed to the container
  options:
    - name: ?
      desc: ?
      type: ?
      required: True/False
  cmds:
    - shell commands to run this tool
  rely:
    - used to define the process sequence of pipeline, could be multiple
    - tool1;tool2
```

## Monitoring
The monitoring of each workflow, tool and k8s cluster resources
### AlertManager
```
Send alert when major error captured
```
1. DingTalk Robot or other? 
    - Workflow error.
    - CronTab misson, monitoring pod numbers, resources usage over a period of time.
### Exporter
```
The general system level logs and logs for Middleware manager (mwm), File transfer manager (ftm), Workflow manager (wfm), and Tool worker manager (twm) will be collected via prometheus, other required metrics might need to be dsigned later.
```
1. Loki for log aggregation of major server running in pods.
2. Grafana for monitoring resource usage, need a dashboard, can can be intergated to our backend UI platform (maybe)