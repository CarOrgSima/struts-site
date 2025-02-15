---
layout: default
title: Security
---

# Security
{:.no_toc}

* Will be replaced with the ToC, excluding a header
{:toc}

## Security tips

The Apache Struts 2 doesn't provide any security mechanism - it is just a pure web framework. Below are few tips 
you should consider during application development with the Apache Struts 2.

### Restrict access to the Config Browser Plugin

[Config Browser Plugin](../plugins/config-browser/) exposes internal configuration and should be used only during 
development phase. If you must use it on production site, we strictly recommend restricting access to it - you can use  
Basic Authentication or any other security mechanism (e.g. [Apache Shiro](https://shiro.apache.org/))

### Don't mix different access levels in the same namespace

Very often access to different resources is controlled based on URL patterns, see snippet below. Because of that 
you cannot mix actions with different security levels in the same namespace. Always group actions in one namespace 
by security level.

```xml
<security-constraint>
    <web-resource-collection>
        <web-resource-name>admin</web-resource-name>
        <url-pattern>/secure/*</url-pattern>
    </web-resource-collection>
    <auth-constraint>
        <role-name>admin</role-name>
    </auth-constraint>
</security-constraint>
```

### Never expose JSP files directly

You must always hide JSP file behind an action, you cannot allow for direct access to the JSP files as this can leads 
to unpredictable security vulnerabilities. You can achieve this by putting all your JSP files under the `WEB-INF` folder 
- most of the JEE containers restrict access to files placed under the `WEB-INF` folder. Second option is to add security 
constraint to the `web.xml` file:

```xml
<!-- Restricts access to pure JSP files - access available only via Struts action -->
<security-constraint>
    <display-name>No direct JSP access</display-name>
    <web-resource-collection>
        <web-resource-name>No-JSP</web-resource-name>
        <url-pattern>*.jsp</url-pattern>
    </web-resource-collection>
    <auth-constraint>
        <role-name>no-users</role-name>
    </auth-constraint>
</security-constraint>

<security-role>
    <description>Don't assign users to this role</description>
    <role-name>no-users</role-name>
</security-role>
```

The best approach is to used the both solutions.

### Disable devMode

The `devMode` is a very useful option during development time, allowing for deep introspection and debugging into you app.

However, in production it exposes your application to be presenting too many informations on application's internals 
or to evaluating risky parameter expressions. Please **always disable** `devMode` before deploying your application 
to a production environment. While it is disabled by default, your 
`struts.xml` might include a line setting it to `true`. The best way is to ensure the following setting is applied 
to our `struts.xml` for production deployment:

```xml
<constant name ="struts.devMode" value="false" />
```

### Reduce logging level

 It's a good practice to reduce logging level from **DEBUG** to **INFO** or less. Framework's classes can produce 
 a lot of logging entries which will pollute the log file. You can even set logging level to **WARN** for classes that 
 belongs to the framework, see example Log4j2 configuration:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
    <Appenders>
        <Console name="STDOUT" target="SYSTEM_OUT">
            <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
        </Console>
    </Appenders>
    <Loggers>
        <Logger name="com.opensymphony.xwork2" level="warn"/>
        <Logger name="org.apache.struts2" level="warn"/>
        <Root level="info">
            <AppenderRef ref="STDOUT"/>
        </Root>
    </Loggers>
</Configuration>
```

### Use UTF-8 encoding

Always use `UTF-8` encoding when building an application with the Apache Struts 2, when using JSPs please add the following 
header to each JSP file

```jsp
<%@ page contentType="text/html; charset=UTF-8" %>
```

### Do not define setters when not needed

You should carefully design your actions without exposing anything via setters and getters, thus can leads to potential 
security vulnerabilities. Any action's setter can be used to set incoming untrusted user's value which can contain 
suspicious expression. Some Struts `Result`s automatically populate params based on values in 
`ValueStack` (action in most cases is the root) which means incoming value will be evaluated as an expression during 
this process.

### Do not use incoming values as an input for localisation logic

All `TextProvider`'s `getText(...)` methods (e.g. in`ActionSupport`) perform evaluation of parameters included in a message 
to properly localize the text. This means using incoming request parameters with `getText(...)` methods is potentially 
dangerous and should be avoided. See example below, assuming that an action implements getter and setter for property 
`message`, the below code allows inject an OGNL expression:

```java
public String execute() throws Exception {
    message = getText(getMessage());
    return SUCCESS;
}
```

**Never use value of incoming request parameter as part of your localization logic.**

### Do not use incoming, untrusted user input in forced expression evaluation

You can use a forced expression evalaution in many tags' attributes by using `%{...}` syntax. This is a very powerful option
but used with wrong data can lead to the Remote Code Execution. Never use forced expression evalaution if you didn't verify
the input or it can be passed in by a user.

**Never use value of incoming request parameter as input for forced expression evalaution.**

### Use Struts tags instead of raw EL expressions

JSP EL doesn't perform any kind of escaping, you must perform this using a dedicated function, see [this example](https://stackoverflow.com/a/6135001/1805267).
Never use a raw `${}` EL expression on incoming values as this can lead to injecting a malicious code into the page.

The safest option is to use Struts Tags instead.

### Define custom error pages

As mentioned in [S2-006](https://cwiki.apache.org/confluence/display/WW/S2-006) it's a good practicse to define your own 
error pages. This avoids exposing users to XSS attacks as Struts does not escape action's names in automatically 
generated error pages.

You can either disable [DMI](../core-developers/action-configuration#dynamic-method-invocation)

```xml
<constant name="struts.enable.DynamicMethodInvocation" value="false" />
``` 

or define an error page

```xml
<global-results>
  <result name="error">/error_page.jsp</result>
</global-results>
 
<global-exception-mappings>
  <exception-mapping exception="java.lang.Exception" result="error"/>
</global-exception-mappings>
```

### Ambiguous Action Methods

This can impact actions which have large inheritance hierarchy and use the same method's name throughout the hierarchy, 
this was reported as an issue [WW-4405](https://issues.apache.org/jira/browse/WW-4405). See the example below:

```java
public class RealAction extends BaseAction {  
    @Action("save")
    public String save() throws Exception {
        super.save();
        return SUCCESS;
    }
}

public class BaseAction extends AbstractAction {
    public String save() throws Exception {
        save(Double.MAX_VALUE);
        return SUCCESS;
    }
}

public abstract class AbstractAction extends ActionSupport {
    protected void save(Double val) {
        // some logic
    }
}
```

In such case OGNL cannot properly map which method to call when request is coming. This is do the OGNL limitation. 
To solve the problem don't use the same method's names through the hierarchy, you can simply change the action's method 
from `save()` to `saveAction()` and leaving annotation as is to allow call this action via  `/save.action` request.

### Accepted / Excluded Patterns

As from version 2.3.20 the framework provides two new interfaces which are used to accept / exclude param names 
and values - [AcceptedPatternsChecker](../maven/struts2-core/apidocs/com/opensymphony/xwork2/security/AcceptedPatternsChecker) 
and [ExcludedPatternsChecker](../maven/struts2-core/apidocs/com/opensymphony/xwork2/security/ExcludedPatternsChecker) 
with default implementations. These two interfaces are used by the [Parameters Interceptor](../core-developers/parameters-interceptor) 
and [Cookie Interceptor](../core-developers/cookie-interceptor) to check if param can be accepted or must be excluded. 
If you were using `excludeParams` previously please compare patterns used by you with these provided by the framework in default implementation.

### Strict Method Invocation

This mechanism was introduced in version 2.5. It allows control what methods can be accessed with the bang "!" operator 
via [Dynamic Method Invocation](../core-developers/action-configuration.html#dynamic-method-invocation). Please read 
more in the Strict Method Invocation section of [Action Configuration](../core-developers/action-configuration).

### Resource Isolation Using Fetch Metadata

> Note: since Struts 6.0.0

Fetch Metadata is a mitigation against common cross-origin attacks such as Cross-Site Request Forgery (CSRF). It is 
a web platform security feature designed to help servers defend themselves against cross-origin attacks based 
on the preferred resource isolation policy. The browser provides information about the context of an HTTP request 
in a set of `Sec-Fetch-*` headers. This allows the server processing the request to make decisions on whether the request 
should be accepted or rejected based on the available resource isolation policies.

A Resource Isolation  Policy prevents the resources on a server from being requested by external websites. This policy 
can be enabled for all endpoints of the application or the endpoints that are meant to be loaded in a cross-site context 
can be exempted from applying the policy. Read more about Fetch Metadata and resource isolation [here](https://web.dev/fetch-metadata/).

This mechanism is implemented in Struts using the [FetchMetadata Interceptor](../core-developers/fetch-metadata-interceptor).
 Refer to the documentation for [FetchMetadata Interceptor](../core-developers/fetch-metadata-interceptor) 
 instructions on how to enable Fetch Metadata. 

### Cross Origin Isolation with COOP and COEP

> Note: since Struts 6.0.0

[Cross-Origin Opener Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy) is 
a security mitigation that lets developers isolate their resources against side-channel attacks and information leaks. 
The COOP response header allows a document to request a new browsing context group to better isolate itself from other 
untrustworthy origins.

[Cross-Origin Embedder Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy) 
prevents a document from loading any cross-origin resources which don't explicitly grant the document permission to be loaded. 

COOP and COEP are independent mechanisms that can be enabled, tested and deployed separately. While enabling one doesn’t 
require developers to enable the other, when set together COOP and COEP allows developers to use powerful features (such 
as `SharedArrayBuffer`, `performance.measureMemory()` and the JS Self-Profiling API) securely, without worrying about 
side channel attacks like [Spectre](https://meltdownattack.com/). 
Further reading on [COOP/COEP](https://docs.google.com/document/d/1zDlfvfTJ_9e8Jdc8ehuV4zMEu9ySMCiTGMS9y0GU92k/edit#bookmark=id.uo6kivyh0ge2) 
and [why you need cross-origin isolation](https://web.dev/why-coop-coep/).

The recommended configuration for the policies are:

```
Cross-Origin-Embedder-Policy: require-corp;
Cross-Origin-Opener-Policy: same-origin;
```

COOP and COEP are implemented in Struts using [CoopInterceptor](../core-developers/coop-interceptor) 
and [CoepInterceptor](../core-developers/coep-interceptor).

## Proactively protecting against OGNL Expression Injections attacks

The framework has a history of critical security bugs, many tied to its use of OGNL technology; Due to its ability to
create or change executable code, OGNL is capable of introducing critical security flaws to any framework that uses it.
Multiple Struts 2 versions have been vulnerable to OGNL security flaws. Consequently, we've equipped OGNL and the
framework with a number of additional security capabilities, some of which need to be manually enabled.

> **NOTE**: These might break your current app functionality. Before using in production environment, you're recommended
> to comprehensively test your app UI and functionalities with these enabled.

### Run OGNL expressions inside sandbox

You can do this simply via adding `-Dognl.security.manager` to JVM arguments. OGNL thereupon utilizes Java Security
Manager to run OGNL expressions (which includes your actions either!) inside a sandbox with no permission. It is worth
noting that it affects only OGNL expression execution and thereafter OGNL reverts Java Security Manager to its previous
state.

Note: This feature does not work with JDK 21 and above.

### Apply a maximum allowed length on OGNL expressions

You can enable this via Struts configuration key `struts.ognl.expressionMaxLength` (defaults to 256). OGNL thereupon doesn't evaluate any
expression longer than specified value. You would choose a value large enough to permit ALL valid OGNL expressions used
within the application. Values larger than the 200-400 range have diminishing security value (at which point it is
really only a "style guard" for long OGNL expressions in an application).

### OGNL Member Access

Struts 2 implements an OGNL internal security mechanism which blocks access to particular classes and Java packages -
it's an OGNL-wide mechanism which means it affects any aspect of the framework i.e. incoming parameters, expressions
used in JSPs, etc. Matching is done based on both the target and member class of all components of an OGNL expression.

There are 4 options that can be used to configure excluded packages and classes:

- `struts.excludedClasses`: comma-separated list of excluded classes.
- `struts.excludedPackageNames`: comma-separated list of excluded packages, matched using string comparison via
  `startWith`. Note that classes in subpackages are also excluded.
- `struts.excludedPackageNamePatterns` - comma-separated list of RegEx patterns used to exclude packages. Note that this
  option is more flexible than `struts.excludedPackageNames` but will have a greater impact on performance and page 
  latency.
- `struts.excludedPackageExemptClasses` - comma-separated list of classes to exempt from any of the excluded packages or
  package name patterns. An exact exemption must exist for each exclusion match (target or member or both).

The defaults are defined [here](https://github.com/apache/struts/blob/master/core/src/main/resources/struts-excluded-classes.xml).

Any expression or target which does not pass this criteria will be blocked, and you will see a warning in the logs:

```
[WARNING] Target class [class example.MyBean] or declaring class of member type [public example.MyBean()] are excluded!
```

In that case `new MyBean()` was used to create a new instance of class (inside JSP) - it's blocked because the `target`
of such expression is `java.lang.Class` which is excluded.

It is possible to redefine the above constants in `struts.xml`, but avoid reducing the list, instead extending the list
with other known dangerous classes or packages in your application.

#### Additional Options

We additionally recommend enabling the following options and hope to enable them by default in a future major version.

 * `struts.ognl.allowStaticFieldAccess=false` - static methods are always blocked, but static fields can also optionally be blocked
 * `struts.disallowProxyMemberAccess=true` - disallow proxied objects from being used in OGNL expressions as they may present a security risk
 * `struts.disallowDefaultPackageAccess=true` - disallow access to classes in the default package which should not be used in production
 * `struts.ognl.disallowCustomOgnlMap=true` - disallow construction of custom OGNL maps which can be used to bypass the SecurityMemberAccess policy

#### Allowlist Capability

> Note: since Struts 6.4.

For even more stringent OGNL protection, we recommend enabling the allowlist capability with `struts.allowlist.enable`.

Now, in addition to enforcing the exclusion list, classes involved in OGNL expression must also belong to a list of
allowlisted classes and packages. By default, all required Struts classes are allowlisted as well as any classes that
are defined in your `struts.xml` package configurations.

You can add additional classes and packages to the allowlist with:

- `struts.allowlist.classes`: comma-separated list of allowlisted classes.
- `struts.allowlist.packages`: comma-separated list of allowlisted packages, matched using string comparison via
  `startWith`. Note that classes in subpackages are also allowlisted.

Generally, the only additional classes or packages you will need to configure are those model classes that you wish to
be constructed/manipulated by Struts form submissions (i.e. parameter injected).

#### Extensibility

> Note: since Struts 6.4.

The OGNL Member Access mechanism is extensible, allowing you to define your own rules for blocking access to OGNL
expression evaluations. To do so, you may use the `struts.securityMemberAccess` extension point. Please be vigilant when
overriding methods as not to reduce protections offered by the default implementation.

### Struts OGNL Guard

> Note: since Struts 6.4.

The Struts OGNL Guard allows applications to completely disable certain OGNL expression features/capabilities. This
feature is disabled by default but can be enabled and configured with `struts.ognl.excludedNodeTypes`.

It is recommended to disable any OGNL feature you are not leveraging in your application. For applications using a
minimal number of Struts features, you may find the following list a good starting point.

Please be aware that this list WILL break certain Struts features:

```xml
<constant name="struts.ognl.excludedNodeTypes"
          value="
            ognl.ASTAdd,
            ognl.ASTAssign,
            ognl.ASTBitAnd,
            ognl.ASTBitNegate,
            ognl.ASTBitOr,
            ognl.ASTCtor,
            ognl.ASTDivide,
            ognl.ASTEval,
            ognl.ASTIn,
            ognl.ASTInstanceof,
            ognl.ASTKeyValue,
            ognl.ASTList,
            ognl.ASTMap,
            ognl.ASTMultiply,
            ognl.ASTNegate,
            ognl.ASTNotIn,
            ognl.ASTProject,
            ognl.ASTRootVarRef,
            ognl.ASTSelect,
            ognl.ASTSelectFirst,
            ognl.ASTSelectLast,
            ognl.ASTSequence,
            ognl.ASTShiftLeft,
            ognl.ASTShiftRight,
            ognl.ASTStaticField,
            ognl.ASTStaticMethod,
            ognl.ASTThisVarRef,
            ognl.ASTUnsignedShiftRight,
            ognl.ASTVarRef,
            ognl.ASTXor
"/>
```

#### Extensibility

The Struts OGNL Guard mechanism is extensible, allowing you to define your own rules for blocking access to both raw
OGNL expressions and compiled syntax trees. To do so, you may use the `struts.ognlGuard` extension point. You may choose
to override the default implementation or implement the `OgnlGuard` interface directly.
