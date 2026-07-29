Home {
id
area
field 2
field 3

Home(id, .... all 3 param) {}
Home(all 3 param) {}
Home() {}

}

/api/home Method.POST
- 201 if created
- 200 if already existing

/api/home/{id} Method.GET
- 404 if not found
- 401 if invalid request eg. {id} = -1 or {id}={string}
- 200 if found

/api/home/  Method.GET ALL


Spring Security : admin:admin
Restrict a URL to Specific Roles : missing in the code
```java
.authorizeHttpRequests(auth -> auth .requestMatchers("/admin/**").hasRole("ADMIN") .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN") .requestMatchers("/public/**").permitAll() .anyRequest().authenticated() )
```

4 Test code  : 
Error : IllegalArgumentException : No PasswordEncoder for id = null
