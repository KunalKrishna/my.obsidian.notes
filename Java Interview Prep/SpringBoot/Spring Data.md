
JDBC --> Spring JDBC --> ORM (Hibernate, TopLink, EclipseLink, OpenJPA) --> Spring Data(JPA )

| Year | Technology                  | Description                                                                                                                                                   |
| ---- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1997 | JDBC                        | JDBC (Java Database Connectivity) standardized direct SQL access and manual resource handling; you wrote SQL and managed connections/results yourself.        |
| 2002 | ORM Concept                 | The object-relational mapping (ORM) approach, mapping Java objects to tables, gained attention in the Java world.                                             |
| 2004 | Hibernate (ORM Library)     | ORM tool that automates the mapping between Java objects and database tables. Developers can work mostly with Java objects.                                   |
| 2004 | Spring JDBC (JdbcTemplate)  | Spring introduced JdbcTemplate, a helper class to make JDBC usage simpler and less error-prone (by automating resource management and exception translation). |
| 2006 | JPA (Java Persistence API)  | Standardized ORM interface for Java; Hibernate became its most popular implementation. Developers now work with **annotations** and **entity classes**.       |
| 2011 | Spring Data (including JPA) | Spring Data introduced **repository abstraction** over JPA, automating and simplifying data access with standard interfaces and out-of-the-box **CRUD**.      |
| 2023 | Spring JdbcClient           | New (in Spring Framework 6.1) fluent API for JDBC queries with named or indexed parameters, flexible mapping, and reduced boilerplate.                        |
## JDBC (Java Database Connectivity)
**Architecture**
is composed of several key components that work together to enable Java applications to interact with relational databases : 
- **Driver Manager**: Handles the list of database drivers and manages the connection b/w your Java application and the database. 
- **Driver**: Implements the actual code required to communicate with a specific database (e.g., MySQL, PostgreSQL). 
- **Connection**: Represents a session with a specific database. You use this object to send SQL statements and receive results. 
- **Statement, PreparedStatement, CallableStatement**: used to execute SQL queries & updates. 
    - `Statement` is used for general SQL queries. 
    - `PreparedStatement` is used for queries with parameters and provides protection against SQL injection. 
    - `CallableStatement` is used to call stored procedures. 
- **ResultSet**: This object holds the data returned from an SQL query and allows navigation and retrieval of results. 
**Summary Table of JDBC Core Classes**: 

| **Class/Interface** | **Purpose**                          |
| ------------------- | ------------------------------------ |
| DriverManager       | Manages JDBC drivers and connections |
| Driver              | Library specific                     |
| Connection          | Represents a database session        |
| Statement           | Executes SQL queries                 |
| PreparedStatement   | Executes parameterized SQL queries   |
| CallableStatement   | Executes stored procedures           |
| ResultSet           | Holds query results                  |

```Java
import java.sql.*;

public class JdbcExample {
    public static void main(String[] args) {
        String url = "jdbc:mysql://localhost:3306/mydb";
        String username = "root";
        String password = "password";

        try {
            // 1. Load the JDBC driver
            Class.forName("com.mysql.cj.jdbc.Driver");

            // 2. Establish the connection
            Connection conn = DriverManager.getConnection(url, username, password);

            // 3. Create a statement
            Statement stmt = conn.createStatement();

            // 4. Execute a query
            ResultSet rs = stmt.executeQuery("SELECT * FROM users");

            // 5. Process the results
            while (rs.next()) {
                System.out.println(rs.getString("name"));
            }

            // 6. Close resources
            rs.close();
            stmt.close();
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
Why Move Beyond JDBC?
- **Manual work**: JDBC requires a lot of manual code—managing connections, crafting SQL by hand, handling transactions, and mapping rows to Java objects.
- **Error-prone**: It’s easy to leak resources or make mistakes with SQL and result set handling.
- **Limited abstraction**: As projects grow, you want to work in terms of objects, not records and SQL strings.
--------------------------

## Spring JDBC/ Jdbc Template
- Still uses JDBC under the hood, but makes resource management and error handling easier.
- Reduces duplication by handling lifecycle and exception translation for you.​
- Typical for developers working with the Spring framework as an entry step to cleaner, more robust database interactions.

```Java
import org.springframework.jdbc.core.*;
import org.springframework.jdbc.core.namedparam.*;
import org.springframework.jdbc.support.*;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;
import javax.sql.DataSource;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.*;

