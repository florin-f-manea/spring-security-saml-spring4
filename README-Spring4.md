Spring SAML update for Spring/Spring Security 4x
================================================

This fork provides some minor changes to Spring SAML extension 1.0.1.SNAPSHOT in order to run it over Spring 4.1.6.RELEASE   and Spring Security 4.0.0.RELEASE (the versions at the moment the fork was done).

##The Problem

Several classes were using deprecated (and finally removed) method **getFilterProcessesUrl()** from their respective base class

* SAMLProcessingFilter extending AbstractAuthenticationProcessingFilter
* SAMLLogoutFilter and SAMLLogoutProcessingFilter extending LogoutFilter


##The Solution

Added two classes (just to define the mising methods and intercept initialization of required string to be returned by **getFilterProcessesUrl()** and changed the above mentioned classes tyo inherit from these ones

```java
package org.springframework.security.saml;

import org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter;

public abstract class AbstractAuthenticationProcessingFilter4 extends AbstractAuthenticationProcessingFilter {

    protected AbstractAuthenticationProcessingFilter4(String defaultFilterProcessesUrl) {
        super(defaultFilterProcessesUrl);
    }
    String filterProcessesUrl;

    public String getFilterProcessesUrl() {
        return filterProcessesUrl;
    }
    @Override
    public void setFilterProcessesUrl(String filterProcessesUrl) {
        this.filterProcessesUrl=filterProcessesUrl;
        super.setFilterProcessesUrl(filterProcessesUrl);
    }

}
```

```java
package org.springframework.security.saml;

public class SAMLProcessingFilter extends AbstractAuthenticationProcessingFilter4 {
/*unmodified*/
...
}
```

##Remarks

**By no means this is provided as a fully tested solution.**

Apart from the changes in code the pom.xml was modified to reflect the dependencies against latest Spring versions. This was done to check the code compiles corectly while for building was used the original gradle approach (unmodified).

Changes required to existing projects:
* Schema reference in security context XML files should reflect the latest version
* Some attributes were removed (like *access-denied-page* in *http* element)
* Add \<security:csrf disabled="true"/\> in *http* elements
* Change existing http elements setting use-expressions="false" or switch to expressions in access atribute values (like replacing deprecated IS_AUTHENTICATED_FULLY with expression based "authenticated"
* Specify obsolete values for password/username ( j_*) POST parameters for forms authentication or change login jsp acordingly

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:security="http://www.springframework.org/schema/security"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
            http://www.springframework.org/schema/security http://www.springframework.org/schema/security/spring-security.xsd
            http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
            
...
    <security:http pattern="/saml/web/**" use-expressions="false">
        <!-- default values for password/user parameters changed in version 4-->
        <security:form-login    login-processing-url="/saml/web/login" 
                                password-parameter="j_password" username-parameter="j_username"
                                login-page="/saml/web/metadata/login" default-target-url="/saml/web/metadata" />
        <security:intercept-url pattern="/saml/web/metadata/login" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
        <security:intercept-url pattern="/saml/web/**" access="ROLE_ADMIN"/>
        <security:custom-filter before="FIRST" ref="metadataGeneratorFilter"/>
        
        <!--security:access-denied-handler error-page="/saml/web/metadata/login" /-->
    </security:http>

    <!-- Secured pages with SAML as entry point -->
    <security:http entry-point-ref="samlEntryPoint">
        <security:intercept-url pattern="/**" access="authenticated"/>
        <security:custom-filter before="FIRST" ref="metadataGeneratorFilter"/>
        <security:custom-filter after="BASIC_AUTH_FILTER" ref="samlFilter"/>
        <security:csrf disabled="true"/>
    </security:http>

```