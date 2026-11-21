Application Schema and Architecture
Section 1: Architecture summary
This Spring Boot application uses a hybrid architecture combining both MVC and REST controllers to handle different user workflows. Thymeleaf server-side templates are used to render the Admin and Doctor dashboards, while RESTful APIs serve all other modules, such as patient portals and appointment management.
For persistence, the application interacts with two distinct databases: MySQL for structured relational data (patient, doctor, appointment, and admin records) and MongoDB for flexible, document-based prescription data. All incoming controller requests are routed through a unified service layer that encapsulates business logic and delegates data operations to the appropriate repositories. MySQL interactions rely on Spring Data JPA entities, whereas MongoDB interactions use Spring Data MongoDB document models.
Section 2: Numbered flow of data and control
A user initiates an action through the web interface, such as accessing the Admin Dashboard, Doctor Dashboard, or Appointment pages.
The incoming HTTP request is routed to either a Thymeleaf MVC controller (for dashboard views) or a REST controller (for API endpoints).
The controller validates the request parameters or payload and calls the appropriate method in the unified service layer.
The service layer executes the required business logic and determines which data store is needed for the operation.
For relational operations (patients, doctors, appointments, admins), the service delegates to a JPA repository using entity models; for prescriptions, it delegates to a MongoDB repository using document models.
The target database (MySQL or MongoDB) executes the query or update and returns the resulting records back through the repository and service layers to the controller.
The controller constructs the final response—rendering a Thymeleaf HTML view for MVC requests or returning a serialized JSON payload for REST requests—and sends it back to the user's browser.