@Repository
public class EmployeeRepository {
    private final JdbcTemplate jdbcTemplate;
    private final NamedParameterJdbcTemplate namedParameterJdbcTemplate;

    public EmployeeRepository(DataSource dataSource) {
        this.jdbcTemplate = new JdbcTemplate(dataSource);
        this.namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(dataSource);
    }

    // RowMapper for Employee
    private RowMapper<Employee> employeeRowMapper = (rs, rowNum) ->
        new Employee(
            rs.getLong("id"),
            rs.getString("first_name"),
            rs.getString("last_name"),
            rs.getString("department"),
            rs.getInt("salary")
        );

    // Create (Insert)
    public long addEmployee(Employee emp) {
        String sql = "INSERT INTO employee (first_name, last_name, department, salary) VALUES (?, ?, ?, ?)";
        KeyHolder keyHolder = new GeneratedKeyHolder();
        jdbcTemplate.update(con -> {
            var ps = con.prepareStatement(sql, new String[]{"id"});
            ps.setString(1, emp.getFirstName());
            ps.setString(2, emp.getLastName());
            ps.setString(3, emp.getDepartment());
            ps.setInt(4, emp.getSalary());
            return ps;
        }, keyHolder);
        return keyHolder.getKey().longValue();
    }

    // Read (Query One)
    public Employee findById(long id) {
        String sql = "SELECT * FROM employee WHERE id = ?";
        return jdbcTemplate.queryForObject(sql, employeeRowMapper, id);
    }

    // Read (Query All)
    public List<Employee> findAll() {
        String sql = "SELECT * FROM employee";
        return jdbcTemplate.query(sql, employeeRowMapper);
    }

    // Update
    public int updateSalary(long id, int salary) {
        String sql = "UPDATE employee SET salary = ? WHERE id = ?";
        return jdbcTemplate.update(sql, salary, id);
    }

    // Delete
    public int deleteById(long id) {
        return jdbcTemplate.update("DELETE FROM employee WHERE id = ?", id);
    }

    // Batch Insert
    public int[] insertBatch(List<Employee> employees) {
        String sql = "INSERT INTO employee (first_name, last_name, department, salary) VALUES (?, ?, ?, ?)";
        return jdbcTemplate.batchUpdate(sql, employees, employees.size(), (ps, emp) -> {
            ps.setString(1, emp.getFirstName());
            ps.setString(2, emp.getLastName());
            ps.setString(3, emp.getDepartment());
            ps.setInt(4, emp.getSalary());
        });
    }

    // Query with Named Parameters
    public List<Employee> findByDepartment(String dept) {
        String sql = "SELECT * FROM employee WHERE department = :dept";
        Map<String, Object> params = Map.of("dept", dept);
        return namedParameterJdbcTemplate.query(sql, params, employeeRowMapper);
    }

    // Transaction management example
    @Transactional
    public void transferSalary(long fromId, long toId, int amount) {
        updateSalary(fromId, -amount); // Subtract from sender
        updateSalary(toId, amount);   // Add to receiver
        // RuntimeExceptions will roll back the transaction
    }

    // Stored Procedure call (briefly)
    public int getSalaryByIdUsingProc(long id) {
        SimpleJdbcCall call = new SimpleJdbcCall(jdbcTemplate)
            .withProcedureName("get_salary_by_id")
            .returningResultSet("out_salary", (rs, rowNum) -> rs.getInt(1));
        Map<String, Object> result = call.execute(Map.of("emp_id", id));
        return (int) result.get("out_salary");
    }
}

// Employee POJO (Simple Example)
class Employee {
    private Long id;
    private String firstName;
    private String lastName;
    private String department;
    private Integer salary;
    // Constructor, getters, setters omitted for brevity
}
```
## ORM Framework (Object-Relation Mapping)
- Oracle's TopLink(1998), **Hibernate**(2004), Apache's OpenJPA(2006), EclipseLink(2008 - JPA 2.x),  
- JPA(2006) : Standardized ORM interface for Java;

Analogy : 
- **JPA is like an Interface** (in Java terms)
- **Hibernate/EclipseLink/OpenJPA are Classes** that implement that interface
## Spring Data


![[springboot CRUD repo.png]]
