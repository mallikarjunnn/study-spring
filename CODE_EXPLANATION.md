# Flight Reservation System: Beginner's Code Guide

This guide explains the application's source in the order requested: XML, SQL, Java, and JSP. The CSS section is intentionally omitted. It is written for someone opening these files for the first time.

The explanations follow the source file by file. When a file repeats the same pattern many times (especially Java getters/setters and JSP layout tags), the pattern is explained once and each field or use is identified so the repeated lines are still understandable.

## Before the files: how the application works

A browser sends a request to the Java web application. A Spring **controller** receives that request, calls a **service** for the work, and the service calls a **DAO** (database access object) when data must be read or saved. The controller then chooses a JSP page and gives it the data to display. MySQL stores the records.

```text
Browser -> Controller -> Service -> DAO -> MySQL
Browser <- JSP view <- Controller <- Service <- DAO
```

- **Maven** reads `pom.xml`, downloads libraries, compiles Java, runs tests, and packages the app.
- **SQL** creates the MySQL database, tables, sample flights, and a view.
- **Java** configures Spring and contains request handling, business rules, database access, and data classes.
- **JSP** combines HTML with server-side tags to display pages and forms.

# 1. XML: Maven build file

## `pom.xml`

This is Maven's project file. XML uses nested elements: an opening tag such as `<version>` starts a value, and `</version>` closes it. The values tell Maven what the project is and which libraries it needs.

### The document header and project identity

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.deloitte</groupId>
  <artifactId>flight-reservation-system</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>war</packaging>
```

- `<?xml ...?>` says this is XML and that text is encoded as UTF-8.
- `<project ...>` is the outermost Maven element. `xmlns` names the Maven XML vocabulary; `xsi` and `schemaLocation` tell XML tools how to check the document's structure.
- `<modelVersion>4.0.0</modelVersion>` chooses the Maven project-file format version. It is not the application's Java version.
- `<groupId>` is the organization/project namespace.
- `<artifactId>` is the artifact's name.
- `<version>` is this project's release version. `SNAPSHOT` means it is a development build.
- `<packaging>war</packaging>` tells Maven to make a **WAR** (Web Application Archive) for a servlet container such as Tomcat.

### Shared build properties

```xml
<properties>
  <maven.compiler.source>17</maven.compiler.source>
  <maven.compiler.target>17</maven.compiler.target>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <spring.version>5.3.39</spring.version>
</properties>
```

- `<properties>` holds named values reused elsewhere in this file.
- `maven.compiler.source` and `target` select Java 17 language and bytecode compatibility.
- `project.build.sourceEncoding` says source files use UTF-8.
- `spring.version` stores one Spring version; `${spring.version}` later means “substitute this value here.”

### Dependencies: code written by other projects

Every `<dependency>` tells Maven to download a library. Its `groupId` identifies its publisher/project family; its `artifactId` names the library; `version` chooses the release; `scope` says when it is needed.

- `spring-context`: Spring's application context and dependency-injection features. The source uses annotations such as `@Configuration`, `@Service`, and `@Bean`.
- `spring-webmvc`: Spring MVC request routing, controllers, model data, JSP view resolution, and web configuration.
- `spring-jdbc`: Spring's `JdbcTemplate`, which runs SQL without requiring the project to manually open and close every JDBC resource.
- `mysql:mysql-connector-java`: the JDBC driver that lets Java speak to MySQL.
- `javax.servlet:jstl`: JSTL tags used by JSP pages for conditions, loops, formatting, and functions.
- `javax.servlet-api`: servlet interfaces needed to compile `WebAppInitializer`. `provided` means Tomcat supplies this at runtime; Maven does not bundle a second servlet API in the WAR.
- `javax.servlet.jsp-api`: JSP interfaces. It is also `provided` because Tomcat supplies them.
- `commons-dbcp2`: Apache's database connection pool, used to reuse database connections.
- `slf4j-api`: a common logging interface.
- `logback-classic`: an implementation that writes logs for SLF4J.
- `hibernate-validator` and `validation-api`: implement and define Java Bean Validation annotations such as `@NotBlank`, `@Pattern`, and `@Min`.
- `junit-jupiter`: JUnit 5 test framework. Its `test` scope means it is for tests, not the deployed application.
- `mockito-core` and `mockito-junit-jupiter`: make fake dependencies for isolated unit tests; also test-scoped.

The `<dependencies>` element contains these library declarations. Maven downloads their transitive dependencies too, meaning libraries that the listed libraries themselves require.

### Build plugins

```xml
<build>
  <finalName>flight-reservation-system</finalName>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-war-plugin</artifactId>
      <version>3.4.0</version>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.2.5</version>
      <configuration>
        <argLine>-XX:+EnableDynamicAgentLoading -Xshare:off</argLine>
      </configuration>
    </plugin>
  </plugins>
</build>
```

- `<build>` contains packaging/build instructions.
- `<finalName>` names the resulting WAR file.
- `maven-war-plugin` builds the web archive.
- `maven-surefire-plugin` runs tests during Maven's test phase.
- `<configuration>` customizes that test plugin. `argLine` passes JVM options used for the test process and Mockito's inline mocking agent.
- `</project>` closes the outer XML element. Each closing tag must match its opening tag.

## `src/main/resources/application.properties`

This is Java properties syntax rather than XML. Each line has a `name=value` pair. `AppConfig` reads the `db.*` keys. The file contains a local database password, so this guide deliberately does not repeat its value. Keep real credentials out of shared documentation and source control; use a local-only value or environment-managed secret for your own setup.

- `db.driver` identifies the MySQL JDBC driver class.
- `db.url` identifies the database server and database name.
- `db.username` is the database login name.
- `db.password` is the database login password; do not publish it.
- A line beginning with `#` is a comment and is ignored by Java's properties reader.

# 2. SQL: database structure and sample data

## `db/schema.sql`

SQL is a language for defining and working with relational databases. A table stores rows. A row represents one item, and columns represent the item's properties. The schema script is intended to be run in MySQL.

### Create and select the database

```sql
CREATE DATABASE IF NOT EXISTS flightdb;
USE flightdb;
```

- `CREATE DATABASE` makes a database container named `flightdb`.
- `IF NOT EXISTS` prevents an error if that database already exists.
- `USE flightdb` selects it so later statements operate inside it.
- A semicolon ends a SQL statement. `--` starts a comment that SQL ignores.

