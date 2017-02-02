# Configuring Office add-ins generated by the Yeoman Office Generator for using ADAL JS

If you want to use ADAL JS in your add-in generated by the Yeoman Office Generator you have to change how Angular bootstraps the add-in in the __app.module.js__ file:

```js
if (location.href.indexOf('access_token=') < 0) {
	Office.initialize = function () {
		console.log(">>> Office.initialize()");
		angular.bootstrap(document.getElementById('container'), ['officeAddin']);
	};
}
else {
  angular.bootstrap(document.getElementById('container'), ['officeAddin']);
}
```

By default bootstrapping the Angular application is delayed until the Office add-in has been initiated. This has to do with the fact that if initializing the Office add-in takes too long, loading it is cancelled by the Office client application. Because bootstrapping an Angular application might take longer than what's allowed for Office add-ins, bootstrapping Angular is delayed and executed after the Office add-in has been initialized.

ADAL JS uses a hidden iframe to obtain access tokens for the different endpoints registered with ADAL. By default the whole Angular application is loaded in that iframe. Unfortunately as the __Office.initialize__ handler is not executed within the iframe, the Angular application is not being bootstrapped and the ADAL library won't process the retrieved access token. As a result, although you won't see any errors, requests to endpoints registered with ADAL won't complete.

The code sample showed above checks whether the current request is an ADAL callback: if not, it will delay bootstrapping the Angular application until the Office add-in has been initialized. If it is however, it will immediatelly bootstrap the Angular application allowing the ADAL library to process its callback and complete the outstanding requests.

## Extending Office add-ins generated by the Yeoman Office Generator for using ADAL JS

If you are interested in leveraging Office 365 APIs or the Microsoft Graph in your Office add-ins generated by the Yeoman Office Generator there are a few steps that you need to complete. The following is a sample overview of these steps assuming that you chose for an Angular Office add-in.

### Reference ADAL JS library

In the __index.html__ file, right after the reference to the __angular.sanitize__ module, add the references to the ADAL JS library:

```html
<script src="//secure.aadcdn.microsoftonline-p.com/lib/1.0.0/js/adal.min.js"></script>
<script src="//secure.aadcdn.microsoftonline-p.com/lib/1.0.0/js/adal-angular.min.js"></script>
```

### Include ADAL JS in the officeAddin module

In the __app.module.js__ file extend the officeAddin module definition to include the __AdalAngular__ module:

```js
// create
var officeAddin = angular.module('officeAddin', [
  'ngRoute',
  'ngSanitize',
  'AdalAngular'
]);
```

### Change the Angular application bootstrap routine to support ADAL callbacks

In the __app.module.js__ file change how the Angular application is bootstrapped so that it supports ADAL callbacks:

```js
if (location.href.indexOf('access_token=') < 0) {
	Office.initialize = function () {
		console.log(">>> Office.initialize()");
		angular.bootstrap(document.getElementById('container'), ['officeAddin']);
	};
}
else {
	angular.bootstrap(document.getElementById('container'), ['officeAddin']);
}
``` 

### Create a config file for your application

In the __app__ folder create the __app.config.js__ file and include information about your application:

```js
(function () {
	var tenantName = 'contoso';
	
	var officeAddin = angular.module('officeAddin');
	officeAddin.constant('appId', '00000000-0000-0000-0000-000000000000');								   
	officeAddin.constant('sharePointUrl', 'https://' + tenantName + '.sharepoint.com');
})();
```

This config file contains the name of your Azure Directory and the ID of your application as registered with Azure. The information from this file is used for configuring the ADAL JS library for your application as well as executing requests to the Office 365 API. Although the above example includes only the URL for interacting with the SharePoint API, you can include additional URLs if you need to communicate with other APIs such as e-mail.

### Configure ADAL JS for your application

In the __app__ folder create the __app.adalconfig.js__ file and include ADAL JS configuration for your application:

```js
(function () {
  'use strict';

  var officeAddin = angular.module('officeAddin');

  officeAddin.config(['$httpProvider', 'adalAuthenticationServiceProvider', 'appId', 'sharePointUrl', adalConfigurator]);

  function adalConfigurator($httpProvider, adalProvider, appId, sharePointUrl) {
    var adalConfig = {
      tenant: 'common',
      clientId: appId,
      extraQueryParameter: 'nux=1',
      endpoints: {}
      // cacheLocation: 'localStorage', // enable this for IE, as sessionStorage does not work for localhost. 
    };
    adalConfig.endpoints[sharePointUrl + '/_api/'] = sharePointUrl;
    adalProvider.init(adalConfig, $httpProvider);
  }
})();
```

The above code snippet initializes the ADAL JS provider with the configuration including the application ID and an array of endpoints. Whenever your application executes a request to any of these endpoints, ADAL JS will automatically retrieve the corresponding access token and pass it along with the request.

### Require authentication for the home route

In the __app.routes.js__ file extend the definition of the home route to require Azure AD authentication:

```js
function routeConfigurator($routeProvider) {
  $routeProvider
    .when('/', {
      templateUrl: 'app/home/home.html',
      controller: 'homeController',
      controllerAs: 'vm',
      requireADLogin: true
    });

  $routeProvider.otherwise({ redirectTo: '/' });
}
```

For the ADAL JS library to automatically retrieve access tokens for the specified endpoints, it requires the current user to be authenticated with Azure AD. One way to do that is to require a specific route of your Angular application to authenticate against the Azure AD. This is done by adding the __requireADLogin__ parameter to the route definition.

### Load newly added files with the rest of the application

In the __index.html__ file add references to the newly added __app.config.js__ and __app.adalconfig.js__ files so that they are loaded with your application:

```html
<script src="app/app.module.js"></script>
<script src="app/app.config.js"></script>
<script src="app/app.adalconfig.js"></script>
<script src="app/app.routes.js"></script>
<script src="app/services/data.service.js"></script>
<script src="app/home/home.controller.js"></script>
```

### Register app domains required to complete AAD authentication from Office clients

In the __manifest.xml__ file of your Office add-in, just before the __Hosts__ element, add the following snippet:

```xml
<AppDomains>
  <AppDomain>https://login.windows.net</AppDomain>
  <AppDomain>https://login.microsoftonline.net</AppDomain>
  <AppDomain>https://login.microsoftonline.com</AppDomain>
</AppDomains>
```
The AppDomains section specifies additional domains that your add-in will use to load pages. 

This concludes the configuration steps necessary to add ADAL JS support to your Office add-in created using the Yeoman Office Generator. Assuming you have enabled the implicit authentication flow for your application in Azure AD, you should be able to interact with Office 365 APIs or Microsoft Graph within your application. 