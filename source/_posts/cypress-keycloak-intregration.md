---
title: Cypress.io Keycloak Integration
categories:
- QA
tags:
- QA
- Cypress
- Keycloak
---
[Cypress.io](https://www.cypress.io/) is getting some traction these days and since my favorite FE superstar 
[@bahmutov](https://twitter.com/bahmutov) promotes it a lot I couldn't help myself but get my hands dirty. Writing
FE tests was always a pain and Cypress is promising a painless experience.

Going through the [docs](https://docs.cypress.io/guides/overview/why-cypress.html#What-You’ll-Learn) definitely was 
painless. Documentation is spiced with lots of best practices and I would say it's a good read 
about the FE/E2E testing in general. But still... while they stress you should not log in by 
[visiting the login page](https://docs.cypress.io/guides/references/best-practices.html#Visiting-External-Sites),
 Cypress doesn't work with OAuth/OpenID out of the box.
<!-- more -->

My web-app is secured with [Keycloak](http://www.keycloak.org/) (KC) - an open-source identity and access management 
server. I had the privilege to work on KC while in Red Hat and I still got a strong emotional attachment to the project.
I want to encourage people to use KC thus finding and sharing a proper way to use Cypress with KC secured web-app was 
inevitable. 

After the docs, I've started looking for help on the official Cypress.io 
[chat page](https://gitter.im/cypress-io/cypress). I must admit the community there is friendly, ready to help 
(thanks, @MarcLoupias!) and I definitely got some good advice (although I was asking wrong questions - I've only learned
the [difference between OAuth and OpenID](https://stackoverflow.com/a/1087071/3252949) while writing this blog-post). 
Eventually, gorgeous [mposolda](https://github.com/mposolda) promptly pointed me to the right 
[test code in the KC repository](https://github.com/keycloak/keycloak/blob/master/testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/admin/concurrency/ConcurrentLoginTest.java#L275-L277) 
and after a brief inspection of the KC javascript client code, I was ready for a quick&dirty solution. 

## Solution

I came up with a relatively simple KC integration. It's not perfect, but hey, it works, and it can help us 
get started with Cypress! It works OOTB if your web-app is secured the KC "tutorial" way, using the 
[javascript client](http://www.keycloak.org/docs/latest/securing_apps/topics/oidc/javascript-adapter.html). You just
need to replace few constants in the code and you're ready to go. I've basically mimicked the KC test, to create the 
login [custom command](https://docs.cypress.io/api/cypress-api/custom-commands.html):

```javascript
Cypress.Commands.add('kcLogin', (username, password) => {
  const kcRoot = 'http://my.keycloak.com';
  const kcRealm = 'MYrealm';
  const kcClient = 'my-client';
  const kcRedirectUri = 'http://localhost:3000/';

  const loginPageRequest = {
    url: `${kcRoot}/auth/realms/${kcRealm}/protocol/openid-connect/auth`,
    qs: {
      client_id: kcClient,
      redirect_uri: kcRedirectUri,
      state: createUUID(),
      nonce: createUUID(),
      response_mode: 'fragment',
      response_type: 'code',
      scope: 'openid'
    }
  };

  // Open the KC login page, fill in the form with username and password and submit.
  return cy.request(loginPageRequest)
    .then(submitLoginForm);

  ////////////

  function submitLoginForm(response) {

    const _el = document.createElement('html');
    _el.innerHTML = response.body;

    // This should be more strict depending on your login page template.
    const loginForm = _el.getElementsByTagName('form');
    const isAlreadyLoggedIn = !loginForm.length;

    if (isAlreadyLoggedIn) {
      return;
    }

    return cy.request({
      form: true,
      method: 'POST',
      url: loginForm[0].action,
      followRedirect: false,
      body: {
        username: username,
        password: password        
      }
    });
  }

  // Copy-pasted code from KC javascript client. It probably doesn't need to be 
  // this complicated but I refused to spend time on figuring that out.
  function createUUID() {
    var s = [];
    var hexDigits = '0123456789abcdef';

    for (var i = 0; i < 36; i++) {
      s[i] = hexDigits.substr(Math.floor(Math.random() * 0x10), 1);
    }

    s[14] = '4';
    s[19] = hexDigits.substr((s[19] & 0x3) | 0x8, 1);
    s[8] = s[13] = s[18] = s[23] = '-';

    var uuid = s.join('');

    return uuid;
  }
});
```

The logout command is trivial:

```javascript
Cypress.Commands.add('kcLogout', () => {
  const kcRoot = 'http://my.keycloak.com';
  const kcRealm = 'MYrealm';
  const kcRedirectUri = 'http://localhost:3000/';

  return cy.request({
    url: `${kcRoot}/auth/realms/${kcRealm}/protocol/openid-connect/logout`,
    qs: {
      redirect_uri: kcRedirectUri
    }
  });
});
```

The test itself now couldn't be easier to scaffold:

```javascript
describe('Dummy test', () => {

  beforeEach(() =>  {
    cy.kcLogin('testuser', '********');
  });

  afterEach(() =>  {
    cy.kcLogout();
  });

  it('should render logged user name somewhere on the page', () =>  {
    cy.visit('/');
    cy.get('#login-test')
      .should('contain', 'testuser');
  });
});
```

Moving KC variables to [environment variables](https://docs.cypress.io/guides/guides/environment-variables.html#)
 will be the next natural step. 
Anyway, if you got any issues with this approach, or even better - suggestions, please comment.