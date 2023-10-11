# Notes on MongoDB Development for Executor Pool

## Current MongoDB Implementation in Tapas-Tasks

### Current Project Structure:

```
.
├── persistence
│   └── mongodb
│       ├── MongoTaskDocument.java
│       ├── TaskMapper.java
│       ├── TaskPersistenceAdapter.java
│       └── TaskRepository.java
└── web
    └── PublishNewTaskAddedEventWebAdapter.java

```

The code illustrate a modular setup for managing tasks in a Spring-based Java application using MongoDB as the persistence layer. Here's an in-depth explanation of the parts and how they interact:

1. **Domain Layer (Task, TaskList)**:
   - The `Task` and `TaskList` classes represent domain entities in the application. They encapsulate data and behavior related to tasks and lists of tasks respectively.

2. **Persistence Layer**:
   - This layer is situated under the `persistence/mongodb` directory and consists of several components that facilitate the mapping between domain entities and MongoDB documents, as well as the interaction with the MongoDB database.

    a. **MongoTaskDocument**:
        - This class acts as a Data Transfer Object (DTO) for interacting with MongoDB. It represents the schema of the `tasks` collection in MongoDB with fields such as `taskId`, `taskName`, `taskType`, `taskStatus`, `inputData`, `outputData`, and `taskListName`.

    b. **TaskMapper**:
        - The `TaskMapper` class provides methods to convert between `Task` domain entities and `MongoTaskDocument` DTOs. This facilitates the translation between the domain-centric and database-centric representations of tasks.

    c. **TaskRepository**:
        - This interface extends `MongoRepository` from Spring Data MongoDB, defining methods to interact with the `tasks` collection in MongoDB. It provides methods to find a task by its ID and task list name, and to find all tasks associated with a particular task list name.

    d. **TaskPersistenceAdapter**:
        - This class implements `AddTaskPort` and `LoadTaskPort` interfaces, defining methods to add a task to and load a task from the database. It utilizes `TaskRepository` and `TaskMapper` to interact with the MongoDB database.

3. **Web Layer**:
    - The `web` directory contains a component, `PublishNewTaskAddedEventWebAdapter`, which implements `NewTaskAddedEventPort` interface. This component is responsible for publishing events when a new task is added.

    a. **PublishNewTaskAddedEventWebAdapter**:
        - This class is used to publish a `NewTaskAddedEvent` to an external service via a REST call. The method `publishNewTaskAddedEvent` constructs a HTTP POST request and sends it to a predefined server URL (`http://127.0.0.1:8082/roster/newtask/`). The event data includes the location of the new task and the name of the task list it belongs to.

4. **Overall Interaction**:
   - When a new task is created, it would be passed to the `TaskPersistenceAdapter` via the `addTask` method. This method translates the `Task` domain entity to a `MongoTaskDocument` DTO using `TaskMapper`, and persists it to MongoDB using `TaskRepository`.
   - Similarly, tasks can be loaded from the database using the `loadTask` method in `TaskPersistenceAdapter`, which again leverages `TaskRepository` and `TaskMapper`.
   - On the addition of a new task, an event can be published to an external service using the `PublishNewTaskAddedEventWebAdapter`, notifying it of the new task's addition.

This setup illustrates a well-structured approach to managing tasks, where domain logic is kept separate from persistence logic, and interactions with external services are handled in a dedicated layer.

### In-Depth Explanation of Code:

#### out.persistence.mongodb.MongoTaskDocument

This code defines a Java class `MongoTaskDocument` which serves as a data model for MongoDB documents within a Spring application. Here's a breakdown of its components:

1. **Package Declaration**:
   ```java
   package ch.unisg.tapastasks.tasks.adapter.out.persistence.mongodb;
   ```
   - The code is declared to be part of the package `ch.unisg.tapastasks.tasks.adapter.out.persistence.mongodb`.

2. **Imports**:
   ```java
   import lombok.Data;
   import org.springframework.data.annotation.Id;
   import org.springframework.data.mongodb.core.mapping.Document;
   ```
   - The class imports three annotations:
     - `@Data` from the Lombok library, which is a shortcut for generating boilerplate code such as getters, setters, `equals`, `hashCode`, and `toString` methods.
     - `@Id` from the Spring Data library, which denotes the field to be used as the identifier for the MongoDB document.
     - `@Document` from the Spring Data MongoDB library, which marks this class as a domain object to be persisted to MongoDB.