### Remove old tables in dependency order

```sql
DROP TABLE IF EXISTS ticket;
DROP TABLE IF EXISTS booking;
DROP TABLE IF EXISTS flight;
DROP TABLE IF EXISTS users;
```

- `DROP TABLE` removes a table and its data.
- `IF EXISTS` means no error if the table is not there.
- Tickets depend on bookings, and bookings depend on flights and users. Dropping dependent tables first avoids foreign-key conflicts.
- Important: running this script deletes existing rows in these tables before recreating them.

### `users` table

```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL
);
```

- `CREATE TABLE users` begins the definition of a table named `users`.
- `user_id` is a large integer. `AUTO_INCREMENT` asks MySQL to choose the next number, and `PRIMARY KEY` makes it the unique identifier for each row.
- `email` can hold up to 100 characters. `UNIQUE` prevents two users having the same email; `NOT NULL` requires a value.
- `password` can hold up to 255 characters and cannot be missing.
- `role` stores the user's role, such as `User` or `Admin`; it cannot be missing.
- A comma separates column definitions. The final `);` ends the table definition.

### `flight` table

```sql
CREATE TABLE flight (
    flight_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    flight_number VARCHAR(50) UNIQUE NOT NULL,
    source VARCHAR(50) NOT NULL,
    destination VARCHAR(50) NOT NULL,
    departure_time VARCHAR(50) NOT NULL,
    arrival_time VARCHAR(50) NOT NULL,
    capacity INT NOT NULL DEFAULT 20,
    price_per_seat DOUBLE NOT NULL
);
```

- `flight_id` is the automatically generated primary key.
- `flight_number` identifies the airline's flight and is unique and required.
- `source` and `destination` store the origin and destination city names.
- `departure_time` and `arrival_time` store the schedule as text in this schema.
- `capacity` is an integer. If an insert omits it, `DEFAULT 20` supplies 20.
- `price_per_seat` stores the price of one seat.

### `booking` table and relationships

```sql
CREATE TABLE booking (
    booking_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    booking_date DATE NOT NULL,
    journey_date DATE NOT NULL,
    seats INT NOT NULL,
    total_price DOUBLE NOT NULL,
    status VARCHAR(50) NOT NULL,
    flight_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    FOREIGN KEY (flight_id) REFERENCES flight(flight_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

- `booking_id` uniquely identifies one reservation.
- `booking_date` is when the user made it; `journey_date` is when the flight is taken. `DATE` stores a calendar date.
- `seats` is the count of passenger seats in the booking.
- `total_price` stores the booking's price; `status` stores a value such as `CONFIRMED` or `CANCELLED`.
- `flight_id` and `user_id` connect the booking to its flight and user.
- `FOREIGN KEY (...) REFERENCES ...` makes each value refer to an existing primary key. It prevents a booking from pointing to a nonexistent flight or user.

### `ticket` table

```sql
CREATE TABLE ticket (
    pnr_no BIGINT AUTO_INCREMENT PRIMARY KEY,
    passenger_name VARCHAR(50) NOT NULL,
    gender VARCHAR(50) NOT NULL,
    age INT NOT NULL,
    journey_date DATE NOT NULL,
    seat_no VARCHAR(20) NOT NULL,
    booking_id BIGINT NOT NULL,
    status VARCHAR(50) NOT NULL,
    FOREIGN KEY (booking_id) REFERENCES booking(booking_id)
);
```

- Each row represents one passenger's ticket.
- `pnr_no` is the automatically generated ticket identifier.
- `passenger_name`, `gender`, and `age` store passenger details.
- `journey_date` and `seat_no` store the trip date and assigned seat label.
- `booking_id` links this ticket to the booking that owns it.
- `status` indicates whether the ticket is confirmed or cancelled.
- The foreign key requires the parent booking to exist before its ticket can be inserted.

### Add sample flights

The `INSERT INTO flight (...) VALUES ...` statement adds the ten sample flight rows. The column list names the destination columns in order. Each parenthesized group is one row; its values must appear in the same order as the column list. Text values use single quotes, and decimal prices use a decimal point. The final semicolon ends the multi-row insert.

### Verify row counts

```sql
SELECT 'users' AS tbl, COUNT(*) AS rows_ct FROM users
UNION ALL SELECT 'flight', COUNT(*) FROM flight
UNION ALL SELECT 'booking', COUNT(*) FROM booking
UNION ALL SELECT 'ticket', COUNT(*) FROM ticket;
```

- `SELECT` asks the database to return information.
- `'users' AS tbl` returns the text `users` in a result column named `tbl`.
- `COUNT(*)` counts all rows in that table; `AS rows_ct` names the count column.
- `UNION ALL` stacks the next `SELECT` result below the previous one without removing duplicate rows.
- The four counts let a person check whether the tables contain data.

### Create the flight availability view

```sql
CREATE OR REPLACE VIEW flight_availability AS
SELECT f.flight_id, f.flight_number, f.source, f.destination,
       f.departure_time, f.arrival_time, f.price_per_seat,
       f.capacity,
       f.capacity - COALESCE(SUM(CASE WHEN t.status = 'CONFIRMED'
                                     THEN 1 ELSE 0 END), 0) AS seats_left
FROM flight f
LEFT JOIN booking b ON b.flight_id = f.flight_id AND b.status = 'CONFIRMED'
LEFT JOIN ticket t ON t.booking_id = b.booking_id AND t.status = 'CONFIRMED'
GROUP BY f.flight_id, f.flight_number, f.source, f.destination,
         f.departure_time, f.arrival_time, f.price_per_seat, f.capacity;
