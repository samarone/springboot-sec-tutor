# springboot-sec-tutor
---
## Introduction
This guide covers the security concerns about RESTful-backed applications. Spring boot is used to show example backend implementations of these concerns. Client implementations are not covered, only described. Furthermore guide covers browser defenses that should be implemented.

## Transport Layer
First to think about is whether to use SSL and for what services to use it (e.g. static resources). SSL should always be used when your application involves authentication. Hence at least your authentication process and the restricted part of the application should be restricted to SSL (HTTPS) connections only. The reason is pretty obvious: Prevention of account- and session-hijacking through man-in-the-middle attacks.

As you will probably always require SSL for at least some services of your application. The question is where to implement it. Therefore there are some examples with pros and cons:

### HTTP Server (apache, nginx, ...)
In most cases the SSL implementing part is the HTTP server you front your application with. Be sure to use the latest OpenSSL version!
#### Pros
- Easy implementation
- Configuration
- Scalable
- Most up-to-date SSL implementations

#### Cons
- Not truly end-to-end (The traffic between HTTP server and your application (servlet container) might be unencrypted)

### Servlet Container (Tomcat, Jetty, ...)
This is unusual but can be done. Example implementation and current secure configuration using embedded tomcat can be found in this repository.
#### Pros
- Truly end-to-end encryption

#### Cons
- Configuration. Java SSL implementation (not easily or not configurable at all, e.g. custom DHparams in Java 1.8)
- Keystore file instead of pem/crt/key files
- Scalability complicated (extra Loadbalancer required. Also certificate must be n-times deployed)

### Cloud (AWS) *cloud enviroment*
Mostly the SSL is terminated on the Loadbalancer. AWS lets you create signed certificate (in ACM) and add it to your LB/CloudFront or upload your own signed certificate to IAM and refer it in your LB/CloudFront.
#### Pros
- Easy SSL implementation
- Scalable

