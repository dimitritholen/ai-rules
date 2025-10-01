# React 19 Security Best Practices (2025)

## XSS Prevention Patterns

### Default Security Mechanisms
- React's JSX syntax automatically escapes values when using curly braces `{}`, making dynamic text rendering safe against XSS by default
- Always use default data binding with `{}` to leverage React's automatic escaping
- Never concatenate unsanitized data onto Server-Side Rendering (SSR) output
- When embedding JSON for hydration, always escape the `<` character in your JSON to avoid breaking out of the script context

### dangerouslySetInnerHTML Usage
- **CRITICAL**: `dangerouslySetInnerHTML` bypasses React's built-in XSS protection by directly injecting HTML into the DOM
- Never use `dangerouslySetInnerHTML` without sanitization
- Always sanitize user-generated content with DOMPurify before using `dangerouslySetInnerHTML`
- Encapsulate sanitization in a security component:

```typescript
import DOMPurify from 'dompurify';

function SafeHtmlRenderer({ htmlContent }: { htmlContent: string }) {
  const sanitizedContent = DOMPurify.sanitize(htmlContent);
  return <div dangerouslySetInnerHTML={{ __html: sanitizedContent }} />;
}
```

### DOMPurify Best Practices
- Use DOMPurify v3.2.7+ which supports Node.js v18.x through v23.x
- DOMPurify employs a whitelist-based strategy, allowing only specified safe elements and attributes
- Configure DOMPurify with `RETURN_TRUSTED_TYPE: true` in environments where the Trusted Types API is available
- Always sanitize user inputs before rendering them into UI

### URL Validation
- Links and URLs in props can be attack vectors if they use the `javascript:` or `data:` schemes
- Only allow `http:` or `https:` protocols
- Use native URL parsing functionality to validate URLs before rendering
- Validate URLs from users or external sources to prevent script injections and malicious redirects

### Direct DOM Manipulation
- Avoid accessing the DOM to inject content into DOM nodes directly
- If HTML must be injected, use `dangerouslySetInnerHTML` only after sanitizing with DOMPurify
- Never use `innerHTML`, `outerHTML`, or similar direct DOM APIs without sanitization

## Authentication and Authorization Patterns

### Context API for Authentication
- Use Context API for managing user authentication state across the application
- Create an Ability Context for authorization operations
- Use the Provider Pattern to pass authentication context to all components

### Role-Based Access Control (RBAC)
- Implement RBAC to control access to different parts of the application based on user roles
- Use authentication handlers to manage authentication and authorization logic
- Consider React CASL for performing authorization operations

### Modern Authentication Standards
- Implement OAuth 2.0 for authorization
- Use JWT (JSON Web Tokens) for stateless authentication
- Implement Multi-Factor Authentication (MFA) to prevent unauthorized access
- Use phishing-resistant MFA on all developer accounts

### Token Management
- Mark sensitive cookies like session tokens as `HttpOnly` to prevent JavaScript access
- Use secure, SameSite cookies to prevent CSRF attacks
- Implement proper token rotation and expiration policies
- Store tokens securely, never in localStorage for sensitive data

### React 19 Async Transitions
- React 19 adds support for using async functions in transitions
- Handle pending states, errors, forms, and optimistic updates automatically with React 19's new features
- Leverage Server Components for authentication to keep sensitive logic server-side

## Secure Data Fetching

### Server-Side Rendering Security
- Use Server Components to fetch data on the server, keeping applications more secure
- Server-side data fetching prevents sensitive information like access tokens and API keys from being exposed to the client
- Never embed API keys or secrets in client-side code

### Data Fetching Libraries
- Use community libraries like SWR or React Query which provide built-in security features
- These libraries offer caching, streaming, and error handling with security in mind
- Implement proper error handling to avoid leaking sensitive information in error messages

### API Security
- Use input validation and output sanitization to secure APIs from SQL injection, XSS, and CSRF attacks
- Implement API gateways to centralize security controls
- Use rate limiting to prevent excessive requests and DDoS attacks
- Implement proper CORS policies, never use `*` for production