3. **Class Declaration**:
   ```java
   @Data
   @Document(collection = "tasks")
   public class MongoTaskDocument {
   ```
   - The `@Data` annotation tells Lombok to generate boilerplate code for this class.
   - The `@Document` annotation indicates that instances of this class will be stored in the "tasks" collection within MongoDB.

4. **Field Declarations**:
   ```java
   @Id
   public String taskId;
   
   public String taskName;
   public String taskType;
   public String taskStatus;
   
   public String inputData;
   
   public String outputData;
   
   public String taskListName;
   ```
   - Seven fields are declared, which correspond to the attributes of a task document to be stored in MongoDB.
   - The `@Id` annotation on `taskId` denotes it as the unique identifier for the document.

5. **Constructor**:
   ```java
   public MongoTaskDocument(String taskId, String taskName, String taskType,
                            String inputData, String outputData,
                            String taskStatus, String taskListName) {
   
       this.taskId = taskId;
       this.taskName = taskName;
       this.taskType = taskType;
       this.inputData = inputData;
       this.outputData = outputData;
       this.taskStatus = taskStatus;
       this.taskListName = taskListName;
   
   }
   ```
   - A constructor is defined to initialize a `MongoTaskDocument` object with specified values for each field.

The `MongoTaskDocument` class serves as a representation of a task document within the application, with fields and a constructor allowing for the creation and manipulation of task documents as Java objects. These objects can then be persisted to, or read from, the MongoDB database using Spring Data MongoDB's repository and template abstractions.



#### out.persistence.mongodb.TaskMapper

The code snippet provided is a Java class named `TaskMapper` in the package `ch.unisg.tapastasks.tasks.adapter.out.persistence.mongodb`. This class acts as a mapper between the domain entity `Task` and the MongoDB document entity `MongoTaskDocument`. It's annotated with `@Component`, indicating to Spring Framework that it's a bean to be managed in the Spring context. Below is a detailed breakdown of the `TaskMapper` class:

1. **Package and Import Statements**:
   ```java
   package ch.unisg.tapastasks.tasks.adapter.out.persistence.mongodb;

   import ch.unisg.tapastasks.tasks.domain.Task;
   import ch.unisg.tapastasks.tasks.domain.TaskList;
   import org.springframework.stereotype.Component;
   ```
   - The package name and import statements indicate the location and dependencies of this class within the project's structure.

2. **Class Declaration**:
   ```java
   @Component
   class TaskMapper {
   ```
   - The `@Component` annotation tells Spring to treat this class as a component and manage it as a bean within the application context.

3. **Method `mapToDomainEntity`**:
   ```java
   Task mapToDomainEntity(MongoTaskDocument task) {
       return Task.createwithIdNameTypeInputOutputStatus(
           new Task.TaskId(task.taskId),
           new Task.TaskName(task.taskName),
           new Task.TaskType(task.taskType),
           new Task.InputData(task.inputData),
           new Task.OutputData(task.outputData),
           new Task.TaskStatus(Task.Status.valueOf(task.taskStatus))
       );
   }
   ```
   - This method takes a `MongoTaskDocument` object as input and maps it to a `Task` domain entity.
   - It calls a factory method on the `Task` class (`createwithIdNameTypeInputOutputStatus`) to create a new `Task` object, passing in the values from the `MongoTaskDocument` object as arguments.

4. **Method `mapToMongoDocument`**:
   ```java
   MongoTaskDocument mapToMongoDocument(Task task) {
   
       String taskInputData =
           (task.getInputData() == null) ? ""
               : task.getInputData().getValue();
   
       String taskOutputData =
           (task.getOutputData() == null) ? ""
               : task.getOutputData().getValue();
   
       return new MongoTaskDocument(
           task.getTaskId().getValue(),
           task.getTaskName().getValue(),
           task.getTaskType().getValue(),
           taskInputData,
           taskOutputData,
           task.getTaskStatus().getValue().toString(),
           TaskList.getTapasTaskList().getTaskListName().getValue()
       );
   }
   ```
   - This method takes a `Task` domain entity as input and maps it to a `MongoTaskDocument` object.
   - It checks if the `getInputData()` and `getOutputData()` methods on the `Task` object return null, using ternary operators to either get the value or default to an empty string.
   - It then creates and returns a new `MongoTaskDocument` object, passing in the values from the `Task` object (and the `TaskList` singleton) as arguments.