```

- A **view** is a saved query that behaves like a read-only table. `OR REPLACE` updates it if it already exists.
- `f` is a short alias for `flight`; aliases make long queries easier to read.
- `SELECT f.column` chooses the flight information to return.
- `CASE WHEN ... THEN 1 ELSE 0 END` counts a confirmed ticket as 1 and every other ticket as 0.
- `SUM(...)` adds those values; `COALESCE(value, 0)` changes a missing (`NULL`) sum to zero.
- `capacity - confirmed_ticket_count AS seats_left` calculates remaining seats and names the result column `seats_left`.
- `FROM flight f` starts with every flight.
- `LEFT JOIN` adds matching bookings/tickets while still retaining flights that have no match.
- `ON` states how table rows are related; the status conditions only join confirmed records.
- `GROUP BY` groups the joined rows by flight columns so `SUM` can calculate a separate seat count for each flight.
- `SHOW FULL TABLES WHERE Table_type = 'VIEW'` lists views. `SELECT * FROM flight_availability` displays the view's rows and columns.

# 3. Java: application logic

## Java words used throughout the project

- A **package** is a namespace and folder-like grouping. `package com.flightreservation.service;` says which package owns a class.
- An **import** lets a file use another class by its short name instead of its full package name.
- A **class** is a definition for an object or a collection of related behavior.
- A **field** is data stored on an object. A **method** is an action a class can perform.
- `private` means only this class can access a member directly. `public` means other classes can access it.
- `final` means the variable or field cannot be reassigned after initialization. `static` means it belongs to the class itself rather than a particular object.
- `return` sends a result back to the code that called a method. `void` means a method sends no result back.
- `{` begins a block; `}` ends it. `;` ends most Java statements.
- `@Something` is an annotation: extra information that a framework or compiler uses to give code a role.
- Java's `List<Type>` means a list containing values of that type. `null` means no object/value is present.
- `this.field` means the field on the current object. In a setter, `this.email = email` copies the method argument into that object's field.
- A constructor has the same name as its class and runs when a new object is created.

## 3.1 Configuration and startup

### `config/AppConfig.java`

- The `package` line places this class in the configuration package. The imports bring in the DBCP connection pool, Spring annotations, the `JdbcTemplate`, and JDBC's `DataSource` interface.
- `@Configuration` tells Spring this class declares application objects (called **beans**).
- `@ComponentScan(...)` asks Spring to find annotated classes in the service and DAO packages and create/manage them.
- `@PropertySource(...)` loads `application.properties` from the application's classpath.
- `public class AppConfig` begins the configuration class.
- Each `@Value("${db...}")` field asks Spring to take one named property and place it in the matching Java `String`: driver, URL, database username, or password. The secret itself is not documented here.
- `@Bean` on `placeholderConfigurer()` makes the returned helper a Spring bean. `PropertySourcesPlaceholderConfigurer` replaces `${...}` placeholders with property values.
- `dataSource()` creates `BasicDataSource`, fills its driver, URL, username, and password from the fields, and returns it as the general `DataSource` type. This pool manages reusable database connections.
- `jdbcTemplate(DataSource dataSource)` receives the data source from Spring, creates a `JdbcTemplate`, and returns it. DAOs use this helper for SQL.

### `config/WebConfig.java`

- Imports and package declaration identify the web configuration and Spring MVC types.
- `@Configuration` marks this as Spring configuration.
- `@EnableWebMvc` enables Spring MVC support.
- `@ComponentScan("com.flightreservation")` tells Spring to find web components such as controllers and validators.
- `implements WebMvcConfigurer` means this class can customize Spring MVC by implementing its configuration hooks.
- `viewResolver()` creates an `InternalResourceViewResolver`. `setPrefix` and `setSuffix` tell it to turn a name like `home` into `/WEB-INF/views/home.jsp`.
- `addResourceHandlers(...)` maps web requests beginning with `/resources/` to files in the application's `/resources/` directory. This allows static resources to be served.

### `config/WebAppInitializer.java`

- The imports name Servlet API and Spring classes needed at server startup.
- `implements WebApplicationInitializer` makes this a Java replacement for the traditional `web.xml` startup file.
- `onStartup(ServletContext servletContext)` is called by the servlet container when the app starts. `throws ServletException` allows startup-related servlet errors to be reported.
- The root `AnnotationConfigWebApplicationContext` registers `AppConfig`; a `ContextLoaderListener` attaches it to the servlet lifecycle. This context owns shared services and database objects.
- A second context registers `WebConfig`; it owns web MVC configuration and controllers.
- `addServlet("dispatcher", new DispatcherServlet(mvcContext))` creates Spring's front controller. The front controller receives incoming web requests and routes each to the matching controller.
- `setLoadOnStartup(1)` asks the server to initialize the servlet when the app starts rather than waiting for the first request.
- `addMapping("/")` routes the application's requests through this servlet.

## 3.2 Controllers: matching browser requests

Spring controller methods generally return a `String` naming the JSP view or a redirect. A `Model` is a container of values to pass to the JSP. `HttpSession` stores information across multiple requests from the same browser.

### `controller/HomeController.java`

- `@Controller` marks the class as a web request handler.
- The constructor needs no service because this controller only prepares the landing page.
- `@GetMapping("/")` connects an HTTP GET to the website root to `home()`.
- `home(Model model)` puts a fresh `FlightSearchForm` into the model under `searchForm`, matching the form name used by `home.jsp`.
- `return "home"` asks the view resolver to display `home.jsp`.

### `controller/FlightController.java`

- `@Controller` makes the class handle web requests.
- The two `private final` fields hold the flight service and search validator. The constructor receives them; Spring supplies them automatically (**constructor injection**).
- `@PostMapping("/flights/search")` handles a form submission sent with HTTP POST.
- `@ModelAttribute("searchForm")` tells Spring to create/populate a `FlightSearchForm` using submitted fields.
- `BindingResult` receives validation errors; `Model` holds values for the JSP.
- `searchValidator.validate(form, result)` checks the source/destination pair.
- `if (result.hasErrors())` tests for validation errors; if any exist, returning `home` displays the search form again with errors.
- Otherwise, `flightService.searchFlights(...)` finds matches. The result list is added to the model as `flights`; the submitted form is added as `search`.
- `return "searchResults"` displays `searchResults.jsp`.

### `controller/AuthController.java`

- The class is annotated `@Controller`; its constructor injects `UserService`.
- `@GetMapping("/signup")` shows signup. `signupForm` adds an empty `UserDto` under `userDto`, because the JSP form needs an object to bind to, then returns `signup`.
- `@PostMapping("/signup")` handles submitted signup data. `@Valid` asks Bean Validation to check the DTO annotations; `@ModelAttribute` binds form fields; `BindingResult` receives errors and must follow the validated argument.
- If validation has errors, the method redisplays the form. Otherwise `userService.register` attempts the registration. A `false` result adds a duplicate-email message and redisplays the form. Success returns a redirect to the login page with a `registered` query parameter.
- `@GetMapping("/login")` displays the login JSP.
- `@PostMapping("/login")` reads `email` and `password` request parameters and asks `UserService` to check them.
- If the service returns `null`, the controller adds an error and redisplays login. If it returns a user, `session.setAttribute("currentUser", user)` remembers the signed-in user in the session.
- The role comparison routes an administrator to the admin dashboard and other users to their booking list.
- `@GetMapping("/logout")` handles logout; `session.invalidate()` clears the session, then redirects to `/`.

### `controller/BookingController.java`

- The imports provide booking/flight/ticket/user objects, services, session, date utilities, and list support.
- `@Controller` marks a request handler; `@RequestMapping("/booking")` makes all mappings in the class start with `/booking`.
- The constructor injects `BookingService` and `FlightService` into final fields.
- `isCustomer(session)` reads `currentUser`. It returns true only if a user exists and the role is not Admin. The expression `u != null && ...` uses short-circuit AND, so it checks the role only when a user exists.
- `start(...)` handles `GET /booking/start`. It reads `flightId`, rejects signed-out visitors by redirecting to login, finds the flight, and adds the flight plus today's date and the latest allowed date (12 months ahead) to the model. It returns `bookingForm`.
- `confirm(...)` handles the booking form's POST. The `@RequestParam` values are read from submitted inputs: flight id, journey date, and parallel lists of passenger names, genders, and ages.
- It checks customer access and login, converts the date text into `LocalDate`, and calculates the permitted date window. A date before today or beyond 12 months reloads the form with an error.
- `bookingService.createBooking(...)` performs the actual multi-row operation. The `try/catch` handles an `IllegalStateException` such as insufficient seats by redisplaying the form with the message.
- On success, the returned booking id is added to a redirect URL for the ticket page.
- `confirmGet()` catches a GET to the confirmation URL and redirects home; confirmation should be submitted as POST.
- `ticket(...)` reads the booking id from the URL. It requires a session user, loads the booking, then compares its owner id to the signed-in user's id. A mismatch redirects to the user's own bookings. A valid owner gets the booking, flight, and tickets added to the model and sees `ticket.jsp`.
- `myBookings(...)` requires a signed-in user, loads their summaries, and displays `myBookings.jsp`.
- `cancel(...)` requires a user, asks the service to cancel the booking for that user's id, then redirects to the booking list.
- `cancelTickets(...)` reads optional selected PNR numbers. When at least one was provided, it asks the service to cancel only those tickets, then returns to that booking's ticket page.

### `controller/AdminController.java`

- `@Controller` marks the class; `@RequestMapping("/admin")` adds an `/admin` prefix to its routes.
- Its constructor injects `FlightService`.
- `isAdmin(session)` retrieves the session user and checks that the role equals Admin, ignoring capitalization.
- `dashboard(...)` rejects non-admin sessions by redirecting to login. Otherwise it adds upcoming and completed flight statistics to the model and returns `adminDashboard`.
- `addForm(...)` checks admin access, creates an empty `FlightForm`, puts it in the model under `flightForm`, and returns `addFlight`.
- `addSubmit(...)` receives the validated form, validation result, session, and redirect flash-attribute holder.
- `@Valid` checks field annotations on `FlightForm`. The additional source/destination condition compares trimmed city names without considering letter case and attaches an error to `destination` if they match.
- The arrival/departure condition compares `HH:mm` strings and rejects arrival that is not later than departure.
- `result.hasErrors()` returns the form with its field messages.
- The `try` calls the service to save the flight. A duplicate number becomes an error on `flightNumber`; another exception becomes a general save error.
- `addFlashAttribute` saves success details for the next request only. The redirect follows the Post/Redirect/Get pattern and displays the dashboard.

## 3.3 Services: business rules

### `service/UserService.java`

- `@Service` tells Spring this class contains application/business logic. The constructor injects `UserDao`.
- `register(dto)` asks the DAO whether the email exists. If so, it returns `false` immediately.
- Otherwise it creates a `User`, copies email/password/role from the form DTO, saves it through the DAO, and returns `true`.
- This project's current implementation stores and compares passwords as plain text. This is not suitable for a real account system; real apps should hash passwords with a password-hashing algorithm and use a proper authentication framework.
- `login(email, password)` looks up the email. If a user exists and the stored password equals the submitted password, it returns that user; otherwise it returns `null`.

### `service/FlightService.java`

- `@Service` makes it a Spring-managed service. `DEFAULT_CAPACITY` is a class constant of 20. The constructor injects `FlightDao`.
- `searchFlights`, `getAllFlights`, and `getFlightById` delegate to corresponding DAO methods. They keep controller code separate from SQL details.
- `addFlight(Flight)` validates uniqueness and passes an existing model to the DAO.
- `addFlight(FlightForm)` validates the form's flight number, fills in capacity 20 if it is `null`, converts the form to a `Flight`, then saves it.
- `ensureFlightNumberIsUnique` trims spaces; a missing/blank number throws `IllegalStateException`. It asks the DAO whether the normalized number already exists and throws if so.
- `getUpcomingFlights()` and `getCompletedFlights()` ask the DAO for statistics using `true` and `false` respectively. The boolean selects the date category.

### `service/BookingService.java`

- `@Service` registers the class with Spring. Its constructor injects `BookingDao` and `FlightService`.
- `@Transactional` means Spring runs the method inside a database transaction. If a runtime error occurs during the booking operation, related writes can roll back together.
- `createBooking(...)` first loads the flight, subtracts booked tickets from capacity, and throws an error if the requested passenger count exceeds available seats.
- It computes `seats` from the names list and multiplies per-seat price by seat count using `BigDecimal` for monetary arithmetic.
- It fills a new `Booking` with booking date, journey date, seats, total, confirmed status, flight id, and user id.
- It saves the booking first to get its generated id. It then loops once per passenger, builds a `Ticket` using the same list index for name/gender/age, sets date/seat/booking/status, and inserts it.
- `generateSeatNo(index)` assigns six seats per row. Integer division (`index / 6`) selects a row offset; remainder (`index % 6`) selects a letter from A through F.
- `getBooking` loads one booking and throws the app's not-found exception if the DAO returned no object.
- `getTickets` returns tickets for one booking; `getMyBookings` returns that user's booking summaries.
- `cancelBooking` looks up the booking owner before changing data. If no owner exists or it differs from the supplied user id, it returns false. Otherwise it marks booking and tickets cancelled. `@Transactional` keeps both updates together.
- `cancelTickets` checks the same ownership condition. It cancels selected PNRs, counts still-confirmed tickets, loads the associated flight, recalculates the total, chooses `CANCELLED` if none remain or `CONFIRMED` otherwise, and updates the booking totals/status.
- `getSeatsAvailable` subtracts confirmed booked tickets from flight capacity.

## 3.4 DAOs: SQL and database row conversion

A DAO is the layer that speaks to the database. `JdbcTemplate` runs SQL. A `?` is a parameter slot: the value is supplied separately, rather than joined into the SQL text. This is safer and helps prevent SQL injection. A `RowMapper` tells Spring how to turn one returned SQL row into one Java object.

### `dao/UserDao.java`

- `@Repository` marks a database component. The constructor receives and stores `JdbcTemplate`.
- `mapper` is a `RowMapper<User>` lambda. `(rs, i) -> { ... }` means “for each result row, build a User.” `rs` is the current result set row; `i` is its row number.
- Each `rs.get...("column")` reads a database column, and each `set...` copies it into the matching User field.
- `save(user)` runs `INSERT INTO users...`; the three question marks receive email, password, and role.
- `findByEmail(email)` runs a parameterized `SELECT`. `jdbc.query` returns a list. If it is empty, the method returns `null`; otherwise it returns the first user.

### `dao/FlightDao.java`

- `@Repository` marks this class for Spring. The constructor stores `JdbcTemplate`.
- Its `RowMapper<Flight>` creates a `Flight` and copies each database column into the matching property. The `rs.getInt`, `getString`, and `getBigDecimal` methods match the expected data type.
- `search(source, destination)` selects flights where lowercased city values match, so letter capitalization does not matter. `trim()` removes leading/trailing spaces. `ORDER BY departure_time` sorts the results.
- `findAll()` selects every flight and sorts by departure time.
- `findById(id)` uses `queryForObject`, which expects exactly one row. If no row exists, Spring throws `EmptyResultDataAccessException`; the catch block changes it into the project's `ResourceNotFoundException`.
- `flightNumberExists` counts rows with that number and returns whether the count is greater than zero.
- `addFlight` inserts the model's flight number, route, times, capacity, and price. Arguments are passed separately from the SQL.
- `findFlightStats(upcoming)` chooses `>=` for upcoming dates or `<` for past dates, then runs an aggregate query.
- In that query, `JOIN` combines flights with bookings; `LEFT JOIN` also permits a booking with no ticket rows. `COUNT(DISTINCT b.booking_id)` counts bookings without double-counting them for every passenger. `COUNT(t.pnr_no)` counts tickets. `CASE` and `SUM` include confirmed-ticket prices in revenue; `COALESCE` turns missing sums into zero. `DATEDIFF` counts days between journey date and today. `GROUP BY` makes one aggregate row per flight/date; `ORDER BY` sorts dates.
- The final `jdbc.query` uses a row-mapping lambda to build a `FlightStatsDto` for each result row.

### `dao/BookingDao.java`

- `@Repository` marks the class. Its constructor stores `JdbcTemplate`.
- `createBooking(booking)` creates a `GeneratedKeyHolder` to receive MySQL's auto-generated booking id.
- `jdbc.update(con -> { ... }, keyHolder)` uses a lambda to create a `PreparedStatement` from the connection. SQL uses seven `?` placeholders. Each `set...` line fills the matching placeholder; `Date.valueOf(LocalDate)` converts Java's date to JDBC's date format. `RETURN_GENERATED_KEYS` asks the driver for the new id. `keyHolder.getKey().longValue()` returns that id.
- `addTicket(ticket)` inserts one passenger ticket. Its placeholders are filled from ticket properties.
- `bookingMapper`, `ticketMapper`, and `summaryMapper` each convert one SQL row into a different Java type. The summary mapper combines selected booking and flight columns for the booking-list page.
- `findBookingById` selects one booking by its id. `findTicketsByBookingId` returns all ticket rows for the booking ordered by PNR.
- `findSummariesByUser` joins booking to flight, filters by user id, and orders newest booking id first.
- `findOwnerId` selects the user id belonging to a booking. The service uses it for authorization checks.
- `cancelBooking` sends two updates: one changes the booking status, the other changes every ticket under it.
- `cancelTicketsByPnr` loops through selected PNR values and updates each ticket only if both its PNR and booking id match.
- `countConfirmedTickets` counts confirmed tickets within one booking. `countBookedSeats` counts confirmed tickets on a flight across bookings. Each handles a possible null count by returning zero.
- `syncBooking` updates the booking's seat count, total amount, and status after ticket cancellation.

## 3.5 Models: Java objects for stored data

A model is a simple Java class that carries information. Most model lines are a field declaration followed by a getter and setter. For example:

```java
private String email;
public String getEmail() { return email; }
public void setEmail(String email) { this.email = email; }
```

- `private String email;` stores a text value, hidden from direct outside access.
- `getEmail()` returns the stored value. JavaBean getters let Spring and JSP Expression Language read a property called `email`.
- `setEmail(...)` changes that value. `this.email` is the field; the unqualified `email` on the right is the method argument.

The same pattern applies to the following fields in each model:

### `model/User.java`

- Stores a user's id, email, password, and role.
- Its getters return those values; setters fill them when a DAO reads a row or a service creates a user.

### `model/Flight.java`

- Stores flight id, flight number, source, destination, departure/arrival text, capacity, and price per seat.
- `DEFAULT_CAPACITY` is a constant set to 20. The `capacity` field starts at that value when a new Flight is created.
- `BigDecimal` is used for a price to avoid common floating-point rounding errors in money calculations.
- Its remaining getters/setters provide access to each field.

### `model/Booking.java`

- Stores booking id/date, journey date, number of seats, total price, status, flight id, and user id.
- `JOURNEY_DATE_FORMAT` is a shared formatter for a friendly date such as `Thu, 05 Dec 2023`.
- Its getters/setters expose each stored booking property.
- `getJourneyDateText()` returns an empty string if no journey date is set; otherwise it formats the date. JSPs can display this ready-made text.

### `model/Ticket.java`

- Stores PNR id, passenger name, gender, age, journey date, seat number, parent booking id, and ticket status.
- Its getters/setters expose these fields.
- `getJourneyDateText()` converts the date into the same readable display format, or an empty string if missing.

### `model/BookingSummary.java`

- Stores a flattened summary of a booking together with its flight number, cities, and times. “Flattened” means the JSP can use one object instead of following links between separate objects.
- Its getters/setters follow the standard property pattern.
- `getJourneyDateText()` formats the date for display.
- `isCancellable()` returns true only when status is `CONFIRMED`. JSP EL can use this boolean as `${b.cancellable}` to decide whether to show a cancel control.

### `model/FlightSearchForm.java`

- Stores the two search fields, `source` and `destination`.
- Its getters and setters let Spring copy submitted form values into it and let the JSP read them.

## 3.6 DTOs: form data and query results

DTO means **Data Transfer Object**. It carries data between layers, often specifically for a form or a read-only result.

### `dto/UserDto.java`

- The fields are email, password, and role.
- `@NotBlank` rejects a missing or whitespace-only value.
- `@Size(max = 100)` limits email length.
- `@Pattern(regexp = ...)` checks the text against a regular expression. The email expression checks for a basic `name@domain.suffix` shape.
- The password expression uses look-ahead checks `(?=...)` to require a digit, lowercase letter, uppercase letter, special character, and no whitespace; `.{8,20}` sets the length range.
- The role expression permits exactly `User` or `Admin`.
- The `message` in each annotation is the validation text shown to the user.
- The getters/setters expose all three values to Spring and JSP.

### `dto/FlightForm.java`

- Stores the fields an admin enters when adding a flight.
- `@NotBlank` requires text; `@Size` sets allowed city-name length; `@Pattern` limits flight number, city, and time formats.
- The time expression only permits a 24-hour `HH:mm` time.
- `@NotNull`, `@Min`, and `@Max` require capacity and set its numeric range.
- `@DecimalMin`, `@DecimalMax`, and `@Digits` constrain the price value and its precision.
- `toFlight()` creates a new model object. It trims the number and city text, uppercases the flight number, then copies the times, capacity, and price. This separates web-form input from the object passed to the DAO.
- The getter/setter methods expose each form field to Spring's form binding and validation.

### `dto/FlightStatsDto.java`

- This object carries admin-dashboard statistics returned from an aggregate SQL query. It is not a database table model.
- Every field is `private final`, meaning it can be assigned only once, in the constructor.
- The constructor arguments are copied into the matching fields. This is why there are getters but no setters: the result is read-only after creation.
- The getters return flight number, route/times, journey date, booking/passenger counts, total revenue, and days until the journey.

## 3.7 Validation and errors

### `validator/FlightSearchValidator.java`

- `@Component` lets Spring find and inject the validator.
- `implements Validator` promises that the class supplies Spring's `supports` and `validate` methods.
- `supports(clazz)` returns true only for `FlightSearchForm.class`, so Spring knows which form this validator handles.
- `validate(target, errors)` receives the object to check and a place to record problems.
- `ValidationUtils.rejectIfEmptyOrWhitespace` marks source/destination as required if they contain no useful text.
- The cast `(FlightSearchForm) target` treats the generic object as the expected form type.
- The final condition trims both city names and compares them without case sensitivity. If they are equal and not blank, `errors.rejectValue` attaches a message to `destination`.

### `exception/ResourceNotFoundException.java`

- Extends `RuntimeException`, Java's unchecked exception type.
- Its constructor accepts a message and passes it to `super(message)` so normal exception handling can retrieve it.
- Services/DAOs throw it when a requested database object does not exist.

### `exception/GlobalExceptionHandler.java`

- `@ControllerAdvice` makes this a shared handler for errors thrown by controllers.
- `@ExceptionHandler(...)` connects a method to one or more exception types.
- `@ResponseStatus(...)` chooses the HTTP response status returned to the browser.
- `handleNotFound` catches the app's not-found exception and JDBC's empty-result exception, puts a title/message into the model, and returns the `error` JSP with HTTP 404.
- `handleBusinessRule` catches `IllegalStateException`, displays its message in the error page, and uses HTTP 409 (conflict).
- `handleGeneric` catches other exceptions, logs request URI and exception to standard error, avoids showing technical details to the user, and returns the error view with HTTP 500.
- `HttpServletRequest` supplies request information such as the current URL path.

## 3.8 Java tests

A unit test checks one piece of logic without starting the web server or contacting MySQL. Mockito creates a fake DAO so the test controls what the DAO returns.

### `test/.../UserServiceLoginTest.java`

- `@ExtendWith(MockitoExtension.class)` enables Mockito in JUnit 5.
- `@Mock UserDao userDao` creates a fake DAO.
- `@InjectMocks UserService userService` creates the service and injects that fake DAO.
- `existingUser(...)` builds a test user with supplied email/password and a normal-user role.
- Each `@Test` method is a separate test case.
- `when(...).thenReturn(...)` tells the fake DAO what to return for a particular email lookup.
- The valid-credentials test calls login, then `assertNotNull` checks a user was returned and `assertEquals` checks its email.
- The wrong-password test expects `null` using `assertNull`.
- The unknown-user test configures the fake DAO to return null and also expects null.

### `test/.../UserServiceSignupTest.java`

- It sets up Mockito and injects the fake DAO in the same way as the login test.
- `@BeforeEach setUp()` runs before every test and creates a normal test `UserDto`.
- The valid signup case makes email lookup return null, calls `register`, checks `true`, and uses `verify` to ensure the DAO's `save` method was called.
- The duplicate-email case returns an existing user, expects `false`, and uses `never()` to ensure save was not called.
- The blank-email case configures a matching DAO result, expects false, and confirms there was no save. Bean Validation is not run directly in this service test; the test checks service behavior with the supplied DTO.
- `assertTrue`, `assertFalse`, and Mockito `verify` check outcomes and side effects, not just whether the code ran.

# 4. JSP: pages shown in the browser

## JSP basics used throughout

A JSP is mostly HTML with server-side tags. The server evaluates those tags and sends ordinary HTML to the browser.

- `<%@ page ... %>` is a JSP directive. Here it sets the response to UTF-8 HTML.
- `<%@ taglib prefix="c" ... %>` makes JSTL core tags available with the `c:` prefix. `form:`, `fmt:`, and `fn:` are Spring form, formatting, and JSTL functions tag libraries.
- `<%@ include file="..." %>` inserts a shared JSP fragment into the page.
- `${...}` is Expression Language (EL). It reads a model/session/request value or a JavaBean property. For example, `${flight.source}` calls the `getSource()` getter.
- `<c:if>` conditionally includes markup; `<c:choose>`, `<c:when>`, and `<c:otherwise>` are the if/else form; `<c:forEach>` repeats markup for a list.
- Spring's `<form:form>` and `<form:input>` bind input fields to a Java object. `path="email"` means the object's `email` property.
- An HTML form's `action` is the URL receiving data; `method="post"` sends the data in an HTTP POST request.
- `pageContext.request.contextPath` is the app's deployed URL prefix. Using it keeps links working even if the app is not deployed at the website root.
- HTML attributes such as `class`, `id`, `name`, and `type` label elements or control their behavior. CSS styling is not covered in this guide.

### `WEB-INF/views/fragments/header.jsp`

- The first JSP comments name the fragment and describe its purpose; comments are not shown in the browser.
- The `c` and `fn` taglib directives enable conditional/loop tags and string functions.
- `<!DOCTYPE html>` tells the browser to use modern HTML rules. `<html lang="en">` begins the page and declares its language.
- `<head>` contains page metadata. `charset` selects UTF-8, `viewport` helps mobile browsers size the page, and `<title>` sets the browser-tab title.
- The font `<link>` loads a web font. The stylesheet link points to the app's CSS, but CSS itself is intentionally outside this document.
- `<body>` begins visible page content.
- `c:set` creates a JSP variable named `role` using the session user's role converted to uppercase. The next two `c:set` lines determine if a logged-in user is an admin or a regular user.
- `<header>` and `<nav>` contain site branding and navigation links. The brand URL goes to the admin dashboard for an admin and to the site root otherwise.
- `<c:choose>` checks whether `currentUser` exists. `<c:when>` displays links for signed-in users; `<c:otherwise>` displays login/signup links for visitors.
- Inside the signed-in case, `<c:if>` shows admin links only to admins and customer links only to non-admin users.
- `${sessionScope.currentUser.email}` and `.role` read values from the session user object.
- Logout is a link to the controller's `/logout` route.
- This fragment starts the document and body; the footer fragment closes them.

### `WEB-INF/views/fragments/footer.jsp`

- The JSP comment says this is the shared footer and that it finishes tags opened by the header.
- `<footer>` displays the application's copyright/name text.
- `</body>` and `</html>` close the document started by `header.jsp`.

### `WEB-INF/views/fragments/statCard.jsp`

- The comment says the fragment expects a `FlightStatsDto` variable called `s`.
- The `fmt` taglib enables number formatting.
- The outer `<div>` groups one statistics card.
- `${s.source}`, `${s.destination}`, and `${s.flightNumber}` read the route and flight properties through Java getters.
- Each statistics cell has a label and a value: journey date, booking count, passengers, revenue, and days remaining.
- `<fmt:formatNumber>` formats revenue and limits displayed decimal digits.
- The last EL conditional displays the day count if it is nonnegative; otherwise it displays a dash for a past date.

### `WEB-INF/views/home.jsp`

- The comment lines describe the file and page; they do not appear in the browser.
- The page directive selects UTF-8 HTML. The `form`, `c`, and `fmt` tag libraries make Spring forms, JSTL core, and number/date formatting tags available; `fmt` is declared in this file even though the page does not use it.
- The header include supplies the shared document head and navigation.
- `<div class="wrap">` groups the page content. The hero section contains the page heading and short description.
- `<form:form>` binds to model object `searchForm`, submits to `/flights/search`, and uses POST.
- Each `.field` group has a label and a Spring input whose `path` matches a Java form property (`source` or `destination`).
- `placeholder` is example text shown before typing. `cssErrorClass` tells Spring to add an error class when that field failed validation.
- `<form:errors>` prints validation messages for that field.
- The submit button sends the form. The footer include completes the page.

### `WEB-INF/views/searchResults.jsp`

- The page directive sets UTF-8; `c` enables branching/loops and `fmt` enables number formatting; the header include adds shared page scaffolding.
- The heading displays the searched source and destination; `${flights.size()}` displays how many results were found.
- `<c:choose>` starts the empty-results versus results decision.
- `<c:when test="${empty flights}">` shows the empty-state message and a link back to search if the list is empty.
- `<c:otherwise>` begins the populated-results case.
- `<c:forEach var="f" items="${flights}">` repeats the card once for each flight and names the current item `f`.
- The times, route, flight number, and capacity are read from that flight object's properties.
- `<fmt:formatNumber>` displays the per-seat price with no fraction digits.
- The Book link sends the browser to `/booking/start` and includes the current flight id as a query parameter.
- The footer include closes out the common page structure.

### `WEB-INF/views/signup.jsp`

- The page directive sets UTF-8 and explicitly keeps EL enabled. The `form` and `c` taglibs support Spring-bound fields and conditions.
- The header include creates the top portion of the page.
- The conditional `c:if` shows an error banner only if the controller supplied a nonempty `error` value.
- `<form:form modelAttribute="userDto">` binds fields to the DTO the controller placed in the model and POSTs them to `/signup`.
- The email, password, and role controls each use `path` to name the corresponding DTO property. The password tag masks typed characters.
- Each `<form:errors>` prints validation messages for its field.
- The role select offers blank, `User`, and `Admin` values; the values must match the Java validation rule.
- The final link lets existing users open the login page. The footer include finishes the page.

### `WEB-INF/views/login.jsp`

- The page directive sets UTF-8; the `c` taglib enables conditions; the header include adds shared markup.
- The `registered` query parameter controls a success message after account creation.
- A second `c:if` shows the login error supplied by the controller.
- The HTML form POSTs to `/login`. Its inputs are named `email` and `password`, matching the controller's `@RequestParam` arguments.
- `type="email"` and `type="password"` influence browser input behavior; `required` asks the browser to require a value before submission.
- The submit button sends the credentials. A link opens signup, then the footer closes the page.

### `WEB-INF/views/bookingForm.jsp`

- The directives select UTF-8 and enable JSTL core/format tags; the header include supplies shared page markup.
- The page heading reads route and flight details from the `flight` model object.
- A conditional banner appears when the controller put an `error` into the model.
- The booking HTML form sends a POST to `/booking/confirm`. A hidden input sends `flightId` without showing it as an editable field.
- The date input sends `journeyDate`. Its `min` and `max` come from the controller's date limits; `required` asks the browser not to leave it blank.
- `<c:set>` calculates `seatCap`: the smaller of flight capacity and six. The following `c:forEach` generates a seat-count option from one up through that limit.
- The price field is disabled, so it is displayed but not submitted as editable input. `fmt:formatNumber` formats the price.
- `paxRows` is an initially empty HTML container; JavaScript fills it with passenger inputs. `totalPrice` is another element JavaScript updates.
- The script is an immediately invoked function expression: `(function() { ... })();` runs its contents once while keeping local variables out of the global scope.
- `pricePerSeat`, `seatSelect`, `rows`, and `totalEl` store the price and references to page elements.
- `passengerRow(i)` builds the HTML for one passenger. The index displays a human-friendly passenger number; inputs named `passengerName`, `gender`, and `age` match the controller's request-parameter names. `required`, `min`, and `max` ask the browser to check values.
- `render()` reads the selected seat count, builds one passenger row per seat in a loop, places all rows in the page, and calculates total price. `toFixed(0)` displays a whole-number amount.
- `addEventListener('change', render)` redraws the passenger fields when the selected seat count changes. The final `render()` builds the initial row(s) on page load.
- The footer include closes the page.

### `WEB-INF/views/ticket.jsp`

- The directives enable UTF-8, JSTL conditions/loops, and number formatting; the header include adds shared markup.
- The first sections display the flight route, times, flight number, number of seats, journey date, booking status, and formatted total using values passed by the controller.
- The print button calls the browser's `window.print()` function.
- The cancellation form posts to `/booking/ticket/cancel`; `onsubmit` calls JavaScript before the browser sends it. The hidden `bookingId` identifies the booking.
- `<c:forEach>` renders one passenger row for each ticket in `tickets`.
- `<c:choose>` shows a checkbox for a confirmed ticket. Its submitted name is `pnrNos`, matching the controller's optional list parameter. A cancelled ticket instead shows a placeholder and cannot be selected.
- The remaining expressions print passenger name, gender, age, PNR, seat, and cancellation status.
- `confirmCancel()` counts checked PNR checkboxes. If zero are selected, it displays an alert and returns `false`, stopping form submission. Otherwise it asks for confirmation and returns that answer.
- The footer include closes the page.

### `WEB-INF/views/myBookings.jsp`

- The page directive chooses UTF-8; JSTL core and formatting tags are enabled; the header include supplies the common page frame.
- The heading displays the number of bookings.
- `<c:choose>` shows an empty-state message and search link when there are no bookings, otherwise it enters the list case.
- `<c:forEach var="b" items="${bookings}">` repeats one card per booking summary.
- The route and status are read from `b`. EL's ternary expression chooses a status class depending on whether status is `CANCELLED`.
- The details cells show flight, formatted journey date, times, seats, and total. `fmt:formatNumber` formats the amount.
- The ticket link includes the booking id in the path.
- `<c:if test="${b.cancellable}">` shows a cancellation form only if the Java `isCancellable()` method returns true.
- The form posts the hidden booking id to `/booking/cancel`. Its browser confirmation dialog can cancel submission if the user declines.
- The footer include closes the page.

### `WEB-INF/views/adminDashboard.jsp`

- The directives enable JSTL core, formatting, and string functions. The header include adds shared page elements.
- If `added` is true, the success panel displays the new flight details passed as flash attributes after a successful add.
- `fn:substring(value, 0, 5)` takes the first five characters from a time string for display.
- The dashboard header links to the add-flight form.
- The two tab buttons call `showTab` in JavaScript to switch between upcoming and completed groups.
- For upcoming flights, `<c:choose>` displays an empty message if the list is empty, otherwise `<c:forEach>` repeats a statistics card per item.
- Each item `s` displays times, route, flight number, journey date, booking count, passenger count, revenue, and days remaining. Revenue uses `fmt:formatNumber`.
- The completed section repeats the same pattern with `completed`; it leaves out the days-left value.
- `showTab(id, btn)` chooses which group is visible and toggles the active marker on the two tab buttons. `btn` is accepted but is not used in this function.
- The footer include closes the page.

### `WEB-INF/views/addFlight.jsp`

- The page directive sets UTF-8, the `form` tag library enables Spring form binding, and the header include adds shared page structure.
- `<form:errors path="">` shows form-level errors that are not tied to a single field.
- `<form:form modelAttribute="flightForm">` binds to the `FlightForm` object and POSTs to `/admin/flight/add`.
- Each field group has a label, a Spring input with a `path` matching one DTO property, and `<form:errors>` for that property's validation message.
- The listed fields are flight number, source, destination, departure time, arrival time, capacity, and price per seat. `type="number"` asks the browser for numeric input; `step="0.01"` allows price increments with two decimal places.
- The Add Flight button submits the form; the footer closes the page.

### `WEB-INF/views/error.jsp`

- The page directive sets UTF-8, `c` enables JSTL tags, and the header include supplies common markup.
- The title and message use `<c:out>`, which writes a value as escaped text rather than treating it as HTML. `default` supplies fallback text if the value is absent.
- The two links return to the home page or the user's bookings page.
- The footer include closes the page.

# Reading order for learning the project

1. Read `pom.xml` to learn what libraries and packaging this project uses.
2. Read `db/schema.sql` to understand the records and relationships.
3. Start in Java with `WebAppInitializer`, `AppConfig`, and `WebConfig` to see how the application starts.
4. Follow one user journey: `HomeController` -> `FlightController` -> `FlightService` -> `FlightDao` -> `searchResults.jsp`.
5. Follow booking: `BookingController` -> `BookingService` -> `BookingDao` -> `ticket.jsp`.
6. Read the model/DTO classes after seeing where their values are populated and displayed.
7. Read the JSP pages alongside their controller methods; names in `Model` are the names JSP EL uses.

**CSS is intentionally not included**, as requested.