### Request Patterns
- Avoid sequential data fetching (waterfall requests) which can expose timing attacks
- Use parallel data fetching where possible
- Implement proper timeout handling for API requests
- Never trust client-side validation alone; always validate on the server

## Environment Variable Handling

### Critical Security Limitation
- **WARNING**: Client-side environment variables are NOT secure
- Environment variables in React are embedded into the build and can be accessed through React Developer Tools
- Never place API keys, secrets, or passwords in your React app's .env file
- Client-side data cannot be adequately secured in the browser

### Best Practices
- Only use environment variables for non-sensitive configuration (API endpoints, feature flags)
- In Create React App, only variables starting with `REACT_APP_` are exposed to the browser
- Store sensitive information as environment variables on the server, not in client-side code
- Use a backend API for any requests requiring secrets (logging in, database interactions)

### Configuration Management
- Add .env files with confidential data to .gitignore
- Never push .env files to public repositories
- Use secure environment variable managers for server-side secrets
- Rotate credentials immediately if accidentally committed

### Architecture Pattern
- Create a backend app with an API for your front-end app
- Send requests from front-end to back-end, which handles sensitive operations
- Never expose database credentials, API keys, or authentication secrets to the client

## Dependencies and Supply Chain Security

### NPM Supply Chain Attacks (2025)
- **CRITICAL**: Major npm supply chain attacks occurred in September 2025
- The "Shai-Hulud" worm compromised over 500 packages via self-replicating malware
- 18 widely used packages (chalk, debug, ansi-styles, strip-ansi) were compromised, affecting 2.6 billion weekly downloads
- @ctrl/tinycolor and 40+ other packages were trojanized

### Immediate Actions Required
- Pin npm package dependency versions to known safe releases produced prior to September 16, 2025
- Immediately rotate all developer credentials
- Mandate phishing-resistant MFA on all developer accounts, especially for GitHub and npm
- Use 2FA for all npm publishing operations

### Dependency Management
- Always use the latest stable versions of react and react-dom
- Run `npm outdated` regularly to check for updates
- Use software composition analysis (SCA) tools before adding third-party components
- Check dependencies with npm audit, Snyk, or Retire.js to scan for vulnerabilities
- Update dependencies when newer versions become available

### Package Verification
- Verify package integrity before installation
- Review package dependencies and maintainers
- Be cautious of packages with few maintainers or recent ownership changes
- Monitor security advisories for packages you depend on

### GitHub Authentication Changes (2025)
- GitHub is implementing new authentication requirements:
  - Local publishing with required 2FA
  - Granular tokens with a limited lifetime of seven days
  - Trusted publishing options
- Prepare for these changes in your CI/CD pipelines

## React Version Management

### Keep React Updated
- Always use the latest stable version of React to receive security patches
- Older versions of React may contain unpatched security vulnerabilities
- React 19 includes all Server Components features from the Canary channel
- Libraries shipping with Server Components should target React 19 as a peer dependency

### Security Monitoring
- Regularly check for security updates and patch versions
- Subscribe to React security advisories
- Monitor OWASP bulletins for React-specific vulnerabilities
- Review the React changelog for security-related updates

## Enterprise Security Practices

### Compliance and Encryption
- Encrypt data at rest and in transit to ensure compliance with GDPR, HIPAA, and SOC 2
- Implement TLS/SSL certificates and HSTS policies to protect sensitive data
- Use HTTPS for all communications, never HTTP
- Implement secure headers using Helmet.js in your Node.js backend

### Security Testing
- Use linters with security plugins (ESLint with security rules)
- Configure linters to catch security vulnerabilities early in the development cycle
- Implement Content Security Policy (CSP) headers
- Use Subresource Integrity (SRI) for third-party resources

### Monitoring and Incident Response
- Implement security monitoring and logging
- Set up alerts for suspicious activities
- Have an incident response plan for security breaches
- Regularly audit your application for security vulnerabilities

## Additional Security Measures

### Input Validation
- Validate all user inputs on both client and server
- Use schema validation libraries (Zod, Yup) for type-safe validation
- Never trust client-side validation alone
- Sanitize inputs before using them in database queries or rendering