The `TaskMapper` class serves as a bridge between the domain and persistence layers of the application, allowing for a clean separation of concerns and easier testing and maintenance. By providing methods to map between domain entities and persistence entities, it encapsulates the logic needed to translate between these two representations of a task.



#### out.persistence.mongodb.TaskPersistenceAdapter

The provided Java code defines a class named `TaskPersistenceAdapter` in the package `ch.unisg.tapastasks.tasks.adapter.out.persistence.mongodb`. This class plays a role in persisting and retrieving `Task` domain objects to/from MongoDB. Here's a detailed breakdown:

1. **Package and Import Statements**:
   ```java
   package ch.unisg.tapastasks.tasks.adapter.out.persistence.mongodb;

   import ...
   ```
   - The package name and import statements indicate the location and dependencies of this class within the project's structure.

2. **Class Declaration**:
   ```java
   @Component
   @RequiredArgsConstructor
   public class TaskPersistenceAdapter implements
       AddTaskPort,
       LoadTaskPort {
   ```
   - The `@Component` annotation tells Spring to treat this class as a component and manage it as a bean within the application context.
   - The `@RequiredArgsConstructor` annotation from Lombok generates a constructor requiring all final fields, and fields with constraints such as `@NonNull`.
   - The class implements two interfaces: `AddTaskPort` and `LoadTaskPort`, defining contracts for adding and loading tasks, respectively.

3. **Member Variables**:
   ```java
   @Autowired
   private final TaskRepository taskRepository;

   private final TaskMapper taskMapper;
   ```
   - `taskRepository` is an instance of `TaskRepository`, annotated with `@Autowired` to instruct Spring to inject a bean of type `TaskRepository`.
   - `taskMapper` is an instance of `TaskMapper`, used for converting between domain entities and database entities.

4. **Method `addTask`**:
   ```java
   @Override
   public void addTask(Task task) {
       MongoTaskDocument mongoTaskDocument = taskMapper.mapToMongoDocument(task);
       taskRepository.save(mongoTaskDocument);
   }
   ```
   - This method overrides `addTask` from `AddTaskPort` interface.
   - It first converts the `Task` domain object to a `MongoTaskDocument` using the `taskMapper`.
   - It then saves the `MongoTaskDocument` to the database using `taskRepository.save`.

5. **Method `loadTask`**:
   ```java
   @Override
   public Task loadTask(Task.TaskId taskId, TaskList.TaskListName taskListName) throws TaskNotFoundError {
       MongoTaskDocument mongoTaskDocument = taskRepository.findByTaskId(taskId.getValue(),taskListName.getValue());
       if (mongoTaskDocument == null) {
           throw new TaskNotFoundError();
       }
       Task task = taskMapper.mapToDomainEntity(mongoTaskDocument);
       return task;
   }
   ```
   - This method overrides `loadTask` from `LoadTaskPort` interface.
   - It tries to find a `MongoTaskDocument` in the database using `taskRepository.findByTaskId` with `taskId` and `taskListName` as arguments.
   - If no `MongoTaskDocument` is found, it throws a `TaskNotFoundError`.
   - If a `MongoTaskDocument` is found, it converts it back to a `Task` domain object using `taskMapper`, and returns it.

This `TaskPersistenceAdapter` class is an example of the Adapter pattern, acting as a bridge between the application's core logic (domain layer) and the database (persistence layer). By implementing the `AddTaskPort` and `LoadTaskPort` interfaces, it ensures a clean separation of concerns, making the system easier to maintain and test.

#### out.persistence.mongodb.TaskRepository

The provided Java code defines an interface named `TaskRepository` within the package `ch.unisg.tapastasks.tasks.adapter.out.persistence.mongodb`. This interface is intended to be a Spring Data repository for interacting with MongoDB to persist and retrieve `MongoTaskDocument` objects. Here's a detailed explanation:

