# tapas-tasks adapter logic and explanation 

'''
├── in
│   ├── formats
│   │   ├── TaskJsonPatchRepresentation.java
│   │   └── TaskJsonRepresentation.java
│   ├── messaging
│   │   ├── UnknownEventException.java
│   │   └── http
│   │       ├── TaskEventHttpDispatcher.java
│   │       ├── TaskEventListener.java
│   │       ├── TaskExecutedEventListenerHttpAdapter.java
│   │       └── TaskStartedEventListenerHttpAdapter.java
│   └── web
│       ├── AddNewTaskToTaskListWebController.java
│       └── RetrieveTaskFromTaskListWebController.java
└── out
    ├── persistence
    │   └── mongodb
    │       ├── MongoTaskDocument.java
    │       ├── TaskMapper.java
    │       ├── TaskPersistenceAdapter.java
    │       └── TaskRepository.java
    └── web
        └── PublishNewTaskAddedEventWebAdapter.java
'''

In your Spring Boot application, you have organized your code in a well-structured manner adhering to the Hexagonal Architecture (also known as Ports and Adapters Architecture). Here's a brief explanation of the classes within the `http` package you've provided, focusing on their roles and interactions:

### 1. `TaskEventHttpDispatcher`:
This class serves as a REST Controller to handle HTTP PATCH requests aimed at updating the state of Task resources. It utilizes the JSON Patch format to process requests, which can contain multiple operations (e.g., add, remove, replace) to update various aspects of a task's representation. Here are the key points:
- The `dispatchTaskEvents` method is mapped to handle PATCH requests on the `/tasks/{taskId}` endpoint.
- The method first validates the JSON Patch syntax, then identifies and routes events based on the patch operations to appropriate event listeners (`TaskStartedEventListenerHttpAdapter` or `TaskExecutedEventListenerHttpAdapter`).
- It generates a response with appropriate headers and status based on the processing outcome.

### 2. `TaskEventListener`:
This is an abstract class which serves as a blueprint for handling task events received via HTTP PATCH requests. It declares an abstract method `handleTaskEvent` that must be implemented by subclasses to process the events.

### 3. `TaskExecutedEventListenerHttpAdapter`:
- This class extends `TaskEventListener` and implements `handleTaskEvent` to process task executed events.
- It utilizes `TaskJsonPatchRepresentation` to extract output data from the JSON Patch payload, if any.
- It then creates a `TaskExecutedEvent` object and invokes a `TaskExecutedHandler` to handle the event, returning a boolean indicating the handling outcome.

### 4. `TaskStartedEventListenerHttpAdapter`:
- Similar to `TaskExecutedEventListenerHttpAdapter`, this class extends `TaskEventListener` and implements `handleTaskEvent` to process task started events.
- It constructs a `TaskStartedEvent` object and invokes a `TaskStartedHandler` to handle the event, returning a boolean indicating the handling outcome.

### General Flow:
1. An HTTP PATCH request is sent to update a task's status.
2. `TaskEventHttpDispatcher` receives the request, validates the JSON Patch syntax, identifies the event type, and routes the event to the appropriate event listener.
3. The event listener (`TaskExecutedEventListenerHttpAdapter` or `TaskStartedEventListenerHttpAdapter`) processes the event, potentially invoking application layer handlers (`TaskExecutedHandler` or `TaskStartedHandler`) to perform domain-specific operations.
4. A response is generated and sent back to the client, indicating the outcome of the request.



# Copying this structure to tapas-executor-pool:

Creating a similar communication mechanism in `tapas-executor-pool` to receive tasks from `tapas-tasks` can be done by adhering to the principles of Hexagonal Architecture as you've done in `tapas-tasks`. Here’s how you could implement the `adapter` folder structure and classes to facilitate communication between the two modules using a RESTful HTTP approach:

### Folder Structure:
```plaintext
├── adapter
│   ├── in
│   │   ├── messaging
│   │   │   └── http
│   │   │       ├── TaskReceivedEventListenerHttpAdapter.java
│   │   │       └── TaskEventHttpDispatcher.java
│   │   └── web
│   │       ├── AssignExecutorToTaskWebController.java
│   │       └── ...
│   └── out
│       └── ...
```
### HTTP Adapter Classes:

1. **TaskEventHttpDispatcher.java**:
```java
package ch.unisg.tapasexecutorpool.executorpool.adapter.in.messaging.http;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TaskEventHttpDispatcher {

    @Autowired
    private TaskReceivedEventListenerHttpAdapter taskReceivedListener;

    @PostMapping(path = "/taskEvents")
    public void dispatchTaskEvents(@RequestBody TaskJsonRepresentation taskJsonRepresentation) {
        taskReceivedListener.handleTaskReceivedEvent(taskJsonRepresentation);
    }
}
```

2. **TaskReceivedEventListenerHttpAdapter.java**:
```java
package ch.unisg.tapasexecutorpool.executorpool.adapter.in.messaging.http;

import ch.unisg.tapasexecutorpool.executorpool.application.port.in.AssignExecutorToTaskUseCase;
import ch.unisg.tapasexecutorpool.executorpool.application.port.in.AssignExecutorToTaskCommand;
import ch.unisg.tapasexecutorpool.executorpool.domain.Task;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class TaskReceivedEventListenerHttpAdapter {

    @Autowired
    private AssignExecutorToTaskUseCase assignExecutorToTaskUseCase;

    public void handleTaskReceivedEvent(TaskJsonRepresentation taskJsonRepresentation) {
        Task task = mapToTask(taskJsonRepresentation);
        AssignExecutorToTaskCommand command = new AssignExecutorToTaskCommand(task);
        assignExecutorToTaskUseCase.assignExecutorToTask(command);
    }

    private Task mapToTask(TaskJsonRepresentation taskJsonRepresentation) {
        // Implement the mapping logic from TaskJsonRepresentation to Task
        // ...
    }
}
```

3. **TaskJsonRepresentation.java** (Create a new class to represent the JSON format of the task):
```java
package ch.unisg.tapasexecutorpool.executorpool.adapter.in.messaging.http;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class TaskJsonRepresentation {
    private String taskId;
    private String taskName;
    private String taskType;
    private String taskStatus;
    private String inputData;
    private String outputData;
    // other fields...
}
```

Explanation:
- `TaskEventHttpDispatcher`: This class is a Spring Rest Controller that listens for incoming POST requests on the `/taskEvents` endpoint. When a request is received, it passes the task data to a `TaskReceivedEventListenerHttpAdapter`.
  
- `TaskReceivedEventListenerHttpAdapter`: This class handles the event of a task being received. It maps the JSON representation of the task to a domain entity and triggers the use case to assign an executor to the task.
  
- `TaskJsonRepresentation`: This is a data transfer object that represents the JSON structure of a task.

In the `tapas-tasks` module, you would then need to create an HTTP client that sends a POST request to the `/taskEvents` endpoint of the `tapas-executor-pool` whenever a new task is added to the task list.

Please adjust the class names, package names, and other details to fit your project's naming conventions and structure.