### CSRF Protection
- Implement CSRF tokens for state-changing operations
- Use SameSite cookie attributes
- Verify origin and referer headers for sensitive operations
- Consider using anti-CSRF tokens for all POST, PUT, DELETE requests

### SQL Injection Prevention
- Use parameterized queries or prepared statements
- Never concatenate user input into SQL queries
- Use ORM libraries that provide built-in protection
- Implement proper database access controls

### Error Handling
- Never expose stack traces or detailed error messages to users
- Log errors securely on the server
- Implement generic error messages for users
- Use error boundaries to prevent React from unmounting on errors

## OWASP Resources

### Official OWASP Projects
- **OWASP Bullet-proof React**: Comprehensive resource for enhancing React and Node.js application security
- **OWASP Cross Site Scripting Prevention Cheat Sheet**: Covers React-specific XSS considerations
- **OWASP Cheat Sheet Series**: General security guidance applicable to React applications

### Framework-Specific Guidance
- While a dedicated OWASP React Security Cheatsheet is not yet published, refer to:
  - OWASP Bullet-proof React project: https://owasp.org/www-project-bullet-proof-react/
  - Cross Site Scripting Prevention: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
  - Node.js Security: https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html

## Security Checklist

### Development Phase
- [ ] Use latest stable React version (19.x)
- [ ] Configure ESLint with security plugins
- [ ] Implement DOMPurify for all user-generated HTML
- [ ] Validate and sanitize all user inputs
- [ ] Use Context API for authentication state
- [ ] Implement RBAC for authorization
- [ ] Never use dangerouslySetInnerHTML without sanitization
- [ ] Validate all URLs before rendering
- [ ] Avoid direct DOM manipulation

### Dependency Management
- [ ] Pin all dependency versions
- [ ] Run npm audit regularly
- [ ] Use Snyk or similar SCA tools
- [ ] Review dependencies before adding
- [ ] Keep all dependencies updated
- [ ] Verify package integrity
- [ ] Monitor security advisories

### Configuration
- [ ] Never commit .env files
- [ ] Use REACT_APP_ prefix for public env vars
- [ ] Store secrets server-side only
- [ ] Implement proper CORS policies
- [ ] Enable HTTPS/TLS everywhere
- [ ] Configure CSP headers
- [ ] Set secure cookie attributes

### Authentication & Authorization
- [ ] Implement OAuth 2.0 or similar
- [ ] Use JWT with proper expiration
- [ ] Enable MFA for all accounts
- [ ] Use HttpOnly cookies for tokens
- [ ] Implement proper session management
- [ ] Rotate credentials regularly
- [ ] Use phishing-resistant MFA

### Data Security
- [ ] Fetch sensitive data server-side
- [ ] Never expose API keys client-side
- [ ] Implement rate limiting
- [ ] Use parameterized queries
- [ ] Encrypt data at rest and in transit
- [ ] Implement proper error handling
- [ ] Validate all API responses

### Deployment & Monitoring
- [ ] Enable security headers (Helmet.js)
- [ ] Implement monitoring and logging
- [ ] Set up security alerts
- [ ] Regular security audits
- [ ] Incident response plan
- [ ] Security training for team
- [ ] Regular penetration testing

## Conclusion

React 19 provides strong default security protections, but developers must remain vigilant about specific attack vectors. The 2025 npm supply chain attacks highlight the critical importance of dependency management and secure development practices. Always assume that client-side code is visible to attackers and design your security architecture accordingly. Use server-side rendering, proper authentication patterns, and defense-in-depth strategies to build secure React applications.

## Sources

This document is based on research from reputable sources including:
- OWASP Foundation (Bullet-proof React, Cheat Sheet Series)
- React Official Documentation (react.dev)
- CISA Security Advisories (2025 npm supply chain attack)
- GitHub Security Blog
- Security research from StackHawk, Snyk, and other security experts
- Industry best practices from enterprise security teams

Last updated: October 2025 based on mid-2025 security landscape
