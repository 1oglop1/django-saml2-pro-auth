[![Coverage Status](https://coveralls.io/repos/github/MindPointGroup/django-saml2-pro-auth/badge.svg?branch=master)](https://coveralls.io/github/MindPointGroup/django-saml2-pro-auth?branch=master)

[![CircleCI](https://circleci.com/gh/MindPointGroup/django-saml2-pro-auth.svg?style=svg)](https://circleci.com/gh/MindPointGroup/django-saml2-pro-auth)


# django-saml2-pro-auth
SAML2 authentication backend for Django


## Installation

### Prerequisites

On your OS you must install xmlsec1 and openssl (dev packages where available). The package name will vary by OS.

## Configuration

### Installation

**Python 2**

```
  pip install django-saml2-pro-auth
```

**Python 3**

```
  pip3 install django-saml2-pro-auth
```


### settings.py

Here is an example full configuration. Scroll down to read about each option

```python

AUTHENTICATION_BACKENDS = [
      'django_saml2_pro_auth.auth.Backend'
]

SAML_ROUTE = 'sso/saml/'

SAML_REDIRECT = '/'

SAML_USERS_MAP = [{
    "MyProvider" : {
      "email": dict(key="Email", index=0),
      "name": dict(key="Username", index=0)
    }

}]


SAML_PROVIDERS = [{
    "MyProvider": {
        "strict": True,
        "debug": False,
        "sp": {
            "entityId": "https://test.davila.io/sso/saml/metadata",
            "assertionConsumerService": {
                "url": "https://test.davila.io/sso/saml/?acs",
                "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
            },
            "singleLogoutService": {
                "url": "https://test.davila.io/sso/saml/?sls",
                "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
            },
            "NameIDFormat": "urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified",
            "x509cert": open(os.path.join(BASE_DIR,'certs/sp.crt'), 'r').read(),
            "privateKey": open(os.path.join(BASE_DIR,'certs/sp.key'), 'r').read(),
        },
        "idp": {
            "entityId": "https://kdkdfjdfsklj.my.MyProvider.com/0f3172cf-5aa6-40f4-8023-baf9d0996cec",
            "singleSignOnService": {
                "url": "https://kdkdfjdfsklj.my.MyProvider.com/applogin/appKey/0f3172cf-5aa6-40f4-8023-baf9d0996cec/customerId/kdkdfjdfsklj",
                "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
            },
            "singleLogoutService": {
                "url": "https://kdkdfjdfsklj.my.MyProvider.com/applogout",
                "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
            },
            "x509cert": open(os.path.join(BASE_DIR,'certs/MyProvider.crt'), 'r').read(),
        },
        "organization": {
            "en-US": {
                "name": "example inc",
                "displayname": "Example Incorporated",
                "url": "example.com"
            }
        },
        "contact_person": {
            "technical": {
                "given_name": "Jane Doe",
                "email_address": "jdoe@examp.com"
            },
            "support": {
                "given_name": "Jane Doe",
                "email_address": "jdoe@examp.com"
            }
        },
        "security": {
            "name_id_encrypted": False,
            "authn_requests_signed": True,
            "logout_requests_signed": False,
            "logout_response_signed": False,
            "sign_metadata": False,
            "want_messages_signed": False,
            "want_assertions_signed": True,
            "want_name_id": True,
            "want_name_id_encrypted": False,
            "want_assertions_encrypted": True,
            "signature_algorithm": "http://www.w3.org/2000/09/xmldsig#rsa-sha1",
            "digest_algorithm": "http://www.w3.org/2000/09/xmldsig#rsa-sha1",
        }

    }
}]
```

**AUTHENTICATION_BACKENDS:** This is required exactly as in the example. It tells Django to use this as a valid auth mechanism.

**SAML_ROUTE (optional, default=/sso/saml/):** This tells Django where to do all SAML related activities. The default route is /sso/saml/. You still need to include the source urls in your own `urls.py`. For example:

```python
from django.conf.urls import include, url
from django.contrib import admin
from django.conf import settings
from django.conf.urls.static import static
import profiles.urls
import accounts.urls
import django_saml2_pro_auth.urls as saml_urls

from . import views

urlpatterns = [
    url(r'^$', views.HomePage.as_view(), name='home'),
    url(r'^about/$', views.AboutPage.as_view(), name='about'),
    url(r'^users/', include(profiles.urls, namespace='profiles')),
    url(r'^admin/', include(admin.site.urls)),
    url(r'^', include(accounts.urls, namespace='accounts')),
    url(r'^', include(saml_urls, namespace='saml')),
]

```
So first import the urls via `import django_saml2_pro_auth.urls as saml_urls` (it's up to you if you want name it or not). Then add it to your patterns via `url(r'^', include(saml_urls, namespace='saml'))`. This example will give you the default routes that this auth backend provides.

**SAML_REDIRECT (optional, default=None):** This tells the auth backend where to redirect users after they've logged in via the IdP. **NOTE**: This is not needed for _most_ users. Order of precedence is: SAML_REDIRECT value (if defined), RELAY_STATE provided in the SAML response, and the fallback is simply to go to the root path of your application.

**SAML_USERS_MAP (required):** This is what makes it possible to map the attributes as they come from your IdP into attributes that are part of your User model in Django. There a few ways you can define this. The dict keys (the left-side) are the attributes as defined in YOUR User model, the dict values (the right-side) are the attributes as supplied by your IdP.

```python
## Simplest Approach, when the SAML attributes supplied by the IdP are just plain strings
## This means my User model has an 'email' and 'name' attribute while my IdP passes 'Email' and 'Username' attrs
SAML_USERS_MAP = [{
    "myIdp" : {
      "email": "Email",
      "name": "Username
    }
}]
```

Sometimes, IdPs might provide values as Arrays (even when it really should just be a string). This package supports that too. For example, suppose your IdP supplied user attributes with the following data structure:
`{"Email": ["foo@example.com"], "Username": "foo"}`
You simply would make the key slightly more complex where `key` is the key and `index` represents the index where the desired value is located. See below:

```python
SAML_USERS_MAP = [{
    "myIdp" : {
      "email": {"key": "Email", "index": 0},
      "name": "Username
    }
```

And of course, you can use the dict structure even when there IdP supplied attribute isn't an array. For example:

```python
SAML_USERS_MAP = [{
    "myIdp" : {
      "email": {"key": "Email"},
      "name": {"key": "Username
    }
```



**SAML_PROVIDERS:** This is exactly the same spec as OneLogin's [python-saml and python3-saml packages](https://github.com/onelogin/python3-saml#settings). The big difference is here you supply a list of dicts where the top most key(s) must map 1:1 to the top most keys in `SAML_USERS_MAP`. Also, this package allows you to ref the cert/key files via `open()` calls. This is to allow those of you with multiple external customers to login to your platform with any N number of IdPs.


## Routes

| **Route**                                 | **Uses**                                                                                                                                                                                                              |
|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/sso/saml2/?acs&provider=MyProvider`     | The Assertion Consumer Service Endpoint. This is where your IdP will be POSTing assertions. The 'provider' query string must have a value that matches a top level key of your SAML_PROVIDERS settings.               |
| `/sso/saml2/metadata&provider=MyProvider` | This is where the SP (ie your Django App) has metadata. Some IdPs request this to generate configuration. The 'provider' query string must have a value that matches a top level key of your SAML_PROVIDERS settings. |
| `/sso/saml2/?provider=MyProvider`         | Use this endpoint when you want to trigger an SP-initiated login. For example, this could be the `href`of a "Login with ClientX Okta" button.                                                                         |

## Gotchas

The following are things that you may run into issue with. Here are some tips.

* Ensure the value of the SP `entityId` config matches up with what you supply in your IdPs configuration.
* Your IdP may default to particular Signature type, usually `Assertion` or `Response` are the options. Depending on how you define your SAML provider config, it will dictate what this value should be.

# Wishlist and TODOs

The following are things that arent present yet but would be cool to have 

* Implement logic for Single Logout Service
* ADFS IdP support
* Integration test with full on mock saml interactions to test the actual backend auth
* Tests add coverage to views and the authenticate() get_user() methods in the auth backend