1. **Package and Import Statements**:
   ```java
   package ch.unisg.tapastasks.tasks.adapter.out.persistence.mongodb;

   import org.springframework.data.mongodb.repository.MongoRepository;
   import org.springframework.stereotype.Repository;

   import java.util.List;
   ```
   - The package name indicates the location of this class within the project's structure.
   - The import statements include necessary classes and interfaces from the Spring framework and the Java standard library that are used in this interface.

2. **Interface Declaration**:
   ```java
   @Repository
   public interface TaskRepository extends MongoRepository<MongoTaskDocument,String> {
   ```
   - The `@Repository` annotation is a Spring stereotype annotation that indicates this interface is a repository.
   - The interface `TaskRepository` extends `MongoRepository`, a Spring Data repository interface for MongoDB. The generic parameters `<MongoTaskDocument, String>` indicate that this repository manages `MongoTaskDocument` entities with `String` as the type of the ID field.

3. **Method Declarations**:
   ```java
   public MongoTaskDocument findByTaskId(String taskId, String taskListName);

   public List<MongoTaskDocument> findByTaskListName(String taskListName);
   ```
   - `findByTaskId(String taskId, String taskListName)`: This method declaration is for a custom query method to find a `MongoTaskDocument` based on the provided `taskId` and `taskListName`. The method returns a single `MongoTaskDocument` object.
   - `findByTaskListName(String taskListName)`: This method declaration is for a custom query method to find all `MongoTaskDocument` objects associated with the provided `taskListName`. The method returns a `List` of `MongoTaskDocument` objects.

These method declarations follow the Spring Data naming conventions, which allow for the automatic generation of the underlying MongoDB queries. The `TaskRepository` interface serves as a contract for the repository, and Spring Data MongoDB will provide the implementation at runtime, allowing for easy data access and manipulation with MongoDB.

The process of interacting with MongoDB, as depicted in the provided code, is structured within a typical Spring Boot application where the Spring Data MongoDB framework is utilized to simplify database interactions. Here's how the different pieces fit together based on the given code:

1. **Configuration and Initialization**:
   - The structure of the MongoDB database and its initialization is likely configured through the Spring Boot application's properties or YAML file, although this file hasn't been provided.
   - The `Docker-compose` file you provided earlier also includes the setup for a MongoDB container which would start a MongoDB instance with the specified configuration.

2. **Domain Representation**:
   - `MongoTaskDocument.java` represents the structure of a task document in MongoDB. The `@Document` annotation specifies the collection name as "tasks".

3. **Repository Interface**:
   - `TaskRepository.java` is the repository interface that extends `MongoRepository`, providing the application with methods to interact with MongoDB. It declares methods for querying tasks by ID and task list name.

4. **Persistence Adapter**:
   - `TaskPersistenceAdapter.java` is where the process of interacting with MongoDB is initiated. This class is annotated with `@Component`, so it's a Spring-managed bean.
   - It contains references to `TaskRepository` and `TaskMapper` which are injected by Spring (denoted by `@Autowired` and `@RequiredArgsConstructor`). 
   - The methods `addTask` and `loadTask` in `TaskPersistenceAdapter` use `TaskRepository` to interact with MongoDB. When `addTask` is called, it maps the domain `Task` object to a `MongoTaskDocument` object and saves it to MongoDB using `TaskRepository.save()`. When `loadTask` is called, it queries MongoDB using `TaskRepository.findByTaskId()` and maps the returned `MongoTaskDocument` object back to a domain `Task` object.

5. **Mapping**:
   - `TaskMapper.java` provides the mapping between the domain model (`Task`) and the MongoDB document model (`MongoTaskDocument`), enabling the conversion between these two models for saving and retrieving data.

6. **Web Adapter (Possibly)**:
   - It's plausible that a web adapter, like the `PublishNewTaskAddedEventWebAdapter.java` class you provided earlier, could trigger the interaction with the MongoDB by invoking methods on a service that, in turn, interacts with `TaskPersistenceAdapter`.

The actual initiation of a process that leads to interaction with MongoDB would typically come from a service layer or a web layer within the application, which calls the `addTask` or `loadTask` methods on `TaskPersistenceAdapter`.