#### Cons
- Not truly end-to-end if terminated on LB (see [http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/configuring-https-endtoend.html])
- Configuration. You can choose from predefined set of SSL Security Policies or configure it via command line or JSON (see [https://mozilla.github.io/server-side-tls/ssl-config-generator/] and choose AWS LB)

### SSL configuration
Now that the implementing part is known, the question is what to consider a secure SSL config. Therefore you want to check the following sites:
- [https://mozilla.github.io/server-side-tls/ssl-config-generator/] suggesting to use modern configuration (see for details and configure ciphers as wanted/required [https://wiki.mozilla.org/Security/Server_Side_TLS])
- [https://weakdh.org/] if you use DH cipher then you should consider creating your own DHparams since the default are weak (this is only required if you use a DHE- cipher not for ECDHE- therefore you don't have to do this if you use 'modern' config above). Also check [https://weakdh.org/sysadmin.html] for creating/deploying customn DHparams.pem to your HTTP server.
- [https://drownattack.com/] disable SSLv2 since its vulnerable to the DROWN attack. Instructions for some HTTP server can be found on the site.
- [https://en.wikipedia.org/wiki/POODLE] disable SSLv3 since its vulnerable to the POODLE attack.
- [https://en.wikipedia.org/wiki/CRIME_(security_exploit)] disable SSL compression since it is vulnerable to th CRIME attack (and others). For confiuration refer to your HTTP server documentation.
- [http://breachattack.com/] disable HTTP compression since it is vulnerable to oracle attacks (e.g. BREACH). This is mostly default in the Servlet Containers. Don't enable it!

For apache documentation refer to:
- [http://httpd.apache.org/docs/trunk/ssl/ssl_howto.html] example apache ssl configuration (including TSLv1.2 only, SSLCompression, SSLHonorCipherOrder, SSLSessionTickets)

For AWS LB HTTPS configuration refer to:
- [http://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-add-or-delete-listeners.html] how to add a certificate to you LB
- [http://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-ssl-security-policy.html] ssl configuration

### SSL security check **Important**!
Once the configuration is done and you think you did everything right, its time to face the truth. Deploy your app and check your ssl configuration with [https://www.ssllabs.com/ssltest/]. Fix configuration until you have at least an A mark (or better A+).

## Authentication (who you are)
Most applications need authorization (what is who allowed to do). Authentication is precondition for authorization since you need to make sure who the application is interacting with. 
There are different ways to implement authentication in spring boot, each having their pros and cons.

### Fromlogin (Cookie Session-ID)
The most common and easiest to use solution. Implementation can be found in the master branch. The relevant part is the code in the `WebSecurityConfigurerAdapter`

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
            .authorizeRequests()
            .anyRequest().authenticated()
            .and()
            .formLogin()
            .and()
            .logout().permitAll();
}
```

The `formLogin()` does it all. This means that the form will be rendered by spring boot (default login template). If you want to have another template you can specify it with `loginPage("/login")` and have a View "login" configured (see [http://docs.spring.io/spring-security/site/docs/current/guides/html5/form-javaconfig.html#configuring-a-custom-login-page] for details).
If you plan to use the spring boot app only as backend then I recommend to use **REST Authentication Login**. Why? Well because else it results in having to deal with the CSRF-TOKEN by yourself.
**DO NOT DISABLE CSRF IF YOU HAVE A FORM LOGIN LIKE THIS**

#### Pros
- Easy to configure (everything implemented by spring)

#### Cons
- Requires spring to render CSRF-Token or handle defense manually

### REST Authentication Login (Cookie Session-ID)
This is recommended when you use spring boot as a backend service only. An example implementation can be found in the [rest-auth](https://github.com/Zuehlke/springboot-sec-tutor/tree/rest-auth) branch. It requires some caution when implementing it.
Meaning CSRF prevention may have to be dealt with by yourself.

#### CSRF prevention
There are more than one strategy that you can choose from to prevent CSRF (see [https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet]). Spring uses the "Synchronizer Token" per default. This is also used in the Formlogin above. But since we can't render the Token with spring into a form we must find another way to prevent CSRF attacks.

The easiest way for that is to use spring's `CookieCsrfTokenRepository` with `csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())` which implements the "Double Submit Cookie" strategy and works as default with AngularJS and therefore is a really nice solution. Be aware that your frontend and backend must have the same Origin, if you have different Origins for the frontend and backend the frontend will not be able to read the backend Cookies through `document.cookie` even if you configured CORS. The Cookies will follow SOP (Same-origin Policy) (see [https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/withCredentials]).
If the Origins differ you have to correctly configure CORS (see HTTP Security Headers) and implement a custom `CsrfTokenRepository` or your own `@RestController` (with e.g. /hello) which provides the Session-Token in a Header (but only in CORS-aware requests! Else you are not safe at all). 

In the branch [rest-auth](https://github.com/Zuehlke/springboot-sec-tutor/tree/rest-auth) there is an implementation of "having a custom header" which need some Filters to be implemented but fits pretty good for a pure backend (no token handling and Origin of frontend doesn't have to be the same).
For this we have to carefully configure CORS (see HTTP Security Headers) first. It needs to be done anyway to make your REST backend work with your frontend if the Origins differ!
When we know CORS is configured correctly we need to make sure that all the request are not simple requests. Relying on CORS we know that if a custom header is present (e.g. X-Requested-With) the browser will either not make the response accessible or will preflight the request (for details see [https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS] and [https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet#Protecting_REST_Services:_Use_of_Custom_Request_Headers]).
To ensure browser's requests are CORS-aware this Header is required to be present for every request and therefore need a custom Filter `XRequestedWithHeaderFilter`:

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
    String header = request.getHeader("X-Requested-With");
    if(header == null || header.isEmpty()){
        response.sendError(HttpServletResponse.SC_BAD_REQUEST,"X-Requested-With Header is not present or empty");
    } else {
        filterChain.doFilter(request, response);
    }
}

```
On the client side the header has to be added to every AJAX request. In AngularJS you can use [$httpProvider#defaults](https://code.angularjs.org/1.4.10/docs/api/ng/provider/$httpProvider#defaults) for that.
The OWASP article suggests to check Origin and Referer Header to prevent some exotic attacks with Flash. The Origin header is already checked (if present) by spring CORS handling and in case if it doesn't match the request is rejected (controller code not executed). Since in our case all request (done by instances of XMLHttpRequest) should have an Origin header we are going to implement a Filter, that denies all requests without the Origin header, called `EnforceCorsFilter`:

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
    if (!CorsUtils.isCorsRequest(request)) {
        response.sendError(HttpServletResponse.SC_BAD_REQUEST, "Not a CORS request (Origin header is missing)");
    } else {
        filterChain.doFilter(request, response);
    }
}
```

Now disable the default CSRF handling with `csrf().disable()` and add the Filters to the FilterChain in your `WebSecurityConfigurerAdapter` at the right position (before CsrfFilter is just fine).

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
            .authorizeRequests()
            .anyRequest().authenticated()
            .and()
            .csrf().disable()
            .addFilterBefore(new XRequestedWithHeaderFilter(), CsrfFilter.class)
            .addFilterBefore(new EnforceCorsFilter(), CsrfFilter.class);
}
```

#### Custom authentication (REST) service

Now we should be save against CSRF attacks. But we haven't implemented the authentication service yet. This doesn't rely on one of the implemented "CSRF prevention".

We will use `HttpStatusEntryPoint` as `AuthenticationEntryPoint` and implement our own `AbstractAuthenticationProcessingFilter`. For the Filter there is a `ObjectMapper` required for deserialization `LoginRequestPOJO` of the login request's JSON body. Furthermore spring's `UsernamePasswordAuthenticationToken` is used to represent the `Authentication`.
The authentication has to be done with a POST request to /login. 

```java
public class RESTAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    private final ObjectMapper objectMapper;

    public RESTAuthenticationFilter(ObjectMapper objectMapper) {
        super(new AntPathRequestMatcher("/login", "POST"));
        this.objectMapper = objectMapper;
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException, IOException, ServletException {
        LoginRequestPOJO loginRequestPOJO;
        try {
            loginRequestPOJO = objectMapper.readValue(request.getReader(), LoginRequestPOJO.class);
        } catch (JsonMappingException | JsonParseException e) {
            throw new AuthenticationServiceException("Invalid login request");
        }

        UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
                loginRequestPOJO.getUsername(), loginRequestPOJO.getPassword());

        authRequest.setDetails(authenticationDetailsSource.buildDetails(request));

        return this.getAuthenticationManager().authenticate(authRequest);
    }
}
```
The default login success behavior of `AbstractAuthenticationProcessingFilter` is `SavedRequestAwareAuthenticationSuccessHandler` (which returns 302 Found for successfull logins) is replaced with a custom (200 OK) implementation of `AuthenticationSuccessHandler`:

```java
@Component
public class RESTAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        response.setStatus(HttpServletResponse.SC_OK);
    }
}
``` 

You might also want to change the default logout behavior which also returns 302 for successfull logins. We can do this pretty simple with `logout().logoutSuccessHandler((request, response, authentication) -> response.setStatus(HttpServletResponse.SC_OK))` in the `WebSecurityConfigurerAdapter`.

Well all of this doesn't yet make sense if it is not wired together somehow. This is done in `WebSecurityConfigurerAdapter`:

```java
@Autowired
private RESTAuthenticationSuccessHandler restAuthenticationSuccessHandler;

@Autowired
private AuthenticationManager authenticationManager;

@Autowired
private ObjectMapper objectMapper;

@Bean
public RESTAuthenticationFilter restAuthenticationFilter() {
    RESTAuthenticationFilter restAuthenticationFilter = new RESTAuthenticationFilter(objectMapper);
    restAuthenticationFilter.setAuthenticationManager(authenticationManager);
    restAuthenticationFilter.setAuthenticationSuccessHandler(restAuthenticationSuccessHandler);
    return restAuthenticationFilter;
}

@Override
protected void configure(HttpSecurity http) throws Exception {
    http
            .authorizeRequests()
            .anyRequest().authenticated()
            .and().exceptionHandling().authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
            .and().anonymous().disable()
            .csrf().disable()
            .addFilterBefore(new XRequestedWithHeaderFilter(), CsrfFilter.class)
            .addFilterBefore(new EnforceCorsFilter(), CsrfFilter.class)
            .addFilterBefore(restAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)
            .logout().logoutSuccessHandler((request, response, authentication) -> response.setStatus(HttpServletResponse.SC_OK));
}
```

#### Pros
- No form rendering
- No Token handling
- Frontends can have different Origins

#### Cons
- No default implementation by spring

That's it for the custom Authentication.

Be careful when implementing your REST services. If you for example change state with a GET, HEAD or OPTIONS request the security may be compromised (this also applies to non-persistent state like session-state).

If you don't want to use Cookies as session identifier store you have to either include spring-session (see [http://docs.spring.io/spring-session/docs/current/reference/html5/guides/boot.html]) and e.g. use HeaderHttpSessionStrategy or you could implement another AuthenticationFilter to check the request for the valid Token (you might also think about disabling the creation of sessions with `sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)` if you have a custom AuthenticationFilter and don't need any other server (session) state). But be careful since cookies have securing mechanisms like secure, httpOnly, max-age etc. which you can't enforce when using another browser persistency mechanism.

### JWT
We first have to look into JWT a bit. JWT is a signed Token and therefore can be checked by the signing party if it has originally signed it. 
So when we use JWT for Authentication the server issues signed tokens and sends it to the client (browser) which has to store it. The server normally can forget about the issued tokens. Everytime when the client now sends this JWT token with a request, the server can check if it has issued this token. But this means the complete session state is transfered with every request.
Also if the server forgets about the issued tokens then they can't be revoked. You can find a lot of example implementations on the internet. **But** there is no good reason to use JWT's as session mechanism ([check out this article reasoning why](http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/)).
We agree with this reasoning and therefore will not cover an example implementation but only describe an example of a reasonable use case: *Single-use authorization token*: The actual authentication has nothing to do with JWTs.
The use case could look like this:
As a registered and authenticated user I want to issue an invitation so that another user can register. The given roles at the registration have to be fixed by the registered user.

Having a JWT here is convenient: It contains the claims (right to register, roles) and an expiration date. The server can completely forget about this token (no state required). The token should be short-lived. To make sure it is only used once the email or another user-unique value could be saved in the token.

---

## Authorization (what are you allowed to do)
There are different ways to implement authorization. Here we will cover the role based authorization. If you want to have another type of access controll (authorization) then you can either implement your own `AccessDecisionManager` or an `AccessDecisionVoter` (see [http://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#tech-intro-access-control]).

We can control which role can invoke which methods or call which URLs.
The implementation of a domain model containing `Role` can be found in this repository. In our case users can have multiple roles and therefore is modeled as a many-to-many relationship and our role model is not hierarchical. Therefore one role (e.g. ADMIN) doesn't contain another role (e.g. USER).

There are two ways to restrict the access for a specific role (or multiple roles):
- Spring security configuration `WebSecurityConfigurerAdapter`
	```java
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http
			.authorizeRequests()
			.antMatchers("/users/**").hasRole("USER")
			.and()
			// ....
	}
	``` 
	This will restrict the access for URL /users/** to users with the "ROLE_USER" authority. Be aware that this is invoked (and therefore blocked) before any of the Controllers code and even maybe some of your Filters. So you can't generally restrict access here and relax it somewhere else.
- Method based access control (Annotations)
	You have to enable global method security with the annotation `@EnableGlobalMethodSecurity(prePostEnabled = true)` in one of your `@Configuration`s (preferably in `WebSecurityConfigurerAdapter`). After that you can use `@PreAuthorize` or `@PostAuthorize` on class- and method-level with Spring EL to control access. This controller is restricted to users with "ROLE_USER" but it contains a method that is restricted for users with "ROLE_ADMIN" only.
	```java
	@RestController
	@RequestMapping("/users")
	@PreAuthorize("hasRole('USER')")
	public class UserController {
    		@Autowired
    		private UserService userService;

    		@PreAuthorize("hasRole('ADMIN')")
    		public List<UserDTO> getUsers() {
		// ...
		}
	}
	```


---

## Data storage
*TODO*

---

## HTTP Security Headers (Browsers defenses)
*TODO